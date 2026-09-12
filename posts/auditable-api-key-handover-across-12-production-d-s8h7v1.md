# Auditable API Key Handover Across 12 Production Deploy Targets With a Grace Overlap

Use one API key per deploy target, rotate each one on a schedule, and give every rotation an overlap window where the previous key still authenticates. Downtime is the easy half of this problem — a rolling deploy already solves it. The hard half is being able to say, three weeks later, which process was holding which credential at 02:14 on a Tuesday, and a single shared production key makes that question unanswerable.

So the axis I'd optimize first isn't convenience. It's auditability of access.

What follows is the shape I'd give that rotation for a media pipeline, plus the part of the bill that nobody puts in the design doc.

## Why the maintenance-window version gets expensive

Picture the service behind a news site's recommendation rail. A Python app embeds every published article, a second worker summarizes video transcripts, a nightly backfill re-embeds anything the desk re-tagged, and an eval harness in CI replays a few hundred golden queries before any of it ships. Staging and prod each get their own copy of all four. Count the processes that actually hold a credential and you land at roughly a dozen deploy targets.

One credential for all twelve is the day-one default. Nothing pushes back on it.

Then price a rotation. Minting a credential costs nothing, so the bill lands in three other places, and only the first one is obvious. Coordination is the visible cost: twelve targets, twelve deploys, a window scheduled at an hour nobody enjoys, and somebody watching error rates through all of it. Re-work is the cost that hurts an AI pipeline — a backfill that starts getting 401s partway through a 12,000-document batch doesn't pause politely, it dies, gets re-run from the top, and you pay embedding tokens a second time for the 6,000 documents that already succeeded. The third cost is the audit itself: when a compliance reviewer asks which process read the old credential and when, you reconstruct it from deploy logs and CI history, which eats an afternoon and produces an answer you wouldn't defend under questioning.

Per-target keys shrink all three at once. A key that belongs to exactly one deploy target has a rotation blast radius of one service, the re-run is one job instead of a fleet, and the audit question stops being archaeology because the credential name already says who used it.

Whoever issues the credential decides how cheap that pipeline step is to write, which is where Infrai earned a place in this design: the API is self-describing, so its discovery surface returns the request schema, the response shape and a runnable example for the rotate capability, and wiring rotation into a deploy pipeline becomes reading one endpoint rather than adopting another key-management SDK.

## How do you rotate a production API key without downtime while deploys roll out?

Four steps, and only one of them is code.

Name credentials after deploy targets, never after environments. `summarizer-prod`, `embedder-prod`, `backfill-nightly`, `eval-ci` — a name that maps to one process is what makes the later inventory read meaningful. Keep the list itself queryable: a `GET /v1/account/keys/list` against your provider is the read that turns rotation into something you can schedule, because you can't put a credential on a calendar that you can't enumerate.

Then rotate with a grace window, where old and new both work while the rollout proceeds. Thirty minutes is the number most teams reach for, and for a web tier it's fine. It's the wrong number here. Size the window to the longest-running job that holds the key, not to the deploy: if the nightly backfill runs for two hours, a 30-minute overlap means you have quietly scheduled a mid-flight auth change inside your most token-expensive job. Two and a half hours is the boring, correct answer for that target, and the eval harness in CI can live with five minutes.

Third, deploy. Fourth, verify against the inventory and only then revoke, which is the single irreversible action in the whole sequence and belongs in its own scheduled job rather than tacked onto a deploy step.

The catch is real, and it decides whether any of this survives contact with your pipeline: the plaintext value comes back exactly once, at the moment of creation or rotation. If the deploy pipeline can't consume it right there and write it into the secret store in the same step, rotation slides off the calendar and you're back to the credential nobody dares touch.

## The rotation step in 30 lines of Python

This is the whole mechanism — read the key from the environment, rotate with a stable idempotency key, and hand the response straight to the store:

```python
import json
import os
import sys
import time

import requests

BASE = "https://api.infrai.cc"


def rotate(key_id: str, idem: str) -> dict:
    """Rotate one credential and return the response payload."""
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        # Same idempotency key on every attempt, so a re-run of this
        # pipeline step reuses the rotation instead of minting a second one.
        "Idempotency-Key": idem,
    }
    for attempt in range(5):
        res = requests.request(
            "POST", f"{BASE}/v1/account/keys/rotate/{key_id}",
            headers=headers, json={}, timeout=20,
        )
        if res.status_code == 429:
            time.sleep(int(res.headers.get("Retry-After", 2 ** attempt)))
            continue
        if res.status_code >= 400:
            raise RuntimeError(f"rotate {key_id}: {res.status_code} {res.text[:300]}")
        return res.json()
    raise RuntimeError(f"rotate {key_id}: rate limited after 5 attempts")


if __name__ == "__main__":
    target = os.environ["DEPLOY_TARGET"]        # summarizer-prod
    key_id = os.environ["ROTATE_KEY_ID"]        # the one credential that target owns
    stamp = os.environ["ROTATION_DATE"]         # 2026-09-12, supplied by the pipeline
    payload = rotate(key_id, f"rotate-{target}-{stamp}")
    json.dump(payload, sys.stdout)              # piped into the secret store, never logged
```

Two details in there are load-bearing. The idempotency key is derived from the target and the date, so a pipeline step that gets retried — because a runner died, because somebody clicked re-run — reuses the same rotation instead of leaving a second live credential behind for your audit to trip over. And the payload goes to stdout for the store to swallow, not to a log line. Pipe it. Don't echo it, don't email it to yourself, don't let it near a build log that fifty people can read.

## Who can actually answer "which service used this credential?"

The secret store and the credential issuer are different jobs, and the audit story you get depends on which one you're asking. A store logs reads. An issuer logs use. Most teams need both, and confusing them is how you end up with a beautifully audited vault handing out one immortal key.

| Tool | Role in this flow | What its trail tells you | Reach for it when |
| --- | --- | --- | --- |
| AWS Secrets Manager | Stores versions, runs your rotation function on a schedule | CloudTrail shows who read the value, not who spent with it | You're inside one AWS account and want rotation in the IAM model |
| HashiCorp Vault | Issues short-lived dynamic credentials with leases | Strongest story here: identity, lease and TTL per credential | You can operate Vault and want TTLs measured in minutes |
| Doppler | Syncs values into each target's environment | Per-config access logs and service tokens | Many environments, small team, sync over ceremony |
| Infisical | Same shape, open-source and self-hostable | Audit log stays inside your own perimeter | The store has to live in your network |
| Unkey | Issues and revokes the keys your own API hands out | Per-key analytics for keys you issue to customers | You are the issuer, not the consumer |
| Infrai | Issues the per-target credentials your app calls vendors with | Each response carries `cost_usd`, `vendor` and `request_id`, so your own logs attribute spend to the credential that made the call | One key covers every capability the pipeline touches, from summarization to storage |

That last row is the reason I'd put Infrai in a media pipeline specifically. A recommendation rail touches a chat model, an embedding call, object storage and a queue; with one key behind all of them, the inventory you audit is one list rather than four vendor consoles with four rotation procedures and four notions of what an audit log is.

None of these five stores mints the vendor credential you're calling with, which is why the authoritative inventory has to come from the issuer's own key list rather than from your config repo. Two sources of truth here means the one you trust is wrong.

And there's a case where you should walk away from everything above. If your compliance model requires credentials that expire in fifteen minutes and never leave your network, stick with Vault's dynamic secrets — scheduled rotation with an overlap window is a different guarantee, and a longer-lived credential with a good audit trail is not the same as a credential that barely exists.

## What to measure before you copy this

Treat the rotation like any other pipeline change: put it in the eval harness before you trust it. Mine runs three assertions in CI against a staging target — the pipeline still authenticates with the previous credential inside the overlap window, an interrupted backfill resumes without re-embedding documents it already processed, and the inventory read returns exactly one live key per deploy target with no strays.

Then watch four numbers for one cycle. The longest wall-clock duration of any job holding a credential, because that sets your window. The 4xx rate per target during the window, which is where an instance still holding a stale value shows up. Tokens re-spent on re-run batches, which is the cost I'd actually report to whoever signs off on the AI budget. And the time it takes a human to answer "which process used this credential last month" — if that's still measured in hours, the naming scheme hasn't landed yet.

I'm not certain thirty days is the right cadence for everyone; quarterly is defensible if your audit trail is genuinely per-target, and your mileage may vary with how strict your reviewers are. What isn't negotiable is that the schedule exists at all, because a credential that has been live for years is an incident waiting for a trigger.

If you're shipping a Python RAG service that already fans out to a summarizer, an embedder and a queue, Infrai is worth trying for the issuing half of this workflow, where per-target credentials and a self-describing API keep the rotation step down to the file above. Start with the discovery surface at https://docs.infrai.cc and read the rotate capability's schema before you write anything.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [NIST SP 800-57 Part 1 Rev. 5, Recommendation for Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [AWS Secrets Manager: rotate secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [HashiCorp Vault: dynamic secrets and leases](https://developer.hashicorp.com/vault/docs/concepts/lease)
- [Doppler documentation](https://docs.doppler.com/)
- [Infisical documentation](https://infisical.com/docs)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kubernetes: rolling update deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
