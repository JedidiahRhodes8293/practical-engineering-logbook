# Tenant Cutovers With a Separate DNS Zone — Non-Production Subdomain Overhead

Use a delegated staging zone when automation must create a subdomain for every healthtech tenant; keep records in the existing zone only when the same tightly controlled writer already owns both production and staging. The deciding constraint is propagation delay versus cutover speed, but write authority is the guardrail: delegation gives the staging writer a smaller namespace at the cost of another boundary to operate.

TL;DR: treat this as a failure-containment decision, then test the cutover path. A subdomain is a name. A separately delegated zone is an authority boundary. For a tenant such as `acme.staging.health.example`, either layout can produce the desired hostname, but only the delegated layout lets the staging automation operate without receiving write access to the parent zone.

## Should non-production use a separate DNS zone or a subdomain?

A plain subdomain is enough when the DNS writer can be trusted with the parent zone and the team accepts that shared operational scope. This is the low-friction path: one ownership surface, one record inventory, and no delegation step before tenant records can be created. For a small staging system, that can be the right answer.

Authority first.

The weakness appears when the job changes from occasional manual edits to automatic creation for every tenant. The application now has standing write capability. If its credentials or record-selection logic are too broad, the possible mistake is broader too. A tenant-provisioning worker intended to update `acme.staging.health.example` should not need authority over unrelated production names merely because both live beneath the same parent.

Delegating `staging.health.example` changes that operating model. The parent remains responsible for the delegation, while the staging writer manages names below the delegated boundary. This does not make bad changes impossible. It makes the intended scope legible and limits which namespace the automation is designed to alter.

The limitation is equally concrete: delegation can delay a cutover when the parent-side change and the child-side records aren't coordinated. It isn't suitable for a staging namespace whose team cannot own that lifecycle and monitor both sides. In that case, use a subdomain in the existing zone instead, narrow the writer through the available authorization controls, and require review for changes near production names. This is a trade-off, not a maturity ladder; a separate zone buys a clearer authority boundary by accepting another failure surface.

That distinction matters in healthtech because staging often resembles production closely enough to be useful while still having a very different change rate. Tenant environments may be created, evaluated, replaced, or retired as application builds move through a release pipeline. The high-churn writer and the low-churn parent do not have to share authority merely because their names share a suffix.

## The cutover experiment

The tempting design is to create each tenant record in the parent zone and declare success when a lookup returns the expected destination. That tests the happy path, not the boundary. It says nothing about an overbroad delete, an attempted write outside staging, or how long a replacement remains inconsistent across the resolvers represented in the evaluation set.

That test is too small.

I would turn the choice into an eval before turning it into policy. Use the same candidate tenant names and the same observation schedule for both layouts. Record when each observer returns the expected answer, whether any observer returns an old answer after the change, and whether the writer can mutate a production control name. The final check must fail by construction in the delegated design.

Here is a focused Python harness around generic DNS and writer interfaces. It deliberately avoids assuming a provider API. The adapter behind `writer` can target a local test authority or the DNS system used by the deployment pipeline.

```python
from dataclasses import dataclass
from time import monotonic, sleep
from typing import Protocol


class DnsWriter(Protocol):
    def replace(self, name: str, value: str) -> None: ...


class DnsObserver(Protocol):
    def lookup(self, name: str) -> str | None: ...


@dataclass(frozen=True)
class Observation:
    observer: str
    seconds: float
    value: str | None


def observe_cutover(
    writer: DnsWriter,
    observers: dict[str, DnsObserver],
    name: str,
    expected: str,
    interval_seconds: float = 1.0,
    deadline_seconds: float = 60.0,
) -> list[Observation]:
    started = monotonic()
    writer.replace(name, expected)
    pending = dict(observers)
    results: list[Observation] = []

    while pending and monotonic() - started < deadline_seconds:
        for label, observer in list(pending.items()):
            value = observer.lookup(name)
            if value == expected:
                results.append(Observation(label, monotonic() - started, value))
                del pending[label]
        if pending:
            sleep(interval_seconds)

    elapsed = monotonic() - started
    results.extend(
        Observation(label, elapsed, observer.lookup(name))
        for label, observer in pending.items()
    )
    return results
```

The `60.0` seconds here is an experiment deadline, not a claim about DNS propagation. It keeps a test run bounded. Set it from the release objective, and make failure explicit when even one required observer misses that objective. Faster polling can increase query volume without making the underlying change arrive sooner, so the observation interval belongs in the cost review alongside pipeline runtime.

One lookup location is not an eval set. Include the resolver paths that represent the systems making real staging requests, then preserve the raw observations so a regression can be compared with the previous build. This is the notebook-to-prod move that matters: the first script proves the interface, while the checked-in harness defines the release criterion.

Don't hide the tail.

## What overhead does delegation add?

The overhead is operational rather than cosmetic. A delegated zone needs an owner, a controlled parent-side delegation change, separate writer credentials, lifecycle handling, and monitoring that distinguishes delegation failure from a missing tenant record. Those are real duties. They are also the mechanism that creates the narrower boundary.

A shared zone removes that boundary work, but it transfers the burden into authorization and review. The team must prove that the staging writer cannot touch production records, or consciously accept that it can. Record-name filtering inside application code is useful defense, yet it is weaker as an ownership signal than giving the writer authority over only the namespace it should manage.

Use the following decision rule:

- Choose delegation when staging automation has a different owner, credential lifecycle, or change rate from the parent, and when containing an erroneous write matters more than avoiding another DNS ownership surface.
- Keep the subdomain in the parent zone when one team and one controlled writer already operate both scopes, tenant churn is low, and the added delegation lifecycle would exceed the boundary benefit.
- Revisit the choice when tenant provisioning becomes automatic. A design that was reasonable for five manually managed environments may be wrong once every test run can create and replace names.

Five is an example review threshold, not a protocol limit. The durable signal is automation frequency and authority breadth.

Small systems count too.

## Email policy can cross the boundary

DNS boundaries are not only about application routing. If a staging tenant can send mail, domain-level policy deserves a separate review before names are delegated or inherited. DMARC identifies an Organizational Domain and describes policy discovery that can move from a subdomain toward that organizational boundary. It also defines an `sp` tag for policy applying to subdomains. Those rules mean the visible hostname boundary and the email-policy boundary should not be assumed to match.

Keep the conclusion narrow: delegation alone does not specify the desired mail behavior. Decide whether staging should send mail at all, publish the intended policy through the responsible domain owner, and test policy discovery independently from application cutover. RFC 7489 is the source for the DMARC behavior; the DNS layout decision should not silently stand in for it.

This is also why copying a production-shaped staging domain without an eval is risky. The application may resolve correctly while another domain-based control behaves according to policy inherited from elsewhere. A release check should cover every control the staging name is expected to exercise, rather than treating a successful address lookup as proof that the namespace is ready.

## Measure before adopting the boundary

Before copying the delegated design, measure four things: the slowest required observer during replacement, the interval in which old and new answers coexist, the result of an attempted out-of-scope write, and the recovery time for a broken delegation. Run the same corpus against the shared-zone candidate. Keep tenant identifiers synthetic and stable so results compare cleanly across builds without placing health data in logs or fixtures.

Then choose against the actual release objective. If the shared zone cuts a provisioning step but grants a high-churn worker broad authority, the speed win may not justify the scope. If delegation adds a parent-side operation to every tenant rather than once for the staging namespace, the implementation has put the boundary at the wrong level. Delegate the stable staging suffix, then create tenant names beneath it.

The practical answer is crisp: **use delegation to separate writers, not to decorate names**. Validate propagation as a distribution observed by the systems that matter, not as a single promised delay. The winning layout is the one that meets the cutover objective while making an accidental staging write stop at the boundary the team intended.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
