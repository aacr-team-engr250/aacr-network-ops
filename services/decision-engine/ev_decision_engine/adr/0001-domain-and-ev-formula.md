# ADR-0001: Domain and EV Formula

## Status

Accepted

## Context

The decision engine coordinates incident response on a simulated multi-region
Kubernetes platform. It ingests `Event` + `TrustMetadata` messages describing
candidate incidents and must choose one action per incident from a fixed set:
`SUPPRESS`, `VERIFY`, `SEND`, `ESCALATE`, `BROADCAST`, `PROPOSE_REMEDIATION`,
`REQUEST_APPROVAL`.

The system handles 8 network incident types:

- `network_partition`
- `gradient_sync_latency`
- `service_discovery_failure`
- `packet_loss`
- `bandwidth_saturation`
- `load_balancer_failure`
- `connection_timeout`
- `cross_region_latency_spike`

Signals arrive with imperfect trust: a source's confidence, historical false-positive
rate, corroboration count, and the asset criticality of what it's reporting on all
affect how much the signal should be believed.

## Decision

Each candidate action is scored by an expected-value (EV) formula:

```
EV(action) = TaskBenefit(action) - CommunicationCost(action) - Risk(action)
```

- **TaskBenefit** — expected value of catching a real incident with this action,
  scaled by `P(real incident)` and event severity.
- **CommunicationCost** — flat interruption cost of taking the action, independent
  of whether the incident turns out to be real.
- **Risk** — expected cost of getting the action wrong, in either direction:
  - Cost of acting on a false positive (disruption for nothing).
  - Cost of staying silent on a real incident (see below).

The engine picks the highest-EV action, falling back to `SUPPRESS` if the best EV
is still below a floor threshold (nothing is clearly worth doing).

The key design decision: **suppression is not free**. `SUPPRESS` and `VERIFY` carry
a "silence risk" term in `Risk()` — the expected cost of a real incident going
undetected because the system chose not to act. This term scales with severity,
asset criticality, and blast radius, so suppressing a high-severity, high-criticality,
widely-corroborated incident is expensive even though suppression itself triggers no
communication.

High-impact remediation is also routed deliberately: if `PROPOSE_REMEDIATION`'s risk
crosses a threshold, it's excluded as a candidate and `REQUEST_APPROVAL` is used
instead, regardless of confidence — the system defers to a human before taking
high-blast-radius automated action.

## Rationale

A naive policy that treats suppression as a zero-cost default will systematically
under-react to real incidents, since every other action has a nonzero communication
cost and suppression doesn't. Giving suppression its own risk term — rather than
only penalizing false positives — forces the EV comparison to weigh "did nothing and
was wrong" against "acted and was wrong" on the same footing.

Routing high-impact remediation to approval rather than letting it win outright on
EV reflects that confidence in a model's EV estimate doesn't substitute for human
sign-off when the blast radius of being wrong is large.

## Consequences

- `cost_model.py` and `policy.py` must jointly maintain the invariant that only
  `SUPPRESS` and `VERIFY` carry silence risk — other actions are assumed to have
  resolved the incident, correctly or not.
- The EV floor and high-impact risk threshold are tunable constants (see ADR-0002)
  and need recalibration whenever the incident-type mix or trust signal
  distributions shift materially (e.g. as new incident types are added).
- Because EV is computed per action independently, adding a new candidate action
  only requires defining its weights in `cost_model.py`'s tables — no changes to
  the EV formula itself.
