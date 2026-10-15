# Seven API Quality Gates for Orientation Conversion and Print Ready Images

A print-asset API should preserve the uploaded original, normalize orientation before removing the background, and derive each delivery format from one canonical working image. **TL;DR:** to prepare print ready images, spend bandwidth on the source and final master; use smaller previews for customer-support review. This API approach makes orientation normalisation explicit. The deciding constraint is quality versus bandwidth, not the number of conversion options.

This matters in customer support because an agent resolving a product-photo ticket needs two different things: a fast visual answer and an asset that survives production. A small transparent preview can confirm that the cutout is sensible. It cannot prove that the full-resolution edge, color description, or printable dimensions are acceptable. Treating those files as interchangeable is the simple approach, and it fails at the handoff to print.

## How should an API prepare images for print ready conversion?

Image orientation can be represented as metadata rather than baked into the stored pixel order. The EXIF specification defines orientation values that describe the relationship between the stored image and the intended viewing direction. A decoder may honor that metadata during display, while a later transformation or metadata-stripping step may not. The result can be a preview that looked upright and a derived file that does not.

Normalize once at ingress: decode the pixels, apply the orientation transform, set the canonical orientation to top-left, and only then pass the raster to segmentation or matting. Remove stale orientation metadata from derived files. This ordering gives the background-removal stage the same coordinate system that crop checks, edge masks, and print export will use.

It also prevents a subtle evaluation error. Consider the complete failure chain: a phone stores portrait pixels with an orientation tag; the support dashboard honors that tag; an agent approves the upright preview; a background-removal worker decodes the stored pixel order but drops metadata; its mask now follows the sideways raster; and the print exporter rotates only the color image. The mask and product no longer share coordinates, even though every individual stage returned a valid file. If a reference mask is upright but the candidate image still relies on an orientation tag, pixel comparisons measure rotation disagreement instead of cutout quality. A single canonicalization step before matting removes that ambiguity and gives the team one place to test all eight EXIF orientation values defined by the specification.

Bad test, bad decision.

That is the gate.

## One canonical raster, two delivery classes

The failed design converts the upload straight into every requested output. Each branch can decode metadata differently, resize at a different point, or apply transparency rules independently. Drift becomes hard to explain.

A more inspectable pipeline has a narrow waist. Keep the original bytes for reproducibility. Decode and orient them into a canonical working raster, run background removal once at the required resolution, and then create a review derivative plus a print master. The review file is optimized for quick transfer and browser inspection. The master preserves the resolution and color information required by the downstream print workflow.

Format choice follows capability. MDN's image-format guide documents that JPEG does not support an alpha channel, while PNG supports alpha transparency. That makes a transparent JPEG export an invalid requirement, not an encoder preference. If production requires a white background, composite onto an explicit white layer before JPEG encoding. If it requires transparency, choose a format and downstream workflow that preserve alpha. TIFF is widely used for print-oriented raster interchange, but its many possible encodings mean the receiving printer's profile is still the contract.

Do not infer print readiness from a filename or a nominal DPI field. Pixel dimensions determine how many samples exist; the requested physical size determines the needed sampling density. For example, a 3,000-pixel edge placed at 10 inches yields 300 pixels per inch. That arithmetic is deterministic, but acceptable density is a production requirement, so obtain it from the printer rather than hard-coding a universal threshold.

## A focused acceptance harness

I would start in a notebook with a handful of fixtures, then move the same assertions into CI. The harness below deliberately evaluates the service contract rather than a particular imaging library. It catches rotated outputs, accidental dimension loss, and a missing alpha channel without pretending those checks measure edge quality.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class AssetFacts:
    width_px: int
    height_px: int
    orientation: int | None
    has_alpha: bool
    format_name: str


class AssetService(Protocol):
    def prepare(self, source: bytes, *, output_format: str) -> bytes: ...
    def inspect(self, asset: bytes) -> AssetFacts: ...


def assert_transparent_master(
    service: AssetService, source: bytes, expected_size: tuple[int, int]
) -> None:
    result = service.prepare(source, output_format="PNG")
    facts = service.inspect(result)

    assert (facts.width_px, facts.height_px) == expected_size
    assert facts.orientation in (None, 1)
    assert facts.has_alpha
    assert facts.format_name == "PNG"
```

The useful fixture set is small but adversarial: portrait and landscape uploads, every orientation value your decoders claim to support, an opaque product on a near-white background, a pale object with interior holes, hair-like or reflective edges, and an image that already contains alpha. Use synthetic orientation fixtures for deterministic geometry checks. Use rights-cleared product photos for visual scoring.

Then separate binary gates from scored judgments. Dimensions, orientation, declared format, alpha presence, and successful decode can fail a build. Edge halos, foreground loss, and shadow retention need a labeled reference set plus a metric or reviewer rubric. Keep those scores by fixture category; one average can conceal a model that handles boxes well and reflective bottles poorly.

## Where bandwidth is worth spending

Upload the original once. Repeatedly shipping the full print master through an agent dashboard wastes time without improving the decision an agent is making. A bounded preview is enough to approve composition, while zoomed edge crops can expose matting defects at far less transfer cost than another full image.

The full master should move only across the production path and should be protected by an integrity digest. Cache derivatives by the digest of the original plus a versioned transformation policy. That policy must include orientation behavior, background treatment, output format, dimensions, and color-profile handling; changing any of them creates a different asset.

This is also where prompt-cost awareness helps even though the workload is visual. If an AI reviewer receives images, do not send the print master by habit. Give it the same controlled preview and edge crops used by the eval harness, and reserve human review for low-confidence or policy-sensitive cases. The quality decision becomes repeatable, and the model input stays bounded.

## What to measure before copying this design

Measure decoded failures, orientation mismatches, output-dimension mismatches, and format or alpha violations as rates. Measure background-removal quality separately on labeled foreground masks and on the edge cases that matter for the catalog. Track source bytes, preview bytes, master bytes, transform duration, and review latency. These numbers expose the actual quality-bandwidth curve.

Also record the transformation-policy version and an input digest with every result. Do not log customer images or embedded metadata casually; uploads can carry identifying information. Define retention and access rules with the support and production teams before preserving originals.

**The decision rule is straightforward:** adopt the two-class delivery pattern only if preview-based review maintains the team's acceptance quality while materially reducing transferred bytes or review latency. Otherwise, simplify. Orientation-first canonicalization remains useful because it removes ambiguity from every later comparison, regardless of delivery design.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://www.cipa.jp/std/documents/e/DC-008-Translation-2023-E.pdf
- https://www.color.org/specification/ICC.1-2022-05.pdf
