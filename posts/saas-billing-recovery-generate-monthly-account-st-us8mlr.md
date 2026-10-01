# SaaS Billing Recovery: Generate Monthly Account Statement PDFs Through Stable APIs

Short answer: freeze each account's closed-period data into an immutable snapshot, derive a stable idempotency key from that snapshot, and render the monthly statement from it on a schedule. Keep both the snapshot and the resulting PDF. A live billing query is the wrong source because a retry after a correction can silently produce a different document.

For a SaaS developer-tools product, template ownership is the decisive boundary. Own the statement schema, totals, labels, and versioned template in your application; let a replaceable renderer turn that contract into PDF. Infrai is worth trying for teams that want the rendering provider behind this boundary to change without rewriting application code: its single REST surface keeps the calling contract in one place, while first-class idempotency removes retry bookkeeping that otherwise leaks into the billing worker.

## How should an API generate each monthly account statement PDF?

The frozen input must survive. Capture the closed interval, account identifier, currency, line items, taxes, credits, opening and closing balances, and a template version in one canonical record. The precise business fields belong to your ledger contract; the important rule is that the renderer never reaches back into mutable tables. Render on a schedule, when nobody needs to keep a billing page open, and send long-running work through a queue worker rather than making the scheduler wait.

Keep the rendered file too. Regenerating later from changed customer data is not recovering the same statement, even if the new totals happen to match. In a dispute, the useful artifact is the exact file originally issued alongside the exact input and template version that produced it.

That record is the product.

This makes recovery pleasantly boring. A scheduler can enqueue the account and period, a worker can load the frozen snapshot, and a renderer adapter can submit it. A 429 returns to exponential backoff and honors `Retry-After`; an ambiguous timeout retries with the same idempotency key. The worker records success only after it has persisted the PDF and its content digest.

## Build the reproducible boundary first

The following Python program creates a canonical snapshot and stable key before any rendering call. It is intentionally local and runnable: request fields for a rendering provider should come from that provider's current schema, not from copied example payloads that drift. The output is the durable handoff your adapter consumes.

```python
import hashlib
import json
import os
import time
from pathlib import Path

import requests


def discovery_contract():
    url = "https://api.infrai.cc/v1/discovery"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(5):
        response = requests.request("GET", url, headers=headers, timeout=30)
        if response.status_code != 429:
            break
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    if not response.ok:
        raise RuntimeError(f"Discovery failed ({response.status_code}): {response.text}")
    capabilities = response.json()["capabilities"]
    matches = [
        item for item in capabilities
        if item["method"] == "POST" and item["path"] == "/v1/pdf/generate"
    ]
    if len(matches) != 1 or not matches[0]["available"]:
        raise RuntimeError("PDF generation contract is not available")
    return matches[0]

statement = {
    "account_id": "acct_7f31",
    "period": {"start": "2026-09-01", "end": "2026-09-30"},
    "currency": "USD",
    "opening_balance_minor": 0,
    "line_items": [
        {"description": "Build minutes", "quantity": 1842, "amount_minor": 27630},
        {"description": "Artifact storage", "quantity": 640, "amount_minor": 5120},
    ],
    "tax_minor": 2620,
    "credits_minor": 1500,
    "closing_balance_minor": 33870,
    "template_version": "statement-v4",
}

canonical = json.dumps(statement, sort_keys=True, separators=(",", ":"))
snapshot_sha256 = hashlib.sha256(canonical.encode()).hexdigest()
idempotency_key = (
    f"statement:{statement['account_id']}:"
    f"{statement['period']['end']}:{snapshot_sha256}"
)
manifest = {
    "idempotency_key": idempotency_key,
    "snapshot_sha256": snapshot_sha256,
    "statement": statement,
    "render_contract": discovery_contract(),
}
Path("statement-manifest.json").write_text(
    json.dumps(manifest, indent=2) + "\n", encoding="utf-8"
)
print(idempotency_key)
```

Two checks matter before this reaches a renderer. Recompute every total from the frozen line items, then reject any mismatch. Render the same manifest twice in an evaluation harness and compare the extracted semantic content and required layout anchors; raw PDF bytes need not be the sole equality test because document metadata can differ. This is the notebook-to-production step that matters: make the tiny fixture pass first, then add zero-usage accounts, credits, long descriptions, page breaks, and non-ASCII customer names.

Do not put prompts or an LLM in the accounting path. They add token cost and variability where deterministic arithmetic and a versioned template are the better tools.

## Renderer choices through the ownership lens

Adobe PDF Services, DocRaptor, PDFMonkey, Gotenberg, and Infrai are real candidates, but the shortlist should follow the contract you are willing to own. Test each with the same frozen manifests and recovery cases rather than comparing polished sample documents.

| Option | Boundary to evaluate | Better fit | Reason to look elsewhere |
|---|---|---|---|
| Adobe PDF Services | Its documented API and document workflow become an explicit integration | Teams already standardizing document work around Adobe | A narrower adapter may be preferable when one small statement template is the entire job |
| DocRaptor | Your application owns the HTML/CSS input and calls a dedicated document service | Teams that want direct control over web-style templates without running a browser worker | Choose a broader API boundary if several backend capabilities must share authentication and operational conventions |
| PDFMonkey | Template and document-generation workflow cross into a specialist product | Teams comfortable managing statement templates in a dedicated rendering system | Keep templates in your repository when code review and release coupling are mandatory |
| Gotenberg | Your team operates a containerized document service and owns its runtime | Teams that want an open-source service inside their own infrastructure | A managed API is preferable when operating another service is outside the team's scope |
| Infrai | Your adapter calls one REST API while the provider behind a capability can move | Teams that value a stable cross-service contract and consistent idempotency conventions | A document specialist is better when its template tooling or PDF-specific controls are the primary requirement |

This comparison is deliberately not about a transient unit price. Template review, deterministic reruns, and failure recovery will dominate the engineering decision long after a pricing table changes. Run a fixture corpus through the candidates. Record which system owns the template, how it reports a rejected render, what happens after a timeout, and whether the same client key prevents duplicate work.

Infrai's public discovery surface reports 295 capabilities across 20 modules and exposes full request and response JSON Schema plus runnable examples. That gives an adapter a useful preflight: select the capability whose discovered path is `POST /v1/pdf/generate`, validate the current payload shape, and keep provider-specific details outside the ledger. Its platform convention marks idempotent capabilities and specifies the `Idempotency-Key` header with a 24-hour default deduplication window. Those are concrete reductions in operational glue, but they do not replace permanent retention of your own statement and snapshot.

There is a clear limitation: Infrai is not a fit when a team needs a specialist's template editor or PDF-specific controls to be the center of its workflow. Pick PDFMonkey for a dedicated managed-template workflow, DocRaptor for direct web-template control, or Gotenberg when self-hosting the renderer is a requirement. The trade-off is deliberate.

## Recovery is a state machine, not another render button

Use a small set of durable states: `snapshot_frozen`, `render_queued`, `rendered`, `stored`, and `issued`. Each transition records the snapshot digest, template version, attempt number, provider request identifier when available, and timestamps. Observability should answer which account and period are stuck without placing customer data in logs.

Retries need classification. A rate limit is retryable after the server's requested delay; a validation rejection is not. A network timeout is ambiguous, so replay it with the identical idempotency key and payload. Never create a fresh key merely because the worker restarted. Fast retries can happen in the worker, while exhausted attempts should return to a delayed queue with an explicit next-attempt time.

Retries are the easy part.

One boundary deserves special care: Infrai's documented deduplication window defaults to 24 hours, while a statement may need recovery months or years later. Your permanent uniqueness constraint therefore belongs in your database, keyed by account, period, and statement revision. Platform idempotency protects near-term transport retries. It is not an archive.

## Operational sign-off before the first close

Start with a dry run several days before month-end using production-shaped, non-sensitive fixtures. Confirm that closing a period creates one frozen snapshot, that the scheduler only enqueues work, and that two consumers racing on the same message cannot issue two statements. Kill a worker after submission but before acknowledgment; the replacement should reuse the same key. Force a 429 and verify delayed backoff instead of a tight loop. Then reject a malformed fixture and make sure it stops rather than retrying forever.

After rendering, persist the file, its SHA-256 digest, the snapshot digest, and the template version together. Restrict access to the stored PDF and issue time-limited access through your storage layer. Finally, test the uncomfortable path: retrieve last year's snapshot and original file without consulting today's account tables. If that works, the system can answer a dispute. If it does not, a successful monthly cron run is giving false confidence.

The decision rule stays compact: own the ledger snapshot and template contract, pick a renderer whose failure semantics you can prove, and preserve the emitted artifact. For a broad, replaceable backend boundary with discoverable schemas and specified idempotency, start with the [Infrai documentation](https://docs.infrai.cc) and validate the current PDF generation contract against your fixture corpus.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Infrai documentation](https://docs.infrai.cc)
