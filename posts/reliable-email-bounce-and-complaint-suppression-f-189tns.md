# Reliable Email Bounce and Complaint Suppression for a Transactional Marketplace App

## Short answer

Short answer: for a transactional marketplace app notifying a seller about a new order, the least complex reliable design is an event-backed outbox plus a small suppression-list polling job. Record the notification before sending it, stop sending to addresses marked by a hard bounce or complaint, and make retries idempotent. A Node.js app can use the same boundaries even if the worker is written in another language; the reliability properties matter more than the runtime.

The notification path should be boring: an order transaction creates an outbox row, a worker claims that row, the email provider accepts the message, and a separate poller imports bounce and complaint events into a local suppression table. The send worker checks that table immediately before delivery. That last check is important because a seller can complain after the order is created but before a retry runs.

Ship the guard first.

## Start with the delivery decision, not the provider

Treat bounce and complaint data as delivery policy, not as a log that someone reads later. A hard bounce usually means the address should not be retried without an explicit address change or a carefully reviewed correction. A complaint is a stronger signal: the recipient marked the message as unwanted, so the application should suppress future non-essential mail to that address. A transient failure belongs in a retry schedule with a limit and a visible terminal state.

Keep the source event and the decision separate. Store the provider event identifier, event type, recipient, received time, and raw payload or a redacted digest. Then derive a compact table keyed by a normalized recipient address and message purpose. The purpose matters: a seller's order alert is operational mail, while a promotional newsletter is a different consent and suppression decision. Do not let an unsubscribe from one purpose silently redefine every other purpose unless your policy says so.

Keep it boring.

Here is a small, provider-neutral core that a Python worker can share with a Node.js service through the database. It does not pretend to be the provider integration. It makes the decision boundary explicit and is easy to exercise in an eval harness.

```python
from dataclasses import dataclass
from datetime import datetime, timezone


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    address: str
    kind: str
    occurred_at: datetime


def normalize(address: str) -> str:
    return address.strip().casefold()


def apply_event(event: DeliveryEvent, suppression: dict[str, str]) -> None:
    address = normalize(event.address)
    if event.kind in {"hard_bounce", "complaint"}:
        suppression[address] = event.kind


def may_send(address: str, purpose: str, suppression: dict[str, str]) -> bool:
    if purpose != "seller_order_alert":
        raise ValueError("unknown message purpose")
    return normalize(address) not in suppression


received = DeliveryEvent(
    event_id="evt_123",
    address="seller@example.com",
    kind="hard_bounce",
    occurred_at=datetime.now(timezone.utc),
)
suppression: dict[str, str] = {}
apply_event(received, suppression)
assert may_send("SELLER@example.com", "seller_order_alert", suppression) is False
```

In production, `apply_event` must be protected by a unique constraint on `event_id` and an upsert. Polling is allowed to see the same event twice, and the result should be the same either way. That is the sort of invariant worth testing before wiring in an SDK or a queue.

## How can a Node.js app make polling safe for a transactional order alert?

Start with the order database. In the same transaction that marks an order as placed, insert an outbox record containing an immutable order ID, seller ID, recipient address, message purpose, and a deterministic notification key such as `order_id + ":seller_order_alert"`. A worker claims pending records with a lease, renders the message from the order snapshot, and records the provider's acceptance identifier. It must never create a second business notification merely because the worker process restarted.

The poller has a different job. It asks the provider for new bounce and complaint events using a cursor or a time window, stores each event idempotently, advances its checkpoint only after the batch is committed, and updates the local suppression table. If the poller dies after the database commit but before the checkpoint update, it reads the batch again. That is fine. Duplicate input is expected; duplicate email is not.

The following sketch shows the control flow without inventing a provider route or response shape. The adapter is the only boundary that needs provider-specific code.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class ProviderEvent:
    event_id: str
    address: str
    kind: str


class EventSource(Protocol):
    def read(self, cursor: str | None) -> tuple[list[ProviderEvent], str | None]: ...


class Store(Protocol):
    def save_event_once(self, event: ProviderEvent) -> None: ...
    def suppress(self, address: str, reason: str) -> None: ...
    def commit(self) -> None: ...
    def load_cursor(self) -> str | None: ...
    def save_cursor(self, cursor: str) -> None: ...


def poll_once(source: EventSource, store: Store) -> None:
    events, next_cursor = source.read(store.load_cursor())
    for event in events:
        store.save_event_once(event)
        if event.kind in {"hard_bounce", "complaint"}:
            store.suppress(event.address, event.kind)
    store.commit()
    if next_cursor is not None:
        store.save_cursor(next_cursor)
```

There is a subtle transaction choice here. If the cursor is saved in a separate transaction, the poller may repeat a committed batch, but it will not skip events. If the cursor and event writes share one transaction, they advance atomically, but the adapter and store need a stronger boundary. Either design can work. The non-negotiable part is that advancing the cursor cannot happen before the corresponding events are durable.

The send worker should perform the suppression check after claiming the outbox row and before calling the provider. If the address is suppressed, mark the row as `suppressed` with a reason and do not retry it. If the provider reports a transient failure, release it with a next-attempt time. If the provider accepts the message, mark it sent with the acceptance ID. A timeout is ambiguous: query or reconcile using the idempotency key if the provider supports one, otherwise retain the outbox record for a deliberate reconciliation policy instead of blindly sending again.

## The failure map behind a useful suppression list

A suppression list does not fix an application that sends the wrong message to the wrong recipient. Before tuning retry intervals, check the data path: was the seller address verified, did the order snapshot contain the right seller, and can a late retry still be authorized? Logging only the final provider response will miss these failures.

The failure sequence is worth spelling out because it is easy to test the wrong thing. At 09:00, the marketplace commits an order and an outbox row. At 09:01, a worker claims the row, but the provider response times out after the request may already have been accepted. At 09:02, the worker restarts and sees an unresolved row. At 09:03, the poller imports a hard bounce for the seller address. A naive retry at 09:04 sends a second message; a disciplined worker first reconciles the ambiguous attempt, then checks the now-durable suppression decision, and records why it did or did not send. The database states, not a green process dashboard, are what let an operator reconstruct this chain.

That reconstruction is also the fastest way to find a bad assumption: compare the outbox transition, the provider acceptance identifier, the poll checkpoint, and the suppression row in one timeline, then turn any surprising transition into a regression test.

For each notification, capture a correlation ID, order ID, seller ID, purpose, attempt number, outbox state, provider acceptance ID, and suppression decision. Avoid logging message bodies and personal data by default. Metrics should distinguish queued, accepted, transient failure, hard bounce, complaint, suppressed, and permanently failed. An overall delivery percentage hides the exact state transition that needs attention.

There is a human boundary too. A complaint suppression is a durable signal, while a hard bounce may become actionable after the seller changes the address. Give support or account settings a controlled way to replace the address and record who made the change. Do not provide an operator button that merely clears suppression without changing the evidence behind it.

Authentication and account recovery mail deserve their own policy. NIST's digital identity guidance treats authenticator-related communications as part of a security-sensitive system, so a product should not assume that a generic marketing suppression rule is enough for every security notification. Define which messages are essential, which can be delayed, and which require a second channel or an in-app fallback.

## What should the deliverability test harness prove?

Test the state machine, not just the happy-path send. The minimum useful cases are a repeated event ID, a hard bounce followed by a retry, a complaint arriving between two attempts, an empty poll batch, a cursor replay, an adapter timeout, and two workers claiming the same outbox row. Each test should assert both the stored state and the number of provider send calls.

A compact table makes the policy reviewable:

| Signal | Local action | Retry policy | Human action |
| --- | --- | --- | --- |
| Accepted | Store acceptance ID | No immediate retry | Inspect only on reconciliation |
| Transient failure | Keep pending with backoff | Bounded retry | Review after terminal failure |
| Hard bounce | Suppress address for the purpose | Do not retry | Request an address correction |
| Complaint | Suppress address for the purpose | Do not retry | Require an intentional re-consent or channel change |
| Unknown timeout | Keep an ambiguous state | Reconcile before retry | Review if reconciliation is unavailable |

Use a test provider or a fake adapter in CI, then run a small staging exercise with addresses reserved for bounce and complaint simulation. Your mileage may vary across providers because event timing and classifications differ; I’m not sure any single provider's label can replace an application-owned policy. Measure the lag from event occurrence to local suppression, plus the count of sends attempted after a suppression event.

The eval harness should include a property that survives implementation changes: after a hard bounce or complaint is durably imported, no later worker attempt may call `send` for that address and purpose. This catches a common notebook-to-prod gap, where a demo handles a webhook but the retry worker has its own unguarded path.

## Where does polling stop being the right boundary?

For one transactional app, an outbox table, a scheduled poller, an idempotent event store, and a suppression check are usually enough. Add a queue when claim latency or workload isolation requires it. Add a dedicated notification service when multiple products need the same policy and ownership is explicit. A more elaborate platform is not automatically more reliable; each boundary adds checkpoint, replay, and observability work.

The catch is that polling is a poor fit when the provider's event retention is shorter than your polling outage window or when near-real-time suppression is a hard safety requirement. Use a durable event push path with replay support in that case, and keep periodic reconciliation as a backstop. Stick with a simpler scheduled poller when minutes of lag are acceptable, the provider exposes a durable cursor or retention window, and the team can monitor checkpoint age.

Before shipping, walk through the whole path with a clock: place the order, commit the outbox row, claim it, import a bounce or complaint, attempt a retry, and inspect the final state. Check the same sequence after a process restart and after a duplicated poll batch. If the answers are visible in the database and metrics, the implementation is ready for a real provider adapter.

## Further reading

- Amazon SES official documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- NIST SP 800-63B Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://pages.nist.gov/800-63-3/sp800-63b.html
