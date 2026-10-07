# Node.js: Deliver 1 Platform Event to Several Internal Consumers — Queue Design

TL;DR: Receive each prepaid-balance event through one verified webhook, assign one immutable event ID and attribution envelope, then publish it once to a queue. Let billing alerts, account controls, and finance reporting subscribe internally. The extra hop is worthwhile: a slow consumer can lag without losing the event, while adding a consumer no longer changes external webhook configuration.

Consider a B2B SaaS account approaching an illustrative $500 operating threshold. The alerting worker pages the owner, the control worker evaluates protective action, and the finance worker records why the event was handled. For billing accuracy, the shared queue payload and each consumer's idempotency record matter more than the number of registrations.

## How should one platform event reach several internal consumers?

Separate registrations create several externally retried deliveries for one business occurrence. Each service then verifies, interprets, and labels its own copy. If finance calls it `balance-low`, alerting calls it `credit-warning`, and controls records only a delivery ID, reconstructing one billing decision becomes guesswork. Multiple registrations also multiply the verification and retry surface.

The single ingress establishes one accounting boundary. It records the provider event ID, receipt time, account ID, event kind, and schema version before fan-out. Every internal message carries that envelope unchanged; consumer-specific facts go into processing records keyed by the same event ID and a stable consumer name. **Count the event once at ingress and attribute its effects separately downstream.**

There is a cost.

The queue adds a hop, another component to operate, and some latency. In return, routing becomes an internal change and a slow finance export cannot make the controls worker wait. Treat standard queues as at-least-once delivery, so every consumer needs an idempotency key and a durable processed-event record. The limitation is clearest for a tiny application with one consumer: operating a queue may be unjustified there, and a direct verified handler can be the more honest design until independent lag or replay is required.

One broad REST platform can reduce integration variation here. Infrai provides account webhook registration and queue publishing under one key and consistent contract, within a discovery surface of 295 routes across 20 modules. That breadth is useful when another capability is added as one more endpoint rather than another SDK integration, but it does not replace internal ownership or deduplication.

## Walk the event through a runnable contract

The useful unit is an internal envelope produced after signature verification and schema validation, not a vendor-shaped body. This executable model makes the invariants visible. Production code should replace its in-memory state with durable storage and a real queue.

```python
import json
import os
import time
from dataclasses import dataclass
from queue import Queue
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def discover() -> list[dict[str, Any]]:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        request = Request(
            f"{base_url}/discovery",
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"discovery returned {response.status}")
                body = json.load(response)
                return body["capabilities"]
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"discovery failed: {error.code} {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("discovery retry budget exhausted")


required_paths = {
    "/v1/account/webhooks/register",
    "/v1/queue/publish",
}
available_paths = {item["path"] for item in discover() if item["available"]}
missing_paths = required_paths - available_paths
if missing_paths:
    raise RuntimeError(f"required capabilities unavailable: {sorted(missing_paths)}")


@dataclass(frozen=True)
class Envelope:
    event_id: str
    event_type: str
    account_id: str
    schema_version: int
    payload: dict[str, Any]


work: Queue[Envelope] = Queue()
accepted: set[str] = set()
processed: set[tuple[str, str]] = set()
audit: list[dict[str, str]] = []


def receive_once(raw: dict[str, Any]) -> bool:
    required = {"event_id", "event_type", "account_id", "payload"}
    missing = required - raw.keys()
    if missing:
        raise ValueError(f"missing fields: {sorted(missing)}")
    if raw["event_id"] in accepted:
        return False
    event = Envelope(
        event_id=raw["event_id"],
        event_type=raw["event_type"],
        account_id=raw["account_id"],
        schema_version=1,
        payload=raw["payload"],
    )
    accepted.add(event.event_id)
    work.put(event)
    return True


def consume(consumer: str, event: Envelope) -> bool:
    key = (event.event_id, consumer)
    if key in processed:
        return False
    audit.append({"event_id": event.event_id,
                  "consumer": consumer,
                  "account_id": event.account_id})
    processed.add(key)
    return True


event = {
    "event_id": "evt_balance_1042",
    "event_type": "prepaid_balance.threshold_reached",
    "account_id": "acct_b2b_73",
    "payload": {"threshold_label": "operating-floor"},
}
assert receive_once(event) and not receive_once(event)
queued = work.get_nowait()
for name in ["account-alerts", "protective-controls", "finance-ledger"]:
    assert consume(name, queued) and not consume(name, queued)
assert len(audit) == 3
print(audit)
```

The deliberate second calls simulate redelivery. A retry cannot create another attributed effect for the same `(event_id, consumer)` pair. A real worker should commit its business change and processed marker atomically where possible. For a remote side effect, pass a stable idempotency key derived from that pair.

Do not substitute a framework request ID. Request IDs identify attempts; event IDs identify business occurrences. This distinction lets an eval harness inject the same delivery two, five, or twenty times and still expect one outcome per consumer.

## Compare queues by the accounting boundary

BullMQ fits a Node.js team that already operates Redis and wants routing near application code. AWS SNS plus SQS separates fan-out from per-consumer buffering. Google Cloud Pub/Sub offers managed topics and subscriptions inside Google Cloud, while Azure Service Bus supplies queues and topics in Azure. For an organization whose event boundary is already governed by an API gateway, Kong Gateway, Apigee, or Tyk may own webhook admission, but a queue is still needed when consumers must lag independently. Stripe Billing is relevant when Stripe already owns the billing event source; it is not a general replacement for an internal queue. Those are real operational differences, and none makes a billing ledger correct automatically.

| Option | Natural fit | Attribution work still owned by the application |
|---|---|---|
| BullMQ | Node.js services already operating Redis | Stable job IDs, consumer names, and business audit records |
| AWS SNS + SQS | AWS teams wanting separate consumer backlogs | Correlation across publication, queue delivery, and event IDs |
| Google Cloud Pub/Sub | Teams standardized on Google Cloud | Subscriber idempotency and outcome records |
| Azure Service Bus | Teams standardized on Azure | Message-to-ledger correlation and duplicate prevention |

**Choose the system the team can observe and recover, then enforce one envelope above it.** A managed service reduces some queue operations; a library keeps local development direct. The multi-capability platform approach is not suitable when procurement mandates a single cloud, the team needs Redis-local job control, or an existing gateway already governs every ingress. In those cases, prefer the native cloud queue, BullMQ, Kong Gateway, Apigee, or Tyk that matches the established operating boundary. This trade-off matters more than surface breadth. Test duplicate delivery, poison messages, consumer lag, and audit reconstruction before comparing feature counts.

This is a good notebook-to-production checkpoint. Begin with one fixture and three expected outcomes. Then add duplicate deliveries, a stopped finance worker, an invalid schema version, and an older event arriving late. Compare attributed outcomes, not merely successful handler responses. If a consumer invokes an AI model, record its token cost under that consumer outcome rather than charging it vaguely to ingress.

## Keep recovery from rewriting history

Verify the external webhook before constructing the envelope, and keep credentials out of source code. Monitor three separate facts: ingress acceptance, per-consumer lag, and committed outcomes. A green HTTP response proves only that the boundary answered.

Queue retention is not an audit ledger. Preserve consumer results for the period required by billing review, and retain the original event ID through dead-letter replay. If an operator must change a payload during recovery, emit a correction linked to the original event instead of silently editing history.

Short paths deserve short prose. Recovery does not.

Take the illustrative threshold event above. Suppose alerting succeeds immediately, controls succeeds after one retry, and finance remains paused for four hours. The audit view should show one accepted business event, one completed result for each of the first two consumers, and one pending finance result. When finance resumes, its worker may receive a duplicate before or after the original attempt; either way, the `(event_id, consumer)` record permits one committed effect. If the queue instead creates a fresh business ID for every delivery attempt, the ledger will falsely show several threshold events. That is the concrete failure this architecture is designed to prevent, and it is why transport request IDs remain secondary evidence rather than the accounting key.

Finally, pause one consumer while the other two run, redeliver the same event, and rebuild the chain from external event ID to all three outcomes. The operational rule is compact: one verified ingress, one immutable envelope, durable idempotency, visible lag, explicit replay, and an explainable ledger.

## Sources and References

- [BullMQ documentation](https://docs.bullmq.io/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [Google Cloud Pub/Sub documentation](https://cloud.google.com/pubsub/docs)
- [Azure Service Bus messaging documentation](https://learn.microsoft.com/azure/service-bus-messaging/)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
