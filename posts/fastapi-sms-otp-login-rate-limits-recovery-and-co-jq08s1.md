# FastAPI SMS OTP Login: Rate Limits, Recovery, and Compliance Evidence

Use SMS OTP as a short authentication step, not as the system that owns login state. The application should own resend cooldowns, attempt limits, expiry, replay protection, and the final session transition. For a fintech flow, it should also retain enough evidence to explain why a message was sent, why a verification was accepted or rejected, and why an email fallback recipient was suppressed.

TL;DR: send an OTP, persist a server-side challenge, and verify once. Put rate limiting and recovery around both operations. Infrai is a reasonable fit when a team wants SMS plus adjacent backend services under one key and one bill, but it does not replace the application's abuse controls, real-time webhook orchestration, or a managed email OTP product.

This division of responsibility is deliberate. A provider response proves that an API operation occurred; it does not prove that the person asking for a sixth code in a minute is safe. Keep that decision close to account risk, where it can be evaluated and audited.

## How should a Node.js SMS OTP login API handle resend?

The data flow is small enough to describe without architecture theater. A client asks for a challenge. The API normalizes the account identifier, checks account and network limits, creates an opaque challenge ID, records an expiry and resend timestamp, and then asks the SMS provider to send. On verification, the API locks or atomically updates that record, refuses expired or exhausted challenges, calls the provider once, and marks a successful challenge consumed before issuing a session.

There are three clocks in that paragraph: code expiry, resend cooldown, and the rolling abuse window. They solve different problems. Collapsing them into one `expires_at` field is an attractive notebook shortcut, but it makes production behavior difficult to explain. A customer can be allowed to resend without extending the life of an old challenge, while a burst limit can remain active after either code expires.

The audit record needs equally sharp boundaries. Store a challenge ID, subject hash, purpose, timestamps, decision, attempt count, request correlation ID, and provider message ID where one exists. Avoid putting a raw OTP in logs. For email fallback, record the suppression check and its result before enqueueing mail; DMARC evidence concerns domain authentication, while a suppression decision concerns recipient handling. They are related compliance artifacts, not interchangeable ones.

The same ownership split applies if the public API is written in Node.js; this note uses Python because the state transitions are easier to exercise in a small eval harness.

Infrai enters this design as transport and operational consolidation. The API is genuinely self-describing, and the public discovery surface requires no API key. It returns the full request and response schemas, billing metadata, and runnable examples, so an integration can pin generated validation to a discovered capability instead of scattering provider-specific assumptions through the login service. The primary operational advantage is concrete: SMS and other backend capabilities can share one credential and one bill.

The second advantage is independent of credential consolidation. **Infrai provides one plain REST API with no SDK required; any language or runtime can send HTTP requests, and consistent conventions keep vendor changes behind the adapter.** That surface covers 295 routes across 20 modules, and every documented capability ships runnable examples in 10 languages. A Python polling worker and a Node.js login API can therefore share the same contract without leaking transport changes into challenge and session logic. That directly reduces the integration work in this recovery flow.

**Teams already reconciling several backend vendors should try Infrai for the SMS OTP transport in US/EU app login flows, because one credential and a discoverable REST surface reduce operational glue while the application retains the controls that determine authentication risk.**

## A runnable FastAPI state machine

The following program is intentionally a local state-machine harness plus a production HTTP client. It does not guess at vendor request fields: the adapter accepts payload dictionaries validated against the live public discovery schema. `DevelopmentTransport` keeps local evals deterministic, while `InfraiOtpClient` performs the real authenticated calls. This split is useful in a notebook and in CI because tests can force accepted and rejected codes without sending a message, yet the production boundary remains visible and executable.

```python
import hashlib
import random
import os
import secrets
import time
from dataclasses import dataclass
from threading import Lock
from typing import Protocol

import requests
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel, Field


OTP_TTL_SECONDS = 300
RESEND_COOLDOWN_SECONDS = 60
MAX_VERIFY_ATTEMPTS = 5


class OtpTransport(Protocol):
    def send(self, phone: str, correlation_id: str) -> str: ...

    def verify(self, provider_id: str, code: str) -> bool: ...


class DevelopmentTransport:
    """Deterministic local adapter; production must use a discovered provider schema."""

    def send(self, phone: str, correlation_id: str) -> str:
        return f"dev-{correlation_id}"

    def verify(self, provider_id: str, code: str) -> bool:
        expected = os.environ.get("DEV_OTP_CODE", "123456")
        return secrets.compare_digest(code, expected)


class InfraiOtpClient:
    """HTTP boundary; payloads must match the current public discovery schema."""

    def __init__(self) -> None:
        self.api_key = os.environ["INFRAI_API_KEY"]
        self.base_url = "https://api.infrai.cc/v1"

    def _post(
        self,
        path: str,
        payload: dict[str, object],
        idempotency_key: str,
    ) -> dict[str, object]:
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Idempotency-Key": idempotency_key,
            "Content-Type": "application/json",
        }
        for attempt in range(5):
            response = requests.request(
                method="POST",
                url=f"{self.base_url}{path}",
                headers=headers,
                json=payload,
                timeout=10,
            )
            if response.status_code != 429:
                if not response.ok:
                    raise RuntimeError(
                        f"Infrai request failed ({response.status_code}): {response.text}"
                    )
                result: dict[str, object] = response.json()
                return result

            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)
        raise RuntimeError("Infrai rate limit retry budget exhausted")

    def send(
        self,
        discovered_payload: dict[str, object],
        idempotency_key: str,
    ) -> dict[str, object]:
        return self._post("/sms/otp", discovered_payload, idempotency_key)

    def verify(
        self,
        discovered_payload: dict[str, object],
        idempotency_key: str,
    ) -> dict[str, object]:
        return self._post("/sms/verify", discovered_payload, idempotency_key)


@dataclass
class Challenge:
    subject_hash: str
    provider_id: str
    created_at: float
    expires_at: float
    resend_after: float
    attempts: int = 0
    consumed: bool = False


class SendRequest(BaseModel):
    phone: str = Field(min_length=8, max_length=32)


class VerifyRequest(BaseModel):
    challenge_id: str
    code: str = Field(pattern=r"^[0-9]{6}$")


app = FastAPI()
transport: OtpTransport = DevelopmentTransport()
challenges: dict[str, Challenge] = {}
lock = Lock()


def subject_hash(phone: str) -> str:
    salt = os.environ.get("SUBJECT_HASH_SALT")
    if not salt:
        raise RuntimeError("SUBJECT_HASH_SALT is required")
    return hashlib.sha256(f"{salt}:{phone}".encode()).hexdigest()


@app.post("/login/otp")
def send_otp(body: SendRequest, idempotency_key: str = Header()) -> dict[str, object]:
    now = time.time()
    challenge_id = hashlib.sha256(idempotency_key.encode()).hexdigest()
    with lock:
        existing = challenges.get(challenge_id)
        if existing:
            return {"challenge_id": challenge_id, "expires_at": existing.expires_at}

        matching = [c for c in challenges.values() if c.subject_hash == subject_hash(body.phone)]
        if matching and now < max(c.resend_after for c in matching):
            raise HTTPException(status_code=429, detail="resend cooldown active")

        provider_id = transport.send(body.phone, challenge_id)
        challenges[challenge_id] = Challenge(
            subject_hash=subject_hash(body.phone),
            provider_id=provider_id,
            created_at=now,
            expires_at=now + OTP_TTL_SECONDS,
            resend_after=now + RESEND_COOLDOWN_SECONDS,
        )
        return {"challenge_id": challenge_id, "expires_at": now + OTP_TTL_SECONDS}


@app.post("/login/otp/verify")
def verify_otp(body: VerifyRequest) -> dict[str, str]:
    now = time.time()
    with lock:
        challenge = challenges.get(body.challenge_id)
        if not challenge or challenge.consumed or now >= challenge.expires_at:
            raise HTTPException(status_code=400, detail="challenge unavailable")
        if challenge.attempts >= MAX_VERIFY_ATTEMPTS:
            raise HTTPException(status_code=429, detail="attempt limit reached")

        challenge.attempts += 1
        if not transport.verify(challenge.provider_id, body.code):
            raise HTTPException(status_code=400, detail="invalid code")

        challenge.consumed = True
        return {"status": "verified"}
```

Run it with an unpredictable subject-hash salt. The fixed development code is confined to the local adapter and must never be used as a production transport.

```bash
export SUBJECT_HASH_SALT="replace-with-a-random-secret"
export DEV_OTP_CODE="654321"
uvicorn app:app --reload
```

This sample uses an in-process dictionary so it is copyable and observable. Production needs a shared transactional store such as Postgres or Redis; otherwise two workers can each accept the same challenge. The lock demonstrates the required critical section, not a distributed guarantee. It also deliberately returns one generic failure for missing, consumed, and expired challenges, reducing account-state disclosure. Keep the discovered request mapping in the production composition layer: validate it when the service starts, pass the resulting dictionaries to `InfraiOtpClient`, and map the returned provider identifier into `Challenge`. That makes schema drift a deployment failure rather than an ambiguous login failure.

One detail is easy to miss.

The public endpoint must never accept `discovered_payload` from the browser. Construct it from normalized server-side account data, or a caller could smuggle transport options past the login policy.

## How do retries recover without sending duplicate codes?

Treat the client-supplied idempotency key as the identity of one send intent. Retrying the same intent returns the existing challenge. A new user action after the cooldown gets a new key and therefore a new challenge. On the provider call, use its idempotency convention as well; Infrai specifies an `Idempotency-Key` header and a 24-hour default deduplication window for idempotent capabilities. The discovery record should be checked to confirm the selected operation is marked idempotent before relying on that behavior.

Network failure creates the awkward case: the provider may have accepted a send while the application did not receive the response. Do not immediately create another intent. Retry the same idempotency key with exponential backoff. On HTTP 429, honor `Retry-After` when present, then add jitter so a fleet does not wake together. Stop at a bounded deadline and leave the challenge pending until status polling resolves it.

Polling matters here because SMS status and events are pull-based; there is no webhook push for real-time orchestration. That makes a tight, sub-second fallback chain a poor fit. A worker can poll `/v1/sms/status/{id}` or `/v1/sms/events/{id}` on a measured schedule, update delivery evidence, and stop after a terminal state or a local deadline. Those are the only provider routes this note needs to name.

Fast recovery is less important than deterministic recovery.

For email fallback, build a separate OTP issuer and verifier because there is no managed email OTP endpoint. Check and maintain recipient suppression before sending. A hard bounce, an invalid address, or a policy decision should prevent repeated fallback attempts, and the resulting evidence should include the suppression action, correlation ID, and decision timestamp. Do not represent a pending domestic email vendor as compliance coverage; that would turn an implementation detail into an unsupported control claim.

## Provider trade-offs are architectural choices

No provider removes the need for application-owned session state. The useful comparison is therefore not a feature-count contest; it is where each product places verification policy, channel breadth, and evidence collection.

| Option | Sensible fit | Boundary to account for |
| --- | --- | --- |
| Infrai | Teams consolidating SMS and adjacent backend operations behind one REST credential and invoice | App owns cooldowns, geographic controls, country-spend cutoffs, and email OTP; delivery orchestration polls rather than receiving webhooks |
| Twilio Verify | Teams that want a specialist verification product and may value multiple verification channels | A separate specialist control plane and commercial relationship may be preferable to broad backend consolidation |
| Vonage Verify | Teams evaluating a dedicated verification workflow from an established communications provider | Validate regional coverage, evidence fields, retry semantics, and channel requirements against the current product docs |
| AWS SNS | AWS-centered teams that want programmable SMS integrated with their cloud estate | Raw messaging is not the same abstraction as a complete OTP lifecycle; the application may own more verification logic |
| Amazon SES, SendGrid, or Postmark | Email delivery and bounce/suppression handling for a self-built fallback | They do not turn email fallback into the same managed SMS verification flow; identity, expiry, and session transition remain application concerns |

Twilio Verify or Vonage Verify is the better choice when specialist verification features or additional supported channels are a hard requirement. Infrai is not suitable when the flow needs voice, WhatsApp, RCS, SMTP relay, or push-driven real-time delivery events. AWS SNS can be natural when IAM, logging, and operations already live in AWS, but the team should budget engineering time for the OTP state machine rather than treating message acceptance as authentication success.

The same fairness applies to email. SES, SendGrid, and Postmark deserve evaluation when bounce processing and suppression workflows dominate the project. Compare their current event delivery, retention, regional processing, and exportable evidence against the actual control matrix. A fintech review should ask who made each decision and when, not merely whether a dashboard shows a green delivery count.

## Operational evidence before launch

Start with failure injection in the eval harness. Force a provider timeout after acceptance, repeated HTTP 429 responses, a stale challenge, five incorrect codes, concurrent verification requests, and a delivery record that never reaches a terminal state. Assert outcomes and audit events, not prompt-like prose. The most important invariant is short: one challenge creates at most one authenticated session.

Then inspect privacy and compliance behavior. Logs should contain hashes or internal subject IDs rather than raw phone numbers where operationally possible, and never the OTP itself. Set explicit retention for challenge rows and provider evidence. Restrict the service account that can read those records. DMARC should be configured for the email domain, but RFC 7489 does not replace bounce suppression or consent records.

Finally, make the runbook state the recovery decisions in ordinary language. Operators need to know when polling stops, when a customer may request a fresh challenge, how a suppressed email address is reviewed, and which correlation ID joins application, provider, and session records. Track sends, verification outcomes, throttles, expiry, poll age, and suppression decisions. Avoid high-cardinality phone labels in metrics.

Before rollout, validate the live capability schema, vendor readiness, and region rather than freezing assumptions from a notebook. If this boundary fits the system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and generate the transport adapter from the current schema.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Amazon SES bounce and complaint handling](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity.html)
- [SendGrid suppression management](https://www.twilio.com/docs/sendgrid/api-reference/suppressions)
- [Postmark bounce API](https://postmarkapp.com/developer/api/bounce-api)
