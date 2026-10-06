# ADR-0002: Cost Model Weights

## Status

Accepted (pending calibration)

## Context

`cost_model.py` scores every candidate action using three per-action weight tables:

- `COMM_COST` — flat communication/interruption cost of taking the action.
- `CATCH_VALUE` — relative value of catching a real incident with this action.
- `DISRUPTION_COST` — relative cost of acting on a false positive with this action.

These tables are consulted directly by `task_benefit()`, `communication_cost()`,
and `risk()` to compute each action's EV (see ADR-0001).

## Decision

The current table values are hand-picked, reasoned estimates on a 0.0-1.0 relative
scale — not derived from data. They encode an ordering and rough spacing between
actions based on how disruptive or valuable each action is believed to be, not an
empirically fit cost.

Relative-scale reasoning used to set the values:

- **`COMM_COST`**: ordered by how many people/systems an action interrupts and how
  intrusive that interruption is. `SUPPRESS` is 0 by definition (no communication
  happens). `BROADCAST` (0.60) is the highest because it fans out to the widest
  audience. `REQUEST_APPROVAL` (0.35) and `ESCALATE` (0.30) are next — both pull in
  a specific person's attention. `SEND` (0.10) and `VERIFY` (0.05) are cheap,
  narrowly-scoped actions.

- **`CATCH_VALUE`**: ordered by how effectively the action actually resolves a real
  incident. `SUPPRESS` is 0 (does nothing even if the incident is real). `VERIFY`
  (0.15) only confirms, it doesn't resolve. `SEND` (0.35) notifies one party.
  `ESCALATE` (0.80), `REQUEST_APPROVAL` (0.85), and `BROADCAST` (0.90) are high
  because they put the incident in front of people who can act on it at scale.
  `PROPOSE_REMEDIATION` (0.75) is slightly below those because proposing a fix
  isn't the same as it being applied.

- **`DISRUPTION_COST`**: ordered by how much damage the action causes *if the
  incident turns out to be a false positive*. `SUPPRESS` (0.0) and `VERIFY` (0.05)
  cause essentially no disruption since nothing externally visible happens.
  `BROADCAST` (0.70) is the highest: a false broadcast interrupts the most people
  for nothing. `PROPOSE_REMEDIATION` (0.50) is high because an unnecessary
  remediation can itself destabilize a healthy system. `REQUEST_APPROVAL` (0.20)
  is lower than `PROPOSE_REMEDIATION` despite being "higher impact" conceptually,
  because requesting approval for nothing just costs one person's attention and
  is never auto-applied.

Example of the relative-scale logic in practice: `BROADCAST` costs more than
`ESCALATE` in both `COMM_COST` (0.60 vs. 0.30) and `DISRUPTION_COST` (0.70 vs. 0.30)
because broadcasting interrupts an entire audience, while escalation reaches one
responder.

## Rationale

Hand-picked weights make the initial ordering and spacing between actions legible
and debuggable — each number is traceable to a stated reason, not a fit coefficient
no one can explain. This matters while the hand-labeled evaluation set (`eval/`) is
still being built: there's no labeled ground truth yet to calibrate against, so a
principled starting point is more useful than a prematurely "learned" one.

## Consequences

- These weights are a baseline, not a final answer. Once `eval/` has enough
  hand-labeled scenarios, the tables should be recalibrated (or replaced by a
  learned component) against observed decision quality, and this ADR updated or
  superseded.
- Because all three tables share the same 0.0-1.0 relative scale, adding a new
  candidate action requires picking consistent values across all three tables
  (not just one) to avoid a skewed EV comparison.
- Until calibration happens, treat absolute EV magnitudes as not meaningful —
  only the relative ranking of actions for a given message should be trusted.
