# Architectural Boundaries

This document records design decisions already made during discovery for Agentic Simulation Sandbox.

It is intentionally not a SPEC. It exists to prevent future design work from reopening settled product boundaries without concrete evidence from the reference environment.

The public runtime is developed against the private repository `DanieleSuppo/agentic-simulation-reference-game`, which is the first authoritative reference environment.

## 1. Vertical-first rule

The private reference environment drives requirements.

The public repository must not invent abstractions merely because they might be useful to hypothetical future simulations.

A new public primitive should normally answer the question:

> Which concrete need in the reference environment required this capability?

If that question cannot be answered, the abstraction probably does not belong in the public runtime yet.

## 2. Repository ownership

### Public repository: `agentic-simulation-sandbox`

Owns reusable runtime and simulation concerns that do not require knowledge of the reference game's domain.

Candidate responsibilities include:

- environment lifecycle contracts;
- decision-point protocol;
- actor observations;
- structured legal action offers;
- controlled RNG and deterministic execution support;
- policy interfaces;
- basic reusable policy machinery where genuinely domain-independent;
- run identity;
- campaign execution;
- balanced assignment infrastructure;
- paired run execution;
- hooks/subscriber infrastructure;
- read-only privileged inspection support;
- metric collector integration;
- per-run result handling and campaign aggregation.

### Private repository: `agentic-simulation-reference-game`

Owns the real game domain.

Responsibilities include:

- authoritative game state;
- concrete game setup;
- turn and round semantics;
- roles;
- cards and other game content;
- concrete rules;
- card/effect implementation;
- victory conditions;
- game-specific action vocabulary;
- game-specific observations;
- game-specific scoring features;
- game-specific player profiles;
- game-specific collectors;
- balancing variants and experiments.

## 3. Dependency direction

The dependency direction is one-way:

```text
agentic-simulation-sandbox          PUBLIC
        ↑
        │ runtime dependency
        │
agentic-simulation-reference-game   PRIVATE
```

The public runtime must never depend on the private game.

The private game may depend on the public runtime.

Cross-repository design work may inspect both repositories, but domain-specific concepts must not leak into the public repository merely to make implementation convenient.

## 4. One authoritative engine

The first reference environment must have one authoritative implementation of its rules.

The same rule implementation must be usable for:

- a future real human-driven game;
- bot-driven play;
- large-scale headless simulation.

The simulation layer must not contain a second simplified implementation of game mechanics.

The engine must not know whether it is currently being used by a human client, a bot or a simulation campaign.

There must be no semantic `simulationMode` branch in the rules.

Performance optimisations are allowed only if they preserve exactly the same game semantics.

## 5. Authoritative state

The authoritative environment/game state may be mutable internally.

This is a deliberate choice for implementation simplicity and high-throughput simulation.

Constraints:

- the mutable authoritative state is private to the engine;
- external callers do not receive references that allow them to mutate it;
- policies and normal players receive actor-specific observations instead;
- rule/card code must not mutate arbitrary state directly if the mutation represents a domain operation that should preserve invariants;
- privileged inspection is read-only.

State immutability is not an architectural requirement.

If future planning/search policies require branching, snapshot/clone/checkpoint support may be added when needed rather than redesigning the whole engine around immutable state now.

## 6. Engine ↔ decision-maker boundary

The engine owns legality and resolution.

The decision maker owns preference.

The normal flow is:

```text
Authoritative state
      ↓
    Engine
      ↓
Observation + legal action offers
      ↓
Player / policy
      ↓
Decision
      ↓
    Engine
```

A policy must not reimplement game legality merely to determine what it is allowed to do.

The engine exposes structured legal action offers.

The final submitted decision is still validated by the engine.

## 7. Observations

A normal player or bot sees only information available to that actor under the rules.

The engine may expose deterministic facts derived directly from the rules, such as legal targets or remaining actions.

The engine should not expose strategic judgements such as the best target, threat rankings or estimated action quality.

Strategic interpretation belongs to the policy layer.

A future Author/Inspector capability may observe the complete state through a privileged read-only view.

That inspector is not a player with extra privileges.

## 8. Multi-step decisions

An action may open further decision points.

The engine must not assume that a turn is equivalent to one atomic player command.

Examples of the required shape include:

```text
actor A chooses an action
→ automatic consequences
→ actor B must react or choose
→ automatic consequences
→ another decision may be required
```

The engine resolves deterministic/automatic consequences until:

- another external decision is needed; or
- the environment terminates.

The call stack must not be the only representation of a pending external decision.

The exact pending-resolution/state-machine representation remains a design question for Wayfinder/SPEC.

## 9. Waiting and online play

The engine does not model networking or asynchronous delivery.

If a player is slow, the environment simply remains at the current decision point until a decision is provided.

Future concerns such as:

- network transport;
- lobbies;
- reconnects;
- clocks/timeouts;
- authentication;

belong outside the game engine.

## 10. Rules are code

Rules are code by default.

The project explicitly rejects the requirement that all game rules or cards be declarative or editable without development.

If implementing a new rule or card correctly requires code, code should be written.

Configuration exists where it provides current experimental value, not as a universal design goal.

The preferred lifecycle is:

```text
canonical rule in code
        ↓
needs experimental variation
        ↓
temporary configurable parameter/override
        ↓
experiment resolves design decision
        ↓
chosen behaviour becomes canonical code
```

Avoid permanent accumulation of historical feature flags.

Backward reproducibility should use engine/environment version identity rather than keeping all old rule paths active forever.

## 11. Card/effect logic

The public runtime must not require card concepts.

Within the private reference game, cards and special effects may be implemented with normal domain code.

Game-specific effect code should interact with protected engine/domain operations rather than freely mutating arbitrary internal state where doing so would bypass invariants.

No universal effect DSL is an MVP goal.

## 12. Determinism

Strong reproducibility is a core invariant:

> Same relevant software versions + same experimental configuration + same seed + same deterministic decisions/policy inputs must produce the same outcome.

Randomness must be explicit and controlled.

Hidden/ambient sources of non-determinism must be avoided, including accidental direct use of global random functions, wall-clock time or non-deterministic ordering where it affects behaviour.

Independent RNG streams may be introduced when a real need exists to prevent unrelated randomness from contaminating paired comparisons.

Examples may include:

- setup/shuffle randomness;
- tie-break randomness;
- policy decision noise.

Do not introduce unnecessary RNG hierarchy before it is useful.

## 13. Policies

All decision makers should share the same policy boundary where practical.

Initial priorities:

- `RandomPolicy` as a baseline/control;
- a scoring-based heuristic policy approach;
- environment-specific feature extraction;
- parameterised profiles/personalities;
- controlled decision noise.

Do not model every behavioural archetype as a separate bot implementation when profiles can express the variation.

Do not create a universal personality ontology.

Skill, planning depth and other higher-order concepts can be explored later if needed.

LLM-native or LLM-assisted policies are later experiments, not a runtime assumption.

## 14. Population scenarios

Initial simulations should use explicit, repeatable policy populations.

Prefer named/configured scenarios such as balanced, aggressive, conservative or mixed over immediately sampling arbitrary behavioural distributions.

Population definition is distinct from seat/role assignment.

The campaign runner should make it possible to rotate the same profiles through seats/roles so policy effects are not confused with positional bias.

## 15. Campaigns and runs

The engine does not know about campaigns.

The simulation layer repeatedly drives the engine through the same external decision interface used by other decision makers.

A campaign controls relevant experimental dimensions such as:

- rule/environment variant;
- experimental overrides;
- population scenario;
- assignment strategy;
- seeds;
- repetitions;
- selected collectors.

A run is one fully identified execution.

A campaign generates many runs.

A comparison relates compatible runs/campaigns across variants.

Do not build a universal experiment-design framework in the MVP.

## 16. Bias control

Where relevant, the runner should reduce obvious experimental confounding through explicit mechanisms rather than relying on the caller to remember them.

Initial examples:

- balanced seat/role rotations;
- paired seeds between baseline and treatment;
- stable population definitions;
- explicit tie-break randomness;
- reproducible assignment identity.

The runtime is responsible for generating trustworthy experimental conditions, not for becoming a complete statistical inference library.

## 17. Run identity and historical replay

Each run must carry enough identity to reproduce it later, including the relevant software/rule/environment version information, experimental configuration, policy population, assignment and seed.

Do not require persistence of the full authoritative state for every run.

When versions evolve, historical reproducibility should normally rely on the corresponding version/commit rather than compatibility code inside the latest engine.

If future non-deterministic actors such as external LLMs prevent exact decision reconstruction, recording their decisions may become necessary. Do not add that requirement to deterministic baseline runs prematurely.

## 18. Hooks and subscribers

The engine exposes hook points for observation.

Subscribers receive conceptually:

```text
(event, readOnlyEnvironmentView)
```

The event identifies what happened.

The read-only view allows an authorised observer to inspect state at that moment.

The event does not need to duplicate every state value that a future collector might want.

Hooks are observation infrastructure, not the authoritative state model.

The project does not require event sourcing.

## 19. Metric collectors

Collectors are granular, code-defined measurement components selected explicitly for a campaign.

Example shape:

```text
collect:
  - win_rate_by_role
  - empty_resource_at_turn_end
```

A new measurement may require writing a new collector. This is acceptable and preferred over premature generic analytics abstractions.

Only requested measurements should incur collection/storage cost.

There are no mandatory telemetry tiers.

A run can be executed again later with additional collectors if more information is needed.

Collectors must be causally inert:

- no state mutation;
- no policy decisions;
- no environment RNG consumption;
- no changes to legal actions;
- no changes to execution ordering.

Enabling different collectors must not change the outcome of a deterministic run.

## 20. Per-run results

Initial preference:

- collectors produce small per-run outputs;
- run identity is retained with those outputs;
- campaign aggregation is derived from per-run results;
- full game state is not persisted by default.

This supports both efficient aggregate analysis and later identification/reproduction of anomalous runs.

Do not optimise prematurely for extremely large result volumes. Streaming-only aggregation can be introduced if actual scale requires it.

## 21. Inspector/author tooling

A future author or inspector should be able to observe the real environment state without being constrained to a player's partial observation.

Possible future capabilities include:

- state snapshots;
- richer logging;
- targeted run inspection;
- conditional breakpoints;
- state queries.

These are explicitly deferred until concrete authoring/debugging needs require them.

## 22. Explicit initial non-goals

Do not design the MVP around:

- generic multi-agent research abstractions unrelated to the reference environment;
- a universal simulation language;
- a rule DSL;
- no-code rule editing;
- generic state machines for every possible domain;
- event sourcing;
- networking or multiplayer transport;
- UI;
- account/session infrastructure;
- a generic statistics package;
- full tracing of every run;
- universal agent/personality models;
- LLM agents as the default execution model;
- synthetic-company scenarios;
- speculative support for future marketplace, workflow or organisation simulations.

## 23. How Wayfinder should use this document

Wayfinder should treat the decisions above as current constraints, not as questions to reopen automatically.

It should focus on unresolved implementation-relevant questions required to move the public runtime to SPEC.

A settled boundary may be challenged only when one of the following is true:

1. the private reference environment demonstrates that the boundary cannot support a real rule or interaction;
2. two settled decisions conflict in implementation;
3. the choice creates a concrete correctness, reproducibility or maintainability failure;
4. a substantially simpler design achieves the same requirements without violating the product thesis.

When such a conflict appears, document the evidence and the consequence before changing the boundary.

The private reference environment should be consulted freely during design, but game-specific terminology, content and mechanics should remain private unless the public runtime genuinely requires a neutral abstraction derived from them.
