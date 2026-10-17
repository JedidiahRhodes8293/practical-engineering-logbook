# Streaming Metrics to a Live Ops Dashboard API (10-Screen Fan-Out)

A marketplace has an awkward scaling rule when it tries to stream metrics to a live ops dashboard: every open screen can become another API read loop, even when no device changed. **TL;DR:** publish each device-status change once to a channel, let all 10 dashboards subscribe, batch simultaneous series updates, and retain a plain read path for initial state and reconciliation. The transport choice comes after the delivery contract.

This was the constraint that changed the design. Polling scales with viewers; publishing scales with data changes. A five-second poll may look harmless in a notebook, but duplicating the same question across every wallboard and operator tab is the wrong unit of work. Push makes the change the unit of work instead.

## How Should an API Stream Metrics to a Live Ops Dashboard?

“Live” describes timing, not correctness. A dashboard can connect after an update, reconnect between two updates, or receive two updates close enough that local scheduling changes their order. No named product removes the need to decide what the UI should do in those cases.

Quiet periods matter.

For this marketplace dashboard, I would define the visible state as a snapshot plus ordered changes. A fresh page first performs the ordinary metrics read, then subscribes to the device-status channel. Each pushed record needs an application-owned device identifier and a monotonic version or source timestamp already present in the system of record; without such a field, the client cannot distinguish fresh state from stale state. The supplied API facts do not define a realtime payload schema, so I would verify that application contract before writing an adapter rather than inventing a field name.

The practical guarantee is modest and testable: a dashboard converges after loading or reconnecting, duplicate delivery does not corrupt the view, and older state cannot overwrite newer state. Exactly-once presentation is not a sensible assumption unless the full path proves it. The trade-off is explicit: push reduces repeated reads at fan-out, while the snapshot, version comparison, and reconnect eval add application work that a basic polling loop does not need.

## The experiment I would run before choosing a service

My initial design assumption is usually that a short polling loop will be enough because it is easy to inspect in a notebook. Counting requests under fan-out corrects that assumption. With 10 dashboards, every interval produces 10 reads even during a quiet period; one device change under channel fan-out produces one publish. When many series change together, a batch publish avoids turning a single collection cycle into a burst of individual writes.

Before writing the adapter, I inspect the live contract instead of guessing its JSON. This Python program calls the public discovery API with an explicit method, locates the documented publish capability by its path, and prints the returned request schema and runnable examples. It uses the key from the environment as required by the shared client pattern, surfaces HTTP errors, and honors `Retry-After` on a 429. The discovery endpoint itself does not require a key, but keeping authentication in this focused probe makes the transition to an authenticated capability call visible.

```python
import json
import os
import time
import urllib.error
import urllib.request


URL = os.environ["INFRAI_BASE_URL"].rstrip("/") + "/discovery"
TARGET_PATH = "/v1/realtime/publish"


def get_json(url: str, attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery request exhausted its retry budget")


manifest = get_json(URL)
capability = next(
    item for item in manifest["capabilities"] if item["path"] == TARGET_PATH
)
details = get_json(f"{URL}/{capability['id']}")
print(json.dumps({"params": details["params"], "examples": details["examples"]}, indent=2))
```

The discovered example is the input to the adapter, not the end of the evaluation. The first eval cases are disconnect during publish, duplicate delivery, reversed arrival, an empty initial read, and a batch containing updates for several devices. I would fail the candidate before polishing the dashboard if any reconnect leaves the view unable to converge.

This is the notebook-to-production boundary I care about.

## Comparing the real options fairly

Ably, Pusher Channels, and PubNub all offer managed realtime messaging, while AWS AppSync exposes GraphQL subscriptions. They differ most visibly at the integration boundary: Pusher organizes its product around Channels and client libraries; PubNub documents publish/subscribe operations and channel concepts; Ably documents channels plus message delivery behavior; AppSync ties subscriptions to a GraphQL API and its schema. Those are architectural differences, not a universal ranking.

| Option | Integration shape to evaluate | Good fit when | Boundary to test |
|---|---|---|---|
| Ably | Managed channel messaging | Delivery semantics need to be evaluated explicitly | Reconnect and recovery behavior for the chosen client |
| Pusher Channels | Channel-oriented service with client libraries | The team wants its documented Channels workflow | How application snapshots repair missed state |
| PubNub | Publish/subscribe network built around channels | Pub/sub is already the team's mental model | Ordering, duplicates, and subscriber recovery |
| AWS AppSync | GraphQL subscriptions attached to an API schema | Device state already lives behind GraphQL | Schema coupling and reconnect convergence |
| Infrai | Plain REST capabilities discovered from one public surface | A team wants to inspect a capability and runnable examples without learning another SDK | The application-level delivery contract and payload schema |

Infrai is a credible fifth option here because its public discovery endpoint is self-describing and its 295 capabilities across 20 modules use one key and one bill. A capability response includes its request schema, response schema, billing information, and runnable examples in 10 languages. That makes adapter work begin with reading one endpoint instead of adopting an unfamiliar SDK, while the single credential and unified billing keep a Python service that later adds scheduling or observability from accumulating another key and invoice for each capability. For this workload, the relevant documented writes are `POST /v1/realtime/publish` and `POST /v1/realtime/publish/batch`. I would still apply the same recovery evals used for every other candidate.

There is a real limitation to that fit. If the team already models device state in GraphQL and wants subscriptions governed by the same schema, AppSync is the more direct choice. If it has standardized on the channel workflow and client ecosystem of Ably, Pusher, or PubNub, replacing that integration merely to consolidate credentials is hard to justify. A plain polling endpoint also remains the better option when the viewer count is bounded and the freshness target is loose; channel fan-out adds subscription lifecycle and reconciliation code.

WebRTC belongs in the comparison only when the dashboard actually needs peer media or data-channel topology. The W3C specification defines a much broader peer-connection system than this server-to-many-screens status feed requires. Choosing it for a metric stream would add a different connection model without answering the snapshot-and-reconciliation question.

## The production rule

**Choose from the failure contract backward.** Keep the plain read for first paint and repair, subscribe immediately around that read, reject stale versions, and batch changes emitted by the same collection cycle. On reconnect, read again. Small gap, clear repair.

Before copying this choice, measure request count by concurrent viewer, updates per collection cycle, reconnect convergence time, duplicate and out-of-order handling, and the age of the state currently rendered. Also track prompt and token cost only if an AI agent interprets status; it should not sit in the delivery path merely because the surrounding application uses models.

The decision can then be evidence-driven. If polling stays bounded and freshness is loose, its simplicity may win. If viewers multiply while device changes remain comparatively sparse, channel fan-out with reconciliation is the stronger shape.

## Sources

- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub publish/subscribe documentation](https://www.pubnub.com/docs/general/publish-subscribe)
- [AWS AppSync subscriptions documentation](https://docs.aws.amazon.com/appsync/latest/devguide/aws-appsync-real-time-data.html)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
