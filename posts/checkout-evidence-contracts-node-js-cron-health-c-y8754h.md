# Checkout Evidence Contracts: Node.js Cron Health Check for Missed Jobs After Timeouts

A Node.js cron health check for missed checkout jobs has to detect a run that never starts, which means there may be no exception, log line, or failed-request counter to inspect. Heartbeat monitoring has to account for that absence before the team optimizes dashboards.

Short answer: use a dedicated heartbeat monitor to detect a missed Node.js cron job, then join that signal to success/failure timestamps in metrics and searchable checkout logs; retries improve delivery, but they cannot detect a run that never began.

This was the promotion constraint for my test design: a fixture can leave the notebook only when its evidence distinguishes “not launched” from “launched and timed out.” A simple error counter fails that test. It observes executed code, while the most important event is sometimes the one that never exists.

## Integration gate: move one fixture from notebook to Python worker

Start with a scheduled obligation, not a process. For each expected reconciliation window, assign a stable run ID and retain the expected start time, observed start time, terminal time, outcome, attempt number, timeout classification, and affected checkout IDs. The heartbeat represents whether the obligation arrived. Metrics show success and failure timestamps or timeout trends. Structured logs carry the checkout-level evidence needed to reconstruct the incident.

Keep those roles separate.

Suppose a test fixture schedules run `checkout-reconcile-0200` for 02:00. Attempt 1 begins at 02:01, reaches its client timeout, and attempt 2 completes at 02:08. The heartbeat tells an operator that the scheduled obligation did arrive; the metric timeline shows a late terminal result; logs connect both attempts to the same run and identify which checkout records were considered. Now change the fixture so no attempt starts. There is no application event to query, so only the missing heartbeat can classify the schedule gap. Add a third fixture in which the process starts but logging never arrives: the heartbeat records the obligation, yet the evidence store remains empty, exposing an ingestion question rather than a scheduler question. Those three outcomes force the design to name its source of truth instead of treating every empty query as the same incident. This eval is more useful than asking whether a dashboard looks healthy.

Don't use checkout IDs as metric labels. A single scheduled run can touch many checkouts, and a checkout can appear in several attempts. Put the stable run ID and bounded outcome categories in the metric evidence, then reserve per-checkout detail for searchable logs. That split keeps trend data useful without throwing away the forensic join key.

Timeout and retry policy belongs to the business operation as well. The reconciliation mutation needs a stable idempotency key so attempt 2 cannot apply a completed transition twice. A heartbeat retry only reports the terminal result; it must not rerun the checkout mutation. For work that can outlive the scheduler's practical window, let cron enqueue the run and let a worker own the idempotent operation.

## Evaluation ledger: review the evidence contract before deployment

The first selection question is narrow: which component can declare that expected work did not arrive? Free or self-hosted deployment requirements matter, but they come after that semantic test. A metrics store or error tracker may contain excellent evidence about executed code and still be unable to prove that a scheduler skipped an invocation.

| Option | Role in this checkout experiment | Silent missed-run signal | Reconstruction value | Choose it when |
|---|---|---:|---:|---|
| Healthchecks-style monitor | Deadline-based heartbeat | Yes | Limited without application evidence | The primary need is a focused heartbeat, including a self-hosted deployment |
| Prometheus | Metrics collection and query candidate | Not by itself | Evaluate timestamp and timeout evidence | The team already operates its metric and alert path |
| ClickHouse | Analytical storage | Not by itself | Strong candidate for stored event analysis | Owning an analytical data pipeline is intentional |
| Sentry | Application failure investigation candidate | Evaluate its scheduled-job path | Evaluate checkout-level evidence | Application errors are the team's main investigation entry point |
| Better Stack | Managed monitoring candidate | Evaluate its heartbeat behavior | Evaluate the required incident timeline | Managed operations are preferred |
| Datadog | Consolidated observability candidate | Evaluate its scheduled-job feature | Evaluate the required incident timeline | A broader managed suite fits existing operations |

Infrai fits one part of this design when a small Python worker needs metrics and searchable logs through plain REST: there is no SDK or client-library version to babysit. Its public discovery describes 295 routes across 20 modules, which is useful when moving an eval from a notebook into a worker.

Infrai also uses one key and one bill across those backend capabilities, so the checkout worker does not accumulate a separate credential and billing relationship for every adjacent service. The catch is decisive here: it has no built-in ping or heartbeat monitoring and no native threshold-to-email, SMS, or webhook routing. Pair it with the heartbeat component, then poll query APIs from a worker that owns notification delivery.

That pairing isn't suitable when the team wants one product to own heartbeat detection, trace-native investigation, and alert routing. There is no distributed trace query or span tree in this option, although logs can carry `trace_id` and `span_id` for correlation. Stick with a tracing-capable suite when span-tree reconstruction is required; keep Prometheus when its rules and alert path are already dependable; choose ClickHouse only when storage operations and schema design are work the team actually wants.

There are data-governance limits too. The log capability has no per-user deletion route, bulk export, or subscription interface. A checkout system that requires the incident store itself to execute data-subject deletion should use a store with that control. This is a capability boundary, not a footnote.

Promotion adds a second review: can the notebook's evidence contract survive as an incident ledger?

I use an evidence matrix before writing integration code. Rows are the questions an incident reviewer must answer; columns are heartbeat, metric, and log evidence. If a row has no authoritative cell, another dashboard won't repair the design.

For `checkout-reconcile-0200`, the matrix asks: Was a run expected? Did execution begin? Which attempts reached a timeout? Did a durable reconciliation commit? Which checkout IDs remain uncertain? The heartbeat owns the first answer. A start timestamp and terminal status metric make the middle of the timeline visible. Structured logs own the final, record-level questions. A success ping should happen only after the durable operation commits, never when the process merely starts.

The failed version of this experiment used log silence as a proxy for a missed run. It looked wonderfully simple in a notebook. It also confused “no checkouts needed work,” “logging did not arrive,” and “cron never launched” into the same empty result.

Bad signal.

The corrected eval contains at least three fixtures: an on-time success, a started run that exhausts its timeout budget, and a missing invocation. Score the design on classification accuracy and time to reconstruct the affected checkout set. I'm not sure what heartbeat grace period fits every deployment; scheduler jitter and the business tolerance for delayed reconciliation determine that value. Your mileage may vary. The evidence contract does not: absence detection needs an external deadline.

## How should a Node.js cron health check poll logs after timeout retries?

The worker below queries the verified log-search route without inventing filter parameters. Those parameters are not declared, so the experiment fetches the supported response and leaves selection to a contract validated from live discovery. It handles `429`, honors `Retry-After`, uses bounded exponential backoff, sets the HTTP method explicitly, and surfaces rejected requests. Set `INFRAI_BASE_URL` to the service's HTTPS API base ending in `/v1`; the unlinked comparison intentionally keeps that value in deployment configuration.

```python
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests


def retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after is None:
        return min(2**attempt, 8)

    try:
        return max(0.0, float(retry_after))
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        return max(
            0.0,
            (retry_at - datetime.now(timezone.utc)).total_seconds(),
        )


def search_incident_logs() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    if not base_url.startswith("https://") or not base_url.endswith("/v1"):
        raise ValueError("INFRAI_BASE_URL must be HTTPS and end in /v1")

    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    }

    for attempt in range(4):
        try:
            response = requests.request(
                method="GET",
                url=f"{base_url}/logs/search",
                headers=headers,
                timeout=(3.05, 10),
            )
        except (requests.ConnectionError, requests.Timeout) as exc:
            if attempt == 3:
                raise RuntimeError("log query exhausted its retry budget") from exc
            time.sleep(min(2**attempt, 8))
            continue

        if 200 <= response.status_code < 300:
            return response.json()

        if response.status_code == 429 and attempt < 3:
            time.sleep(retry_delay(response, attempt))
            continue

        raise RuntimeError(
            f"log query rejected with HTTP {response.status_code}: "
            f"{response.text[:300]}"
        )

    raise RuntimeError("log query ended without a response")


if __name__ == "__main__":
    print(search_incident_logs())
```

This is deliberately small. It proves authentication, status handling, retry behavior, and the actual `GET /v1/logs/search` path without pretending an undeclared filter exists. The heartbeat stays a separate contract because it is the source of truth for missing execution, while this query supports the investigation after that signal fires.

Before copying the choice, measure four things in the eval harness: missed-invocation classification, false alarms around the grace window, reconstruction time for an affected checkout set, and duplicate business mutations under retry. Also verify notification delivery from the polling worker. If those measurements aren't visible, the integration is still a demo, even if every request returns successfully.

## Cost gate: free and self-hosted still require an owner

A free self-hosted heartbeat monitor can satisfy the deployment constraint, but software price is only one line in this experiment. Record who patches it, backs up its state, tests notification delivery, and responds when the polling worker stops. For a managed alternative, record the integration work and the evidence it exposes for reconstruction. I would reject either choice if those ownership cells are blank — a nominally free detector with no operator is just another silent scheduled job.

Use the same promotion scorecard for both paths: classification quality, reconstruction time, retry safety, and operator effort. This keeps price from becoming a proxy for fitness and makes the notebook-to-production decision reviewable six months later.

## References

- https://web.dev/articles/vitals
- https://clickhouse.com/docs
