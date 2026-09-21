# Autonomous-Network-Operations-Incident-Coordination

# Autonomous Network Operations Incident Coordination

## What this project does

We're building a system that watches for problems in ML training jobs running
on a simulated Kubernetes platform (things like memory crashes, training
errors, stalled jobs) and decides how to handle each one automatically —
instead of alerting a human every single time.

For every incident, the system picks one of these actions:

- **Suppress** — ignore it, not worth anyone's time
- **Verify** — double-check with another signal before acting
- **Send** — pass it along normally
- **Escalate** — alert a human
- **Broadcast** — warn everyone, this is a bigger problem
- **Propose remediation** — suggest a specific fix
- **Request approval** — ask a human to approve a fix before it runs

## How it decides

Every possible action gets a score using this formula:

```
Score = Benefit - Cost - Risk
```

- **Benefit** — how much good this action does, if the problem is real
- **Cost** — how annoying/expensive it is to do this action at all (e.g.
  interrupting a human costs more than just logging something quietly)
- **Risk** — how bad it is if we're wrong, in either direction: acting on a
  false alarm, OR staying silent on something that turns out to be real

The system computes this score for all 7 actions and picks the best one.
Staying silent isn't automatically "safe" in this formula — if the system
ignores something that turns out to be a real problem, that counts as a cost
too.

## Folder structure (this module)

```
schemas.py       - defines what an "event", "message", and "decision" look like
cost_model.py    - calculates Benefit, Cost, and Risk for each action
policy.py        - the actual decision logic: scores every action, picks the best
eval/            - synthetic incidents used to test the decision logic
tests/           - automated tests
adr/             - short docs explaining why key decisions were made
api.py           - lets other parts of the system call this module over HTTP
```

## Current status

- Core decision logic (`cost_model.py`, `policy.py`) is built
- Next: generating test incidents and measuring how accurate the decisions are
- Later: a small trained model to improve how confident the system is that an
  incident is real, and eventually smarter fix suggestions

## Running it

```
uv sync
uv run pytest
```
