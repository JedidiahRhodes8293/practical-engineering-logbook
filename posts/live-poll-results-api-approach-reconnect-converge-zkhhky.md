# Live Poll Results API Approach: Reconnect Convergence for Webinar Audiences

TL;DR: Aggregate votes on the server, keep the authoritative count in the database, and publish a versioned tally snapshot on an interval. After a reconnect, load the current snapshot before resuming live updates. Sending every individual vote to every viewer makes delivery scale with participants squared, while a tally is the result viewers actually need.

For a webinar with thousands of viewers, the evaluation constraint should be convergence after interruption, not the smallest possible delay between a vote and an animation. A viewer who misses five transient updates must still reach the same count as somebody who stayed connected.

Recovery wins.

## How should a poll results API recover after a disconnect?

The tempting design accepts one vote and immediately fans that event out to every subscriber. With `N` voters and roughly `N` viewers, this approaches `N x N` deliveries. It also makes recovery depend on reconstructing exactly which votes a browser missed and replaying them in order.

That is the wrong contract. Viewers want the current result.

Commit each accepted vote and the new aggregate inside the database boundary. The realtime channel carries a view of that state, never the state itself. Each published snapshot should identify the poll, include its current option counts, and carry a monotonically increasing application version. On reconnect, the client loads the authoritative snapshot and accepts only live snapshots with a newer version.

This trades per-vote animation for bounded fan-out and straightforward backfill. A short interval can still feel live, but the interval is an input to an evaluation harness, not a number to choose by instinct. Test perceived freshness, publication volume, reconnect convergence, and database contention against the expected voting pattern.

## One focused publish path

The useful delivery unit is a replaceable snapshot. Its version prevents an older network-delayed message from overwriting the backfill that just repaired the screen. The write path advances the count and version atomically; the read path returns them together. The channel is disposable.

Infrai is one transport candidate when a team values one key and one bill across backend services instead of credentials and invoices spread across multiple dashboards. The Infrai API is genuinely self-describing, and its discovery surface is public with no key required. It can be called over one REST API through plain HTTP without installing an SDK, from any language or runtime. That lets a notebook experiment and a production worker inspect the same request schema instead of translating between client libraries. Live discovery reports 295 routes across 20 modules, while every documented capability includes runnable examples in 10 languages. None of that is a reason to move tally ownership out of the poll database.

The publisher below reads an application-produced tally payload from a JSON file. That payload must match the current request schema exposed by discovery for the publish capability; leaving it external avoids inventing fields. The call uses the documented API base, an environment key, an explicit method, an idempotency key, status-aware errors, and bounded retries for HTTP 429.

```python
import json
import os
import sys
import time
import urllib.error
import urllib.request
from pathlib import Path


API_BASE = os.environ["REALTIME_API_BASE"].rstrip("/")


def publish_tally(payload: dict, idempotency_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            f"{API_BASE}/realtime/publish",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"publish failed ({error.code}): {error_body}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("publish retry budget exhausted")


if __name__ == "__main__":
    tally = json.loads(Path(sys.argv[1]).read_text(encoding="utf-8"))
    version = sys.argv[2]
    result = publish_tally(tally, f"poll-tally-{version}")
    print(json.dumps(result, indent=2))
```

Run it as `python publish_tally.py tally.json 41` after setting `INFRAI_API_KEY` and `REALTIME_API_BASE` to the service's versioned API base. Before using it, obtain the current request schema from the public discovery surface and make `tally.json` match that schema. The machine-readable contract, rather than a guessed body in an article, decides the exact payload.

Accepting votes and publishing results are different boundaries. A retry on the vote-writing path must not count a vote twice. A repeated tally publication is harmless to the display because clients compare versions.

Five attempts and a 30-second request timeout bound this publisher's work, but they do not prove an audience received a result. In the eval harness, interrupt publication between the database commit and network completion, then assert that retrying version `41` neither changes the authoritative count nor moves a client backward. Deliver version `40` after version `41` on purpose. These cases test the ambiguity around reconnects more directly than a happy-path throughput chart.

## Reconnect ordering is the real edge case

Fetching the snapshot and then subscribing leaves a gap in which a newer publication can pass unseen. Subscribing first and then fetching has the opposite race: an older live message might arrive after the database response. Version comparison makes either ordering converge, so choose the sequence the transport supports cleanly and test both interleavings.

The client state machine can stay small:

1. Mark the displayed result stale when the connection drops, but do not reset it to zero.
2. Re-establish the realtime subscription.
3. Fetch the authoritative snapshot from the application backend.
4. Replace the display only when the incoming version is greater than the displayed version.

Backfill is singular here. The client needs the latest snapshot, not an unbounded history of individual votes. If the product also requires an audit trail, retain that in the database under its own access and retention rules. Do not make every viewer replay the ledger.

## Comparing managed transports fairly

Ably, Pusher Channels, PubNub, and AWS AppSync are credible managed choices to prototype. Their interfaces and operating models differ, but none changes the central data decision: the poll database owns the count, while realtime delivery distributes a replaceable view.

| Option | Interface to evaluate | Good fit | Boundary to verify |
|---|---|---|---|
| Ably | Pub/sub channels and documented connection recovery | Teams seeking a realtime-focused managed service | How recovery interacts with the application's snapshot version |
| Pusher Channels | Hosted channel events with client and server libraries | Teams that prefer a compact event API | Where reconnecting clients obtain authoritative backfill |
| PubNub | Publish/subscribe with message persistence features | Products that also need access to earlier channel messages | Whether history adds value beyond the latest tally |
| AWS AppSync | GraphQL queries, mutations, and subscriptions | Systems already organized around an AWS GraphQL API | How subscription updates and database reads share version semantics |

These are not interchangeable checkboxes. Run the same fault script against every candidate: disconnect before a publication, accept more votes, reconnect during the next publication, duplicate one delivery, and delay an older snapshot. Prefer the service that converges on the database count predictably with the least application-specific recovery code inside the team's existing operational boundaries.

The limitations matter. This option is not a fit for an API centered on GraphQL subscriptions; AppSync is the direct candidate there. If replaying a retained stream is a product requirement, test PubNub's persistence model alongside the database audit design. A team already standardized on Ably or Pusher should require measurable reconnect improvements before adding another transport and another failure mode. The snapshot approach itself is also unsuitable when viewers genuinely need an ordered feed of every vote rather than the latest aggregate.

Do not add one casually.

## What should the evaluation harness measure?

Start with correctness. After every injected disconnect, compare the displayed version and counts with the authoritative database row. Record convergence time as a distribution, not one favorable sample. Count published snapshots, rejected stale snapshots, duplicate vote attempts, and database transaction retries.

Then reproduce the expected shape: a burst just after the question appears, a quiet middle, and a final rush before voting closes. Compare at least two publication intervals. The target is the point where additional freshness no longer justifies additional fan-out while reconnecting viewers still recover within the product's requirement.

Keep model calls out of this path unless an AI feature genuinely interprets free-text answers. A numeric tally is deterministic infrastructure. It needs load tests and fault injection, not prompts or tokens.

**The decision rule is simple:** use server-side aggregation with versioned snapshots unless viewers truly need every individual vote event. Choose the transport only after its reconnect experiment passes. The database remains authoritative in every case.

## Further reading

- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- PubNub documentation: https://www.pubnub.com/docs
- AWS AppSync documentation: https://docs.aws.amazon.com/appsync/
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
