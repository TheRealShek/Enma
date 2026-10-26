# AGENTS.md

## Purpose and scope

Enma is a Rust LLM inference engine built for understanding and measurement. Read [Project intent](docs/PROJECT_INTENT.md) before proposing or implementing work, then consult [Decisions](docs/decisions.md).

Follow the current milestone. Choose routine implementation details independently; ask when an unresolved choice materially changes behavior, architecture, or scope. Do not pull later milestones into a task without agreement.

## Implementation workflow

- Follow `rust-conventions` when writing or changing Rust.
- Inspect existing code before editing. State the behavior being added and how it will be checked.
- Keep each change focused on a complete, understandable behavior, including necessary integration and tests.
- Keep model execution, scheduling, and cache ownership inspectable. Prefer direct code and explicit dependencies over speculative abstractions.
- Make shared state, GPU synchronization, cancellation, and resource reclamation explicit. Check affected callers and failure paths.
- Explain the completed flow with a useful code reading order, its main tradeoff, and what verification establishes.

## Documentation responsibilities

Give each explanation one home and link to it elsewhere:

- `docs/PROJECT_INTENT.md`: goals, non-goals, current checkpoint, milestones, target architecture, open questions, and completion criteria.
- `docs/architecture.md`: implemented flow, component boundaries, code reading order, state ownership, failures, and cleanup.
- `docs/decisions.md`: agreed non-obvious choices, alternatives, reasons, and tradeoffs.
- `docs/manual-testing.md`: setup, configuration, executable checks, expected results, and test commands.
- `docs/benchmarks.md`: measurements, workload, environment, reproduction commands, raw-data references, and limitations.
- `docs/images/`: images referenced by the documentation.
- `AGENTS.md`: repository-specific workflow and documentation rules.

Update affected documents with the implementation. Keep plans in intent, describe implemented behavior from code, and label unverified checks honestly. Every performance claim needs measurements and reproduction details. Replace stale explanations rather than adding progress logs or duplicate feature documents. Check changed links and commands before finishing.
