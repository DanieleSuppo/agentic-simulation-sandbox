# Agentic Simulation Sandbox Context

## Terms

### Environment

The domain-owned authoritative engine that holds its mutable state privately, resolves automatic work, validates submitted decisions, and exposes only neutral runtime boundaries.

### Decision Boundary

The point at which an Environment has resolved all automatic work that it can and must wait for one external decision. The runtime does not classify why this boundary exists.

### Pending Decision

The serializable, actor-scoped description of a Decision Boundary. It has a stable identity and carries the actor's observation and the Environment-defined structured action offer.

### Decision

The serializable, Environment-defined response to one Pending Decision. The Environment validates it against the current Pending Decision before advancing.

### Terminal Outcome

The serializable, Environment-defined result produced only when an Environment has finished. Its meaning remains domain-owned.

### Inspection Projection

An Environment-defined, privileged, coherent, and non-mutating view supplied to authorised subscribers. It is distinct from an actor observation and never exposes mutable authoritative state, randomness, or decision controls.

### Collector

A per-run, code-defined observer selected by a Campaign. It receives Environment events and Inspection Projections, then produces one small serializable result. A collector failure invalidates the Campaign execution rather than producing partial results.

### Random Source

A deterministic random stream derived by the runner from a run seed. The Environment and each policy receive isolated streams; named child streams are introduced only for demonstrated coupling concerns.

### Run Identity

The serializable record required to reproduce one simulation scenario: runtime and Environment revisions, Environment identity and overrides, policy population, assignment, root seed, and technical execution limits. Collector selection is execution metadata, not part of the semantic identity.

### Run Plan

One fully resolved Run Identity ready for execution.

### Campaign

An ordered, reproducible generator and executor of Run Plans. It retains every per-run result and uses no built-in statistical interpretation.

### Initial Assignment

The stable mapping of policy instances to actor identities or initial slots. It is distinct from Environment-owned dynamic turn ordering.

### Paired Comparison

A comparison that runs each named condition from the same population, initial assignment, and seed, then links the resulting runs with a pair key.

### First Vertical Slice

The smallest executable proof of the public runtime: one seeded reference-environment run driven through the normal Environment and policy boundary by `RandomPolicy`, with an identical rerun producing the same serializable terminal outcome and invalid submissions leaving the Environment unchanged.
