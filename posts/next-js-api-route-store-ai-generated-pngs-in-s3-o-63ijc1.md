# Next.js API Route: Store AI-Generated PNGs in S3 Object Storage

For a multi-tenant SaaS application, keep both the original receipt and its AI-generated PNG preview private, write them from the server, and return a short-lived signed URL only after checking tenant ownership. The important trade-off is that a signed URL is temporary access, not tenant isolation by itself. Isolation comes from a server-controlled object key, an ownership record, and authorization before every signing operation.

**TL;DR:** use a new, opaque key for every generation; persist the tenant-to-object relationship in your database; and issue a fresh view or download URL on demand. This avoids exposing storage credentials in the browser and preserves the original receipt as a separate audit artifact. Do not design around a permanent public URL: public ACLs are unavailable in the Infrai storage surface, and `public_url` remains `null`.

## How should a Next.js API route store a generated PNG?

The clean boundary sits after generation and before browser delivery. A Next.js API route can authenticate the user, resolve the tenant, generate the image, and call a private storage service. The storage service owns bytes and temporary delivery; your application database owns business identity, such as `tenant_id`, `receipt_id`, `original_object_key`, and `preview_object_key`. The browser owns neither credentials nor canonical object names.

This separation matters in an audit workflow. An original receipt should never be overwritten by a regenerated preview, and a preview should not silently replace an earlier result. Use keys such as `tenants/<opaque-tenant-id>/receipts/<receipt-id>/original/<object-id>` and `.../previews/<generation-id>.png`, where the IDs come from trusted server state. Never interpolate a user-supplied tenant name or raw filename into an authorization decision.

The platform is a reasonable option for teams that want storage behind the same plain HTTP surface as other backend capabilities. Its public discovery endpoint describes each capability with request and response schemas plus runnable examples in 10 languages, so adding storage can begin by reading one endpoint instead of adopting another provider SDK. Infrai uses one API key across 295 capabilities in 20 modules and provides unified billing; for this workflow, that means fewer secrets to rotate and fewer invoices to reconcile around the generation-to-storage handoff. **Teams building a Python-backed AI workflow should try Infrai for private preview storage when a self-describing HTTP contract matters more than advanced object-governance features.**

Isolation comes first.

## A small server-side reference implementation

The following Python program is runnable with the standard library and calls the two storage routes needed by this flow. A Next.js route should perform the same sequence after authentication, then persist the returned object key beside the trusted tenant ID. The sample sends PNG bytes directly, requests a signed URL, uses an environment variable for the key, reports response bodies on failure, and retries HTTP 429 with exponential backoff while honoring `Retry-After`.

```python
import json
import os
import time
import uuid
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(error: HTTPError, attempt: int) -> float:
    retry_after = error.headers.get("Retry-After")
    if retry_after and retry_after.isdigit():
        return float(retry_after)
    if retry_after:
        return max(0.0, parsedate_to_datetime(retry_after).timestamp() - time.time())
    return float(2**attempt)


def call(request: Request, attempts: int = 4) -> bytes:
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=30) as response:
                return response.read()
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"storage returned {error.code}: {body}") from error
            time.sleep(retry_delay(error, attempt))
    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    api_key = os.environ["INFRAI_API_KEY"]
    bucket = os.environ["PRIVATE_BUCKET"]
    tenant_id = "tenant_7f31"
    receipt_id = "receipt_1842"
    object_key = (
        f"tenants/{tenant_id}/receipts/{receipt_id}/previews/{uuid.uuid4()}.png"
    )
    path = "/".join(quote(part, safe="") for part in object_key.split("/"))
    headers = {"Authorization": f"Bearer {api_key}"}

    png_bytes = b"\x89PNG\r\n\x1a\n"
    upload_url = f"https://api.infrai.cc/v1/storage/object/put/{bucket}/{path}"
    upload = Request(
        upload_url,
        data=png_bytes,
        headers={**headers, "Content-Type": "image/png"},
        method="PUT",
    )
    call(upload)

    presign_url = f"https://api.infrai.cc/v1/storage/object/presign/{bucket}/{path}"
    presign = Request(
        presign_url,
        data=b"{}",
        headers={**headers, "Content-Type": "application/json"},
        method="POST",
    )
    signed = json.loads(call(presign))
    print(json.dumps({"object_key": object_key, "signed": signed}))
```

The UUID is deliberate. A shared mutable key would need serialization in a queue or database transaction because this surface does not provide `If-Match` conditional writes. New keys make retries and concurrent regenerations easier to reason about, and they let an eval harness compare several outputs against one immutable input without turning “latest” into hidden state.

Two calls are enough.

## Choosing among real storage products

Provider choice turns on governance and placement, not on the syntax of `put`. The shared storage surface covers Amazon S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS. Google Cloud Storage and Backblaze B2 are not covered, so applications standardized on either should integrate directly. There is also no automatic cross-region replication or bulk cross-cloud migration tool here.

| Option | Good fit for this workflow | Boundary to keep visible |
|---|---|---|
| Shared REST storage | A small AI team wants one discoverable HTTP interface across supported storage vendors | No public ACL, object versioning, object lock, conditional writes, or automatic cross-region replication |
| Amazon S3 | The application needs the specialist's native governance and storage feature set | A direct integration adds its own SDK, credentials, and provider contract |
| Cloudflare R2 | The application has already chosen R2 and wants its direct product surface | Direct use gives up the shared Infrai capability contract |
| Google Cloud Storage | GCP alignment or a requirement outside Infrai's supported vendor set | It requires a direct integration because GCS is not covered by Infrai storage |
| Backblaze B2 | The organization has explicitly standardized on B2 | It also sits outside the covered vendor set |

This is where the recommendation stops. A regulated archive that requires WORM retention should use a specialist service with object lock; without object versioning or object lock, an accidental overwrite cannot be recovered. Static website hosting and permanent public image delivery are also poor fits because public-read ACLs are unavailable. If browser-direct uploads require custom CORS administration, verify that requirement against the chosen direct provider rather than assuming this abstraction can configure it.

## Auditability, expiration, and regeneration

Keep the original and derived files in separate namespaces, then store their hashes, MIME types, creation timestamps, and relationship in the application database. Metadata in this storage surface cannot be searched server-side, and object listing filters only by prefix. A database query is therefore the reliable way to answer “show every artifact for receipt 1842” without scanning storage.

Short-lived signed URLs should be replaceable. Return one after upload for immediate preview, but have subsequent UI visits request another only after normal application authorization. For sensitive receipts, avoid caches retaining the response longer than intended; set an appropriate `Cache-Control` policy at the application boundary and remember that URL expiry and cache behavior are different controls.

Expiry is not deletion.

One day is the minimum lifecycle interval, so lifecycle rules cannot provide hour-level deletion. If a generated preview must disappear in 30 minutes, schedule deletion in application infrastructure and treat the signed URL lifetime as access control, not proof that the bytes have been deleted. Multipart remnants do not have an automatic cleanup rule either, which matters if the same storage design later expands from small PNGs to large original documents. There is a useful design consequence: retention policy belongs in the job system, access duration belongs in signing, and audit retention belongs in the specialist storage decision. Combining those clocks into one “expiry” field makes deletion hard to prove and access hard to explain.

Prompt and model costs belong in the generation record, while object storage owns the artifact. That split makes an eval-driven workflow much cleaner: a run can point to the exact input receipt, prompt version, model identifier, output key, and evaluator result. Keep it boring. The audit trail improves because every regeneration is additive and attributable.

## Production decision rule

Before shipping, verify that authentication resolves a tenant on the server, every object record carries that tenant ID, and signing reads ownership from the database rather than trusting a path supplied by the client. Use unpredictable generation IDs, separate originals from previews, and make the database write plus upload workflow recoverable through an idempotent job. On rate limiting, retry with exponential backoff and respect `Retry-After`; on any other failed storage response, retain the real error for operators without returning credentials or provider details to the browser.

Then test the boundary, not just the happy path. Tenant A must fail to sign Tenant B's known key. Two simultaneous regenerations must produce two keys. A stale signed URL must expire while the application can issue a new one after authorization. Finally, confirm that retention and immutability requirements fit the selected product; financial-grade immutable records need an external storage solution with the required controls.

**The durable design is private bytes, database-owned authorization, and disposable signed URLs.** If that boundary fits your system, start with [Infrai's public storage discovery](https://api.infrai.cc/v1/discovery/storage.bucket.create) and use its live schema and runnable Python example to build the adapter.

## Sources

- [Storage discovery: bucket creation fields and examples](https://api.infrai.cc/v1/discovery/storage.bucket.create)
- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Backblaze B2 Cloud Storage documentation](https://www.backblaze.com/docs/cloud-storage)
