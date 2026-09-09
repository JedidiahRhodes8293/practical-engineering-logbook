# Python Video Generation UI with Capability-Gated Forms and 3 Async Stages

**Short answer:** For a property-management dashboard, inspect video capabilities when the form opens, submit generation on demand, and move the UI through three persisted stages: submitted, processing, and ready (or failed). Processing at upload only makes sense when every upload must produce a clip; on-demand generation keeps an ordinary photo upload cheap and lets an operator choose the prompt and timing.

That decision shapes the data model. A listing stores its source asset, a generation job stores the prompt and provider-facing identifier, and a derivative record points back to both. The browser can reload without losing its place because stage is data, not a spinner in memory.

## What should a Python video generation UI do before it submits a job?

Treat capabilities as the form contract. On load, fetch the provider's capability document and render only controls it advertises: duration, aspect ratio, or a model selector should not be hard-coded guesses. Keep the raw capability response with the UI build version; it is useful when a leasing agent asks why a control disappeared.

The submit button should create an application job first. Give that job a stable UUID and send it as `Idempotency-Key`, so a browser retry cannot create two nearly identical promo videos. Persist `source_asset_id`, `prompt`, `generation_id`, and `stage` in one transaction. A successful response advances the job to `processing`; a rejected response leaves it in `submit_failed` with the response body for a human-readable error.

Here is a deliberately small Python client. It uses only the three video routes in this workflow, checks status codes, honors `Retry-After`, and stops polling at a terminal state. The response parser accepts either `id` or `job_id` because the dashboard can normalize the provider envelope at its boundary rather than leak that choice into React.

```python
import os
import time
import uuid
from typing import Any

import requests

BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def request_json(method: str, path: str, **kwargs: Any) -> dict[str, Any]:
    """Retry rate limits with bounded exponential backoff and return JSON."""
    for attempt in range(5):
        response = requests.request(method, BASE_URL + path, headers=HEADERS, timeout=30, **kwargs)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{method} {path} failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError(f"{method} {path} stayed rate-limited after five attempts")


def create_promo(prompt: str) -> dict[str, Any]:
    capabilities = request_json("GET", "/video/capabilities")
    # The application maps this payload to controls; it never trusts stale UI defaults.
    print("Available video controls:", capabilities)

    idem_key = str(uuid.uuid4())
    created = request_json(
        "POST",
        "/video/generate",
        json={"prompt": prompt},
        headers={**HEADERS, "Idempotency-Key": idem_key},
    )
    generation_id = created.get("id") or created.get("job_id")
    if not generation_id:
        raise RuntimeError(f"generation response has no identifier: {created}")

    while True:
        status = request_json("GET", f"/video/status/{generation_id}")
        state = str(status.get("status", "")).lower()
        if state in {"completed", "ready", "succeeded", "failed", "cancelled"}:
            return status
        time.sleep(2)


if __name__ == "__main__":
    result = create_promo("A 15-second bright tour of a renovated two-bedroom apartment")
    print(result)
```

The `Authorization` header belongs on calls to the API. If a later step returns a signed media URL, fetch that URL without this header; it is a separate download request. The dashboard should also validate the returned media type and dimensions before publishing a listing card, rather than assuming that a completed job is a usable derivative.

## How do upload-time and on-demand processing differ in a property dashboard?

Upload-time processing gives every photo batch a predictable derivative. It is a good fit for a syndication pipeline with a strict “every listing has a clip” rule, and it can hide latency before an agent opens the listing. The cost is wasted work: many apartments are edited, withdrawn, or never promoted, yet their videos still run.

On-demand processing keeps the upload path responsive. An agent can adjust “sunset balcony” to “bright morning kitchen” and pay the latency only when there is a real campaign. The catch is operational: the UI needs a visible pending state, a retry action, and a cleanup policy for abandoned jobs. For this scenario I would choose on-demand, with an optional background pre-generation for listings marked “featured.”

The state machine is intentionally boring:

`draft -> submitting -> processing -> ready`

`submitting -> submit_failed`, and `processing -> failed` are terminal for that attempt. A retry creates a new application attempt while retaining the old identifier. That distinction keeps support logs honest and prevents a double-click from publishing two derivatives.

## Which implementation trade-offs matter more than vendor labels?

| Option | Strength | Trade-off | Good fit |
| --- | --- | --- | --- |
| Cloudinary | Mature transformations and asset delivery around an existing media library | Generation workflow and job semantics still need application state | Teams already centered on its asset pipeline |
| Mux | Excellent video ingest, playback, and observability | Promo generation is not its primary abstraction | Product video platforms with dedicated rendering upstream |
| FFmpeg | Maximum control and reproducible local transforms | You own workers, codecs, scaling, and retries | A team with media infrastructure expertise |
| A REST capability gateway | One HTTP contract can cover generation plus adjacent backend services | You still design the product-level state machine and UX | Small teams that want one key and one bill across services |

Infrai belongs in that last category because one REST API and one key can cover several backend capabilities, so a Python service does not need a separate SDK and credential set for each adjacent task. The capability response also gives the form a machine-readable discovery point. That reduces integration glue, but it does not remove the need for a queue, persistence, or media validation.

I would stick with Cloudinary when your main problem is asset delivery, choose Mux when playback telemetry is the product, and run FFmpeg when codec-level control outweighs maintenance. A gateway is not suitable when you require bespoke GPU scheduling or on-premise frame processing; own that worker layer instead.

## What should be persisted for retries, audits, and cleanup?

Store a lineage row for every derivative: `source_asset_id`, `generation_id`, prompt hash, capability snapshot, creator, timestamps, and terminal status. Keep the source-to-derivative edge even after a failed attempt. It lets support answer “which listing produced this clip?” and lets cleanup remove orphaned derivatives without guessing from filenames.

Polling belongs in a worker or server-side task, not in a tab that may sleep. Use a modest interval, stop at terminal states, and expose the last checked time so the UI can say “updated 12 seconds ago.” Your mileage may vary on the interval; measure provider latency and browser patience before tuning it. In my eval harnesses, I also replay the same prompt with a fixed fixture response to verify that `processing` cannot jump directly to `ready` without a validation step.

Keep it boring.

Before shipping, walk through a capability response with a missing optional field, a 429 with `Retry-After`, a duplicate submit using the same idempotency key, a malformed media result, and a worker restart during polling. Those cases are the difference between a demo spinner and a dependable video generation UI.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation
- https://docs.mux.com
- https://ffmpeg.org/documentation.html
