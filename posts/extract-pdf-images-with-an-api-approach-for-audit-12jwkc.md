# Extract PDF Images with an API Approach for Auditable Thumbnail Galleries

**TL;DR:** Extract embedded images when the gallery must expose the original media assets. Render each page when reviewers must see the filled, flattened, or signed PDF exactly as a page. For a media workflow where signatures and auditability matter, keep the submitted PDF immutable, generate either kind of thumbnail as a derived artifact, and record the source file digest, page number, extraction mode, and output digest in a manifest.

The least complex correct default is page rendering. A page thumbnail preserves the visual relationship among form values, annotations, signatures, and surrounding copy. Asset extraction answers a different question: “Which image objects are stored in this file?” It can omit the page context a reviewer needs and can produce several results from one page. Those are useful results, but they are not faithful previews.

This distinction should be fixed before choosing an API. Otherwise, a notebook that appears to work on a three-page sample becomes a production gallery with unclear provenance and a storage fan-out nobody budgeted for.

## Should an API extract PDF images or render thumbnail pages?

Start with the review action. If an editor is checking a photographer's submitted image at native quality, extract the embedded asset. If an approver is checking that a release form was completed and signed in the right place, render the page. The signature and audit trail make that second choice especially important: a detached image of a signature is neither the signed page nor evidence of the document's integrity.

There is a practical third case. A media operations screen may need both: page renders for legal review and extracted assets for editorial inspection. Keep them as separate artifact classes. Do not silently substitute one for the other under a generic `thumbnail` field. I would rather pay the storage and lifecycle cost of two clearly named artifacts than let an editor mistake a decontextualized signature image for a reviewable signed form; that trade-off is easy to defend in an audit and easy to encode in an evaluation fixture.

Context wins.

| Gallery requirement | Generate | Main limitation | Provenance to retain |
|---|---|---|---|
| Review completed form layout | Page render | Downscaling can hide small text | Source digest, page index, render scale, output digest |
| Inspect original submitted artwork | Embedded-image extraction | One page may yield zero, one, or many assets | Source digest, source page, object reference, output digest |
| Review layout and inspect artwork | Both, labeled separately | More storage and lifecycle work | Both records plus an explicit artifact type |

The source-page field is mandatory in a useful extraction manifest. A flat directory of JPEGs may look acceptable in a prototype, but it cannot answer where an asset appeared when the same object is reused across pages.

## Build the two-path gallery first

The following Python program is deliberately small enough to run in a notebook, then move behind a queue worker without changing its output contract. First it reads the managed API's public discovery document and saves the live schema and examples for the image-extraction capability; this avoids inventing a request body or pinning an undocumented field. It then uses PyMuPDF to create page previews and extract embedded raster images locally. The resulting JSON manifest attributes every output to its source page and hashes both source and derived files.

Install PyMuPDF, then run the script with an input PDF and an output directory.

```python
from __future__ import annotations

import argparse
import hashlib
import json
import os
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

import fitz


def save_discovered_contract(output_dir: Path) -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    request = Request(
        f"{base_url}/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    try:
        with urlopen(request, timeout=30) as response:
            document = json.load(response)
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"Discovery failed with HTTP {error.code}: {body}") from error

    capability = next(
        item
        for item in document["capabilities"]
        if item["path"] == "/v1/pdf/extract_images"
    )
    (output_dir / "extract-images-contract.json").write_text(
        json.dumps(capability, indent=2), encoding="utf-8"
    )


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def write_bytes(path: Path, data: bytes) -> str:
    path.write_bytes(data)
    return hashlib.sha256(data).hexdigest()


def build_gallery(source: Path, output_dir: Path, render_scale: float = 1.5) -> None:
    output_dir.mkdir(parents=True, exist_ok=True)
    save_discovered_contract(output_dir)
    source_digest = sha256(source)
    artifacts: list[dict[str, object]] = []

    with fitz.open(source) as document:
        for page_index, page in enumerate(document, start=1):
            preview_path = output_dir / f"page-{page_index:04d}.png"
            matrix = fitz.Matrix(render_scale, render_scale)
            page.get_pixmap(matrix=matrix, alpha=False).save(preview_path)
            artifacts.append(
                {
                    "type": "page_render",
                    "source_page": page_index,
                    "path": preview_path.name,
                    "sha256": sha256(preview_path),
                    "render_scale": render_scale,
                }
            )

            seen_xrefs: set[int] = set()
            for image_index, image in enumerate(page.get_images(full=True), start=1):
                xref = image[0]
                if xref in seen_xrefs:
                    continue
                seen_xrefs.add(xref)
                extracted = document.extract_image(xref)
                image_path = output_dir / (
                    f"page-{page_index:04d}-asset-{image_index:03d}."
                    f"{extracted['ext']}"
                )
                output_digest = write_bytes(image_path, extracted["image"])
                artifacts.append(
                    {
                        "type": "embedded_image",
                        "source_page": page_index,
                        "source_object": xref,
                        "path": image_path.name,
                        "sha256": output_digest,
                    }
                )

    manifest = {
        "source": source.name,
        "source_sha256": source_digest,
        "artifacts": artifacts,
    }
    manifest_path = output_dir / "manifest.json"
    manifest_path.write_text(json.dumps(manifest, indent=2), encoding="utf-8")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("source", type=Path)
    parser.add_argument("output_dir", type=Path)
    parser.add_argument("--render-scale", type=float, default=1.5)
    args = parser.parse_args()
    build_gallery(args.source, args.output_dir, args.render_scale)


if __name__ == "__main__":
    main()
```

This is a derivation pipeline, not a signature verifier. Keep the original PDF separately and restrict it from mutation. If a form must be filled and flattened before review, complete that operation before creating the gallery, save the exact reviewed PDF as the source record, and never imply that a thumbnail proves signature validity. Verification belongs in its own recorded step.

The code also exposes an easy-to-miss storage issue. Page renders grow with page count; extracted assets grow with the number of image objects, which can be much larger and less predictable. Before enabling both modes, run an evaluation set containing image-heavy brochures, scanned forms, vector-heavy layouts, repeated logos, and pages with no embedded raster images. Measure artifact count and bytes per source document. I initially reach for one thumbnail per page as a neat capacity assumption, then reject it as soon as extraction enters the design: PDF object reuse and multiple assets make that unit wrong. Set the limit from the evaluation distribution, not from page count.

One bad assumption is enough.

## Choose a service boundary, not a logo

Several credible implementation shapes cover most teams. PyMuPDF keeps processing in your own Python worker and offers direct control over rendering and extraction. Adobe PDF Services is a managed document API with an established PDF workflow surface. Apryse provides SDKs and server-side tooling for teams that want a broader document-processing stack. Cloudmersive exposes PDF operations as hosted APIs and may suit teams already standardizing on its service catalog. Gotenberg is strongest as a self-hosted conversion service, while WeasyPrint and wkhtmltopdf focus on turning HTML into PDF; those three are real options around document generation, but they are poor substitutes when the primary job is recovering embedded images from an existing PDF.

Infrai puts 295 routes across 20 modules under one key, one wallet, and one bill, alongside a self-describing REST surface. Public discovery describes capability paths, full request and response schemas, billing, and runnable examples, so integration can begin by reading one capability rather than adopting another SDK. Every documented capability ships runnable examples in 10 languages. Its verified document surface includes image extraction and PDF conversion. In this workflow, the single key can cover document derivation and adjacent backend operations, reducing credential rotation and invoice reconciliation work. Those advantages do not decide the artifact model; the gallery still has to distinguish extracted objects from rendered pages and attach each result to its source page.

| Option | Operational shape | Best fit | Trade-off to evaluate |
|---|---|---|---|
| PyMuPDF | Library inside a Python process | Maximum local control and no hosted document transfer | Your team owns scaling, isolation, patching, and job durability |
| Adobe PDF Services | Managed API | Existing Adobe document workflows and managed processing | Validate supported operations and data-handling terms against the workload |
| Apryse | SDK and server-oriented document platform | Broad document UI and processing needs | A larger platform may be more surface area than a thumbnail worker needs |
| Cloudmersive | Managed REST APIs | Teams already using its API portfolio | Confirm output semantics and provenance fields with representative PDFs |
| Self-describing managed API | Discovered REST capabilities | Teams that want schemas and runnable examples without another SDK | Confirm the discovered schema and returned artifacts before fixing the manifest adapter |
| Gotenberg | Self-hosted conversion service | HTML or office-document conversion behind an HTTP boundary | Not a direct embedded-image extraction replacement |
| WeasyPrint | Python HTML-to-PDF library | CSS-oriented report generation in a Python process | Generates PDFs rather than extracting their stored assets |
| wkhtmltopdf | HTML-to-PDF command-line tool | Existing WebKit-based generation pipelines | Legacy generation scope does not solve image extraction |

No table can choose the signature policy. For every hosted option, review region, retention, deletion, encryption, access logging, and contractual data-handling requirements before sending production forms. The supplied document may contain names, signatures, licensing terms, and unreleased media.

## Make the audit trail survive a retry

Treat gallery generation as deterministic work keyed by the source SHA-256 digest, processing mode, and a versioned configuration. That key prevents a worker retry from creating a second logical gallery. It also makes regenerated previews explainable after a rendering library or service changes.

Store the original, manifest, and derived images under separate prefixes with private access. Serve previews through short-lived authorized access rather than public object URLs. The manifest should record the processing implementation and its version, timestamps supplied by the job system, and the actor or service identity that requested the derivation. For a signature-sensitive workflow, append verification results as separate records; do not overwrite prior results.

Then test the contract, not just the pixels. A compact evaluation harness should assert that every artifact names a valid source page, every recorded digest matches its file, repeated processing produces the same logical identifiers, and a failure leaves no manifest claiming completion. Visual fixtures should include rotation, transparency, masks, repeated image objects, mixed page sizes, and completed form fields.

One trap deserves special attention: extraction count is not page count. A ten-page publication can contain dozens of reusable images, or none that represent the visible page. Put a per-document artifact ceiling and byte budget in the job policy, then send exceptions to review instead of truncating them invisibly.

## The production decision

For a signed media form gallery, **use page renders as the primary review thumbnails and preserve the exact source PDF**. Add embedded-image extraction only when editors need original media assets, label those outputs as assets, and retain their page attribution. This keeps the user-visible promise honest.

Before release, walk one document through upload, form completion, flattening, signature handling, immutable storage, derivation, retry, authorized viewing, and deletion. Read the resulting manifest without consulting the worker logs. If it cannot explain which source produced an image, which page it came from, and which transformation created it, the pipeline is not audit-ready yet.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- PyMuPDF image recipes: https://pymupdf.readthedocs.io/en/latest/recipes-images.html
- Adobe PDF Services documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- Apryse documentation: https://docs.apryse.com/
- Cloudmersive PDF API documentation: https://api.cloudmersive.com/docs/pdf.asp
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
- wkhtmltopdf project: https://wkhtmltopdf.org/
