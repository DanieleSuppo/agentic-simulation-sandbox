# Agentic Simulation Sandbox

Agentic Simulation Sandbox is a public runtime and simulation harness for deterministic, headless environments with interchangeable decision policies and reproducible campaign execution.

The project is being developed against a real, evolving, turn-based game maintained in a separate private repository: `DanieleSuppo/agentic-simulation-reference-game`.

That private repository is the first reference environment and the source of concrete requirements for this runtime. The separation is intentional: this repository must not invent generic abstractions in advance. Runtime capabilities are extracted only when the reference environment demonstrates that they are actually needed.

## Why this exists

The immediate problem is practical: evolve the rules and strategies of a non-trivial multiplayer system and understand their consequences without maintaining one implementation for real gameplay and another, simplified implementation for simulation.

The target model is therefore:

- one authoritative headless engine for the environment;
- the same rule execution for human players, bots and large-scale simulations;
- deterministic execution where possible;
- explicit decision boundaries between the engine and decision makers;
- reproducible campaigns for comparing rule variants and policies;
- optional, granular instrumentation rather than mandatory full tracing;
- AI policies as interchangeable decision makers, not as a requirement of the engine.

The first environment is intentionally private. The public repository focuses on the runtime boundary and simulation infrastructure, while domain rules, content and game-specific strategy remain in the reference repository.

## Core thesis

The engine should know how the world evolves and what decisions are legal.

A policy should know how it prefers to choose among those legal decisions.

A simulation layer should know how to run the engine repeatedly under controlled conditions and collect only the measurements required by the current experiment.

The engine itself must not know whether it is being driven by a human, a bot or a simulation campaign.

## Conceptual model

```text
Authoritative mutable state
          │
        Engine
     ┌────┴─────┐
Observation   Action offers
     └────┬─────┘
          │
     Player policy
          │
       Decision
          │
        Engine
```

The authoritative state is mutable internally for simplicity and performance, but remains private to the engine.

Players and policies receive only:

- a read-only observation containing what that actor is allowed to know;
- structured legal action offers describing what the actor can currently do.

A decision may open further decision points, including decisions owned by other actors. The engine resolves automatic consequences until another external decision is required or the environment terminates.

## Decision points and action offers

The runtime must support structured decision points rather than assuming that every turn consists of one atomic action.

An environment may expose choices such as:

```text
DRAW

PLAY
  source_12 -> targets [actor_2, actor_4]
  source_18 -> targets [self]
  source_21 -> no target
```

The policy chooses within the legal action space. The engine still validates the submitted decision before applying it.

The action vocabulary belongs to the environment. The runtime should provide the protocol for offering and choosing actions without trying to define a universal vocabulary for every possible simulation.

## Determinism and replay

A fundamental invariant is:

> Same engine revision + same environment/rules version + same experimental configuration + same seed + same deterministic policy inputs/decisions = same outcome.

Randomness must therefore be explicit and controlled. Environment code and policies must not use hidden or ambient randomness.

Independent random streams may be introduced when required to avoid accidental coupling between unrelated sources of randomness, for example setup/shuffling, tie-breaks or policy decision noise.

Replay is based primarily on reproducible run identity rather than mandatory persistence of full state or a complete event log.

## Policies

Policies are interchangeable decision makers that consume an observation and legal action offers and return a decision.

The initial direction is deliberately non-LLM-first:

- `RandomPolicy` as a control baseline;
- a scoring/heuristic policy mechanism;
- environment-specific feature extraction and player profiles in the private reference environment;
- controlled decision noise using the run RNG;
- more sophisticated planning, ML or LLM-assisted policies only when they demonstrate additional value.

The public runtime should not contain a universal model of personality, aggression, skill or strategy. Those concepts remain environment-specific until genuine reuse is demonstrated.

## Simulation campaigns

The simulation layer sits outside the game/environment engine.

```text
Campaign definition
        │
    Run generator
        │
Headless engine + policies
        │
 Selected collectors
        │
     Run results
        │
Campaign aggregation / comparison
```

A campaign may control dimensions such as:

- environment/rules variant;
- experimental overrides;
- player/policy population;
- seat or role assignment;
- seed set;
- repetitions;
- selected metric collectors.

The runner should support balanced assignment strategies where positional or role bias matters.

Comparative experiments should use paired conditions by default when possible: the same population, assignment and seed are applied to baseline and treatment variants so differences are easier to attribute to the changed rule or policy.

The runtime is not intended to become a general statistical analysis package. It should generate trustworthy, well-identified experimental data that can be analysed using appropriate external tools.

## Run identity and results

Each run should be identifiable by enough information to reproduce it, including the relevant engine/environment revision, rule or environment version, experimental overrides, policy population, assignment and seed.

A run stores only the metrics requested by the campaign, not the full authoritative state by default.

Campaigns retain per-run results so anomalous or interesting runs can be identified and rerun later with different instrumentation.

## Hooks and metric collectors

The engine exposes observation hooks for code that needs to inspect execution without influencing it.

Subscribers receive:

```text
(event, readOnlyEnvironmentView)
```

The event describes what just happened. The read-only view exposes the environment state that an authorised inspector may observe at that point.

Metric collectors are granular pieces of code selected by the campaign. A collector might measure one specific question and subscribe only to the hooks it needs.

There are intentionally no mandatory telemetry tiers and no requirement to record every event.

If a later investigation needs different data, the same run can be reproduced using the same identity and seed with additional collectors enabled.

Instrumentation must be causally inert: enabling or disabling a collector must not change environment behaviour, consume environment RNG, alter legal actions or affect policy decisions.

## Author and inspector access

Normal players and policies receive only their own observations.

The runtime may also expose a privileged, read-only inspection capability for authoring, debugging and analysis. This is conceptually distinct from a player with extra permissions.

Future inspection capabilities may include snapshots, targeted logging, run breakpoints or richer debugging views, but they should be added only when a concrete need appears.

## Rules and configuration philosophy

The current philosophy is code-first:

> Rules are code. Parameters are experimental knobs when they are useful for testing a hypothesis.

The runtime is not intended to provide a universal declarative rules DSL.

When an experimental variant needs to vary a rule repeatedly, that dimension may temporarily become configurable. Once the design decision is settled, the chosen behaviour can return to canonical code rather than leaving a permanent accumulation of historical feature flags.

Backward reproducibility should rely on versioned software and run identity, not on keeping every legacy rule switch in the current runtime forever.

## Public runtime vs private reference environment

The dependency direction is one-way:

```text
agentic-simulation-sandbox          PUBLIC
        ↑ runtime dependency
        │
agentic-simulation-reference-game   PRIVATE
```

This repository owns runtime and simulation concerns that do not require knowledge of the reference game's domain.

The private reference repository owns the real game's:

- authoritative domain state;
- setup and turn structure;
- rules and rule-specific transitions;
- cards/content and their effects;
- victory conditions;
- game-specific feature extraction and policy profiles;
- game-specific metric collectors;
- balancing experiments and experimental variants.

The private environment must not be distorted to fit abstractions invented here. If it exposes a missing reusable runtime capability, that need should be evaluated explicitly and, if appropriate, implemented upstream in this repository.

See [`docs/BOUNDARIES.md`](docs/BOUNDARIES.md) for the current architectural boundaries and decisions that should be treated as constraints during design work.

## Initial MVP direction

The first useful milestone is intentionally narrow. The public runtime should be sufficient for the private reference environment to implement an authoritative headless game engine and run reproducible automated campaigns.

The expected first capabilities are:

- environment/engine lifecycle boundary;
- authoritative state encapsulation;
- actor observations;
- structured legal action offers;
- multi-step decision points;
- controlled deterministic RNG;
- policy interface;
- random baseline policy;
- hooks and read-only inspector access;
- code-defined granular collectors;
- run identity;
- campaign execution;
- balanced assignments where required;
- paired seed execution for comparisons;
- per-run metric results and campaign aggregation.

The exact API and package boundaries are intentionally not fixed by this README. They should be resolved from the needs of the reference environment before implementation.

## Non-goals for the initial project

The initial scope does not include:

- a universal simulation DSL;
- a generic rule language;
- a no-code environment builder;
- networking, lobbies, accounts or multiplayer transport;
- a game UI;
- event sourcing as the authoritative state model;
- mandatory full execution tracing;
- a generic analytics/statistics platform;
- a universal ontology for agents or personalities;
- LLM execution for every decision;
- a synthetic company or generic agent benchmark arena;
- abstractions added solely because another future environment might need them.

## Development approach

Development is vertical-driven.

1. The private reference environment exposes a concrete need.
2. Decide whether that need belongs to the environment or to the reusable runtime.
3. Keep domain-specific behaviour private.
4. Add a runtime capability here only when the environment genuinely requires it.
5. Validate the public abstraction by consuming it from the private environment.
6. Generalise further only after another independent environment demonstrates compatible needs.

This repository may eventually provide reusable simulation or agent-evaluation primitives beyond the initial game. That is a hypothesis to validate, not a requirement to satisfy in advance.

## Current status

The project is in design/discovery. The first architectural boundaries have been agreed, but the runtime API and implementation have not yet been specified.

The next intended step is a Wayfinder design session using both this repository and the private reference environment as context, followed by a SPEC once the remaining implementation-relevant decisions are resolved.
