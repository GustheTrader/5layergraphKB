<!--
SYNC IMPACT REPORT
==================
Version change: TEMPLATE → 1.0.0
Ratification date: 2026-05-03 (initial adoption)
Last amended: 2026-05-03

Modified principles:
  [PRINCIPLE_1_NAME] → I. PRD as Source of Truth (new — initial fill)
  [PRINCIPLE_2_NAME] → II. Test-Driven Development / TDD (new — initial fill)
  [PRINCIPLE_3_NAME] → III. Spec-Driven Development / SDD (new — initial fill)
  [PRINCIPLE_4_NAME] → IV. Bi-Temporal & Regime-Aware Data Modeling (new — initial fill)
  [PRINCIPLE_5_NAME] → V. Plan-Mode Default (new — initial fill)
  [PRINCIPLE_6_NAME] → VI. Observability as a First-Class Citizen (new — added beyond template default of 5)
  [PRINCIPLE_7_NAME] → VII. Simplicity & Minimal Impact (new — added beyond template default of 5)

Added sections:
  § Performance & Reliability Standards (from PRD §11 NFRs)
  § Development Workflow & Self-Improvement (from user workflow orchestration input)

Removed sections:
  [SECTION_2_NAME] placeholder → replaced by Performance & Reliability Standards
  [SECTION_3_NAME] placeholder → replaced by Development Workflow & Self-Improvement

Templates requiring updates:
  ✅ .specify/templates/plan-template.md — Constitution Check gates updated with real gates
  ✅ .specify/templates/tasks-template.md — Test tasks updated from OPTIONAL to MANDATORY (TDD)
  ✅ .specify/templates/spec-template.md — PRD traceability field added
  ⚠️  .specify/templates/commands/ — Directory does not exist; no command templates to update

Deferred TODOs:
  None — all fields resolved from PRD, user input, and repository context.
-->

# Real-Time Knowledge Base (KB) System Constitution

## Core Principles

### I. PRD as Source of Truth

The PRD at `docs/prd/PRD_Layer1_RealtimeKnowledgeBase.md` is the canonical reference for all
architectural decisions, data models, and system behavior. Every requirement, node schema, edge
type, regime classification rule, and API contract MUST trace back to first-principles reasoning
documented in the PRD.

- New requirements MUST be added to the PRD before implementation begins.
- Any deviation from the PRD MUST be documented with explicit justification.
- In case of conflict between code, spec, plan, or any other artifact and the PRD, the PRD
  governs.
- The PRD is the domain-expert voice; all implementation artifacts serve it, not the other way
  around.

**Rationale**: The PRD was authored using First Principles decomposition tracing every requirement
to an atomic truth about financial markets. Deviating from it without amendment first creates
un-traceable technical debt and breaks the causal chain between design intent and implementation.

### II. Test-Driven Development (NON-NEGOTIABLE)

TDD MUST be practiced on all implementation work without exception. The Red-Green-Refactor cycle
is strictly enforced:

1. Write tests describing the required behavior → tests MUST fail before any implementation.
2. Present failing tests for review → receive approval before proceeding.
3. Implement the minimum code to make tests pass.
4. Refactor while keeping all tests green.

- Test files MUST exist and MUST fail before any implementation file is written.
- No component is considered done without passing tests at unit, integration, and contract levels.
- For regime-sensitive logic, tests MUST cover all four regime states:
  `EXPANSION`, `CONTRACTION`, `BREAKOUT`, `CRISIS`.
- Bi-temporal queries MUST be tested against deterministic replay scenarios.

**Rationale**: The KB layer is the epistemic foundation for all upper layers (Attention, Social
Graph, Causal DAG). Untested behavior here corrupts all downstream reasoning. Test-first is not a
development preference — it is a reliability requirement for a system that trades on correctness.

### III. Spec-Driven Development (SDD)

Every feature or component MUST have a written spec in `.specify/specs/` before implementation
begins. The spec MUST include:

- User scenarios with **Given/When/Then** acceptance criteria.
- Functional requirements using MUST/SHOULD language.
- Measurable success criteria that are technology-agnostic and verifiable.
- PRD traceability: each spec MUST reference the PRD section(s) it implements.

No implementation begins without an approved spec. Approval means explicit user sign-off or
confirmed alignment with the PRD.

**Rationale**: Upfront spec writing surfaces ambiguity before code is written. In a multi-layer
system where KB contracts are consumed by three upper layers, interface ambiguity discovered in
implementation is costly to unwind — especially given bi-temporal and regime-conditional
constraints.

### IV. Bi-Temporal & Regime-Aware Data Modeling

All data entities MUST carry the bi-temporal contract defined in PRD §9:

- `valid_time` (VT): when the fact was true in reality.
- `transaction_time` (TT): when the system recorded the fact.

All graph nodes and edges MUST be tagged with the `regime_state` active at time of creation or
mutation. Valid regime states are: `EXPANSION`, `CONTRACTION`, `BREAKOUT`, `CRISIS`.

- No entity may be persisted without both temporal fields populated.
- No graph mutation may be written without a `regime_state` tag.
- The ingestion lag (`TT − VT`) MUST be tracked as a first-class metric with alerting.
- Temporal replay MUST reconstruct the exact graph state at any historical timestamp.
- Schema changes MUST be backward-compatible: fields are deprecated, never deleted.

**Rationale**: Per PRD §2.3, the gap between valid_time and transaction_time is where alpha lives.
Flattening these dimensions destroys the most exploitable information in financial markets, makes
backtesting non-deterministic, and violates the foundational information contract of the system.

### V. Plan-Mode Default

Any task involving 3 or more implementation steps, or any task requiring an architectural
decision, MUST enter plan mode before execution begins.

- Plans MUST be written to `tasks/todo.md` with checkable items before any code is written.
- Implementation MUST NOT begin until the plan is verified and approved.
- If execution goes sideways, work MUST STOP immediately — re-plan before resuming.
- Subagents MUST be used to parallelize independent research and analysis tasks to keep the
  main context window clean.
- After any user correction, `tasks/lessons.md` MUST be updated with the pattern, root cause,
  and a rule that prevents recurrence.

**Rationale**: Unplanned implementation in a multi-layer graph system with strict bi-temporal
contracts causes cascading schema violations. The cost of planning is always lower than the cost
of unplanned rework on interdependent layers with live downstream consumers.

### VI. Observability as a First-Class Citizen

All system components MUST be instrumented per PRD §11.4. No component is considered complete
without passing observability requirements:

- Ingestion pipelines MUST emit structured logs containing:
  `asset_id`, `regime_state`, `valid_time`, `tx_time`, `pipeline_stage`.
- The graph health dashboard MUST expose: node count by type, edge count by type, current
  regime state per asset, ingestion lag distribution, event log consumer offset lag.
- Regime transition events MUST be independently tracked in a dedicated metrics stream with
  alerting thresholds.
- Observability tasks are MANDATORY in every feature task list — they are not polish or optional
  polish phases.

**Rationale**: In a regime-adaptive system, silent failures (wrong regime state, stale graph nodes,
high ingestion lag) are indistinguishable from correct behavior without instrumentation. Observ-
ability is the only mechanism that proves the system is operating correctly in production.

### VII. Simplicity & Minimal Impact

Every change MUST be as simple as possible and MUST touch only what is necessary.

- No abstraction may be introduced before it is required by at least two concrete use cases.
- No feature may be implemented for a hypothetical future requirement.
- Temporary fixes are PROHIBITED — root causes MUST be identified and resolved.
- Three similar lines of code are preferable to a premature abstraction.
- All changes MUST satisfy the standard: "Would a staff engineer approve this without hesitation?"
- Bug reports are resolved autonomously — no hand-holding required from the user.

**Rationale**: The KB layer is the foundation of a multi-layer system. Unnecessary complexity at
Layer 1 propagates upward through Attention, Social Graph, and Causal DAG layers. Simplicity
compounds as a force multiplier; complexity compounds as a liability.

## Performance & Reliability Standards

Performance and reliability targets are non-negotiable hard limits derived from PRD §11.
All targets MUST be verified by load tests and MUST be included in the definition of done for
any component that touches the ingestion or query path.

### Performance Targets

| Metric | Target | Hard Limit |
|--------|--------|------------|
| Ingestion throughput | 50,000 events/sec | 10,000 events/sec minimum |
| Graph write latency (p99) | < 10ms | < 50ms |
| Graph point query latency (p99) | < 5ms (hot store) | < 100ms (warm store) |
| Regime transition notification latency | < 100ms end-to-end | < 500ms |
| Temporal replay speed | 100× realtime | 10× realtime minimum |
| Ingestion lag (p99) | < 200ms (VT to graph) | monitored continuously |

### Reliability Targets

| Requirement | Specification |
|-------------|---------------|
| Availability | 99.9% uptime during market hours |
| Event log durability | Zero data loss (Kafka replication factor ≥ 3) |
| Recovery Time Objective (RTO) | < 5 minutes (hot store loss → cold store replay) |
| Recovery Point Objective (RPO) | < 1 second (event log lag) |
| Temporal replay accuracy | 100% match between replay and live graph state |
| Schema evolution | Zero-downtime deployments — 100% of schema changes |

## Development Workflow & Self-Improvement

### Execution Flow

1. **Spec First**: Write feature spec to `.specify/specs/` before planning. Reference PRD section.
2. **Plan First**: Write plan to `tasks/todo.md` with checkable items. Verify before starting.
3. **Tests First**: Write failing tests before writing implementation (Principle II).
4. **Implement**: Write minimum code to make tests pass.
5. **Verify Before Done**: A task is complete only when behavior is proven — run tests, check
   logs, demonstrate correctness. "Would a staff engineer approve this?" is the standard.
6. **Track Progress**: Mark `tasks/todo.md` items complete immediately as each is done.
7. **Document Results**: Add a review section to `tasks/todo.md` after feature completion.
8. **Capture Lessons**: Update `tasks/lessons.md` after any correction with: pattern observed,
   root cause, and rule to prevent recurrence.

### Subagent Strategy

- Subagents MUST be used for research, exploration, and parallelizable analysis tasks.
- The main context window MUST be kept clean — offload work that does not require main context.
- Each subagent MUST have exactly one focused task.
- For complex problems, additional compute via subagents is preferred over expanding the main
  agent's scope.

### Bug Fixing Protocol

When given a bug report, fix it autonomously:
- Point at logs, errors, and failing tests — then resolve them.
- No context switching required from the user.
- Go fix failing CI tests without waiting to be told how.

### Lessons Review

At the start of each session, review `tasks/lessons.md` for patterns relevant to the current work.
The lessons file is authoritative for known failure modes in this project.

## Governance

This constitution supersedes all other development practices, guidelines, and conventions for this
project. Amendments require:

1. A clear description of the principle or section being changed.
2. Explicit justification — what problem does the amendment solve?
3. Version bump per semantic versioning:
   - **MAJOR**: Backward-incompatible governance/principle removals or redefinitions.
   - **MINOR**: New principle or section added, or materially expanded guidance.
   - **PATCH**: Clarifications, wording fixes, non-semantic refinements.
4. All dependent templates (plan, spec, tasks) MUST be reviewed for alignment after any amendment.
5. The Sync Impact Report MUST be updated at the top of this file on every amendment.

Compliance is verified at the **Constitution Check** gate in every feature plan. All work products
(specs, plans, tasks, code) MUST be traceable to a principle in this constitution.

For runtime development guidance, refer to:
- `tasks/lessons.md` — session-specific learned failure patterns
- `docs/prd/PRD_Layer1_RealtimeKnowledgeBase.md` — canonical domain authority

**Version**: 1.0.0 | **Ratified**: 2026-05-03 | **Last Amended**: 2026-05-03
