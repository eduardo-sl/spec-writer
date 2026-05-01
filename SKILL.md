---
name: spec-writer
description: Use this skill when the user asks to "write a spec", "create a spec", "spec out", "plan the implementation", "draft an implementation plan", "create a feature spec", "document how to implement", or otherwise needs a detailed implementation specification for a feature, integration, refactor, library, CLI, UI flow, data pipeline, or any other piece of work. Produces project-grounded specs with explicit goals and non-goals, decisions paired with rejected alternatives, testable acceptance criteria, and a verifiable definition of done. Works for any language, framework, or domain.
version: 2.0.0
license: MIT
---

# spec-writer

Produce implementation specs grounded in the **actual project** — real names, real paths, real constraints — so an implementing agent (or human) can build the feature without mid-task clarifying questions.

This skill does not write code, review code, or resolve undecided architectural questions on its own. When a question is genuinely open, mark it `[NEEDS CLARIFICATION: ...]` and surface it; do not silently invent the answer.

This file is a single Markdown document so any AI coding tool can use it: Claude Code (reads the frontmatter for routing), Cursor (drop into `.cursor/rules/`), Windsurf, GitHub Copilot, Aider (`--read SKILL.md`), Codex, Continue, and similar tools that load instruction files. The body avoids tool-specific commands.

## Output

Default location: `specs/<feature-name>/SPEC.md` — lowercase, hyphenated, one spec per directory. Create `specs/` if it does not exist. If the user specifies a different path, use it.

If `specs/README.md` does not exist, create it as an index: name | status (`draft` / `ready` / `in progress` / `done`) | dependencies on other specs | primary files touched.

## Process

Five phases. Phases 1–3 are where most specs are won or lost; phase 5 catches the rest. None can be skipped.

### Phase 1 — Investigate the project

The single most common failure mode is generating a generic spec because the codebase was never opened. Read first:

- The project's primary agent context (in priority order — first match wins): `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `.cursor/rules/`, `CONVENTIONS.md`, `README.md`. If one exists, it is the source of truth — read it in full.
- Existing specs (under `specs/`, `docs/`, `rfcs/`, or wherever convention places them). Match their style and depth.
- The dependency manifest for whichever ecosystem applies (`go.mod`, `package.json`, `Cargo.toml`, `pyproject.toml`, `requirements.txt`, `pom.xml`, `build.gradle`, `Gemfile`, `*.csproj`, `pubspec.yaml`, etc.).
- The entry point(s) — wiring / bootstrap / main / app composition root.
- Any environment or configuration contract (`.env.example`, `config.*`, settings schema).
- Infrastructure manifests if the work touches them (`docker-compose.*`, `k8s/`, `terraform/`).
- The directory containing the area being modified — at least three representative files so the spec can match local conventions.

Then **detect the feature class** before drafting — it determines which sections of the template apply:

| Class | Examples | Sections that usually apply | Sections often omitted |
|---|---|---|---|
| Backend service / API | endpoints, jobs, integrations | wiring, infra, config, observability, rollout | UX |
| Frontend / UI feature | pages, components, flows | components, state, routing, accessibility, analytics | infra |
| Library / SDK | public package, internal lib | public API, semver impact, examples, deprecations | wiring, infra |
| CLI / script / automation | commands, tooling | command surface, exit codes, IO contract | UX |
| Data / pipeline / ML | ETL, training, inference | schema, lineage, idempotency, backfill | UX |
| Refactor / migration | restructuring without new behavior | invariants, equivalence proof, rollout, rollback | new behavior |

The spec uses **real names from this project**. Placeholder identifiers (`SomeService`, `MyRepo`, generic `Handler`) are a sign the codebase was not read; rewrite before delivering.

### Phase 2 — Clarify before drafting

Identify what is **decided**, what is **assumed**, and what is **unknown**. Ask the user about the unknowns instead of inventing answers — but ask in one batch, not one question at a time. Strong cues that something needs clarification:

- The choice has security, durability, or money consequences.
- Two reasonable engineers would pick differently with the information you have.
- The user's constraints contradict something you found in the codebase.
- A non-goal is unclear (often the actual scope question).

For anything still unresolved at draft time, mark it inline:

```
[NEEDS CLARIFICATION: token-bucket per-IP, per-user, or both?]
```

A spec is not "ready" while any `[NEEDS CLARIFICATION]` remains. Collect them under "Open questions" at the top so they cannot be missed.

### Phase 3 — Justify the approach

For each material decision (technology, pattern, integration boundary, abstraction shape), answer two questions before documenting the implementation:

1. **Why this, for this project?** Reference an existing constraint, pattern, or decision in the codebase. Generic praise of the technology is rejected.
2. **Why not the obvious alternative?** Name at least one alternative and explain the trade-off in one sentence grounded in a real constraint — not "it's more complex".

If you cannot answer both questions specifically, the decision is not ready to spec. Read more code or ask the user.

Generic, rejected:
> Redis is widely adopted with excellent client support.

Project-grounded, accepted:
> Cache-aside puts invalidation in the application — grep-able and consistent with the explicit-over-magic rule already established in `AGENTS.md`. Write-through would coordinate cache writes with the existing transactional outbox in `Service.Register()`, putting two responsibilities in one transaction.

### Phase 4 — Draft the spec

Use the template below. Include only sections that apply to the feature class. For each omitted optional section, leave a one-line `N/A — <reason>` so reviewers know it was considered.

Write requirements as testable statements. EARS patterns are useful for that:

- Ubiquitous — `The <system> shall <response>.`
- Event — `When <trigger>, the <system> shall <response>.`
- State — `While <state>, the <system> shall <response>.`
- Unwanted — `If <unwanted condition>, then the <system> shall <response>.`
- Optional — `Where <feature included>, the <system> shall <response>.`

Use RFC 2119 keywords (MUST / SHOULD / MAY) only when distinguishing a hard requirement from a preference matters. Don't pepper the spec with all-caps directives — overuse drains their meaning.

Each requirement gets an ID (`R1`, `R2`, …). Each task in the implementation plan back-references the requirement(s) it satisfies, so traceability is one grep away.

### Phase 5 — Self-check before saving

Verify each item. Anything that fails must be addressed before delivering.

1. Did I read the project's primary context document and the area being modified?
2. Are all identifiers in code blocks real names from this project?
3. Is every material decision paired with at least one named, dismissed alternative?
4. Are non-goals explicit — not just implied by the goal?
5. Are acceptance criteria testable? Could a different person verify each one without asking me?
6. Does each Definition-of-Done item have an objective verification — a command, a file existence check, an observable behavior — not a feeling?
7. Are open questions surfaced under "Open questions" rather than silently resolved?
8. For any external dependency: is the failure-mode behavior specified (degrade, retry, fail-fast)?
9. For any feature flag or staged rollout: is the rollback path and the flag-removal trigger documented?
10. Is the spec the right size? A 50-page monolith should be split into a graph; a 5-line stub usually skipped phases 1–3.

## Spec template

Sections marked **required** must appear. Sections marked **conditional** appear only when the feature class needs them; otherwise leave a one-line `N/A — <reason>`.

```markdown
# <Feature Name> — Spec

**Status:** draft | ready | in progress | done
**Owner:** <single name>
**Last updated:** <ISO date>
**Related:** <ADRs, issues, prior specs, PRs>

## 1. Summary (required)
Two to four sentences. What this adds, who benefits, what it explicitly is **not**, and the most common over-engineering trap for this kind of feature.

## 2. Goals and Non-goals (required)

### Goals
- G1 — ...
- G2 — ...

### Non-goals
- N1 — This spec does **not** cover ...
- N2 — ...

Non-goals do more work than goals — they bound the scope. Be specific.

## 3. Open questions (required while non-empty; remove the section once empty)
- [NEEDS CLARIFICATION: ...]

## 4. Requirements (required)
Numbered, testable. EARS patterns where useful.
- R1 — When <trigger>, the system shall <response>.
- R2 — The system shall <ubiquitous requirement>.
- R3 — If <unwanted condition>, then the system shall <response>.

## 5. Decisions and Alternatives Considered (required)
For each material decision:

### D1 — <decision>
- **Chosen:** ...
- **Why this for this project:** reference real constraint or existing pattern.
- **Alternatives considered:**
  - <alt 1> — rejected because ...
  - <alt 2> — rejected because ...
- **Consequences:** positive, negative, neutral.

## 6. Design (required)
A description at one consistent level of detail. Include only what's new or changed:

- Component / module boundaries — use real names.
- Data model deltas — schemas, types, ownership.
- Public API or interface surface.
- Sequence of operations for the main flow(s) — ASCII diagram or numbered steps.
- Error model — how failures propagate and what the caller sees.

Place abstractions at the **consumer** layer, not the implementation layer. Dependencies must flow inward toward the domain.

## 7. Dependencies (conditional — required when introducing any)
Exact install commands grouped by purpose (runtime / dev / tooling). Include version constraints where they matter. Otherwise: "No new dependencies required."

## 8. Configuration (conditional — required for new env vars or config keys)
Table: variable | config key | default | required | description.
Each new optional capability has an `_ENABLED`-style toggle defaulting to off. When off, no infrastructure is initialized and no connections are opened.

## 9. File structure (required)
A tree showing only:
- new files (one-line description)
- modified files (`← Modified: <reason>`)

Do not list files unaffected by this work.

## 10. Implementation outline (required)
Pattern-critical code, not full implementations. For non-trivial points, show:
- The interface or contract.
- The concrete logic that is not obvious.
- The integration site in existing code — file, function, exact insertion point.
- Comments only on the lines whose **why** is non-obvious.

Skip boilerplate that follows the project's established pattern; write `// follows the standard <pattern> pattern` and move on.

## 11. Failure modes and degradation (conditional — required for any external dependency)
For each external dependency:
- Behavior at startup if unavailable.
- Behavior at runtime if it fails.
- Whether the user-facing surface degrades or fails loudly.
- If the dependency is critical (failure = system cannot serve requests), state so and justify.

## 12. Wiring / Bootstrap (conditional — backend / service code)
Position in the existing startup sequence and the reason. Shutdown ordering and the reason. Reference the project's documented wiring sequence if one exists.

## 13. Infrastructure (conditional — new external services or manifests)
Configuration must include healthchecks or readiness criteria, resource limits, restart policy, and secrets handling. State exactly which files are added or modified (compose, k8s, terraform).

## 14. UX / Accessibility (conditional — user-facing UI changes)
- Component placement, all states (loading / empty / error / success).
- Keyboard, screen-reader, and contrast behavior.
- Telemetry on user-visible interactions.

## 15. Public API / SDK impact (conditional — libraries, SDKs, public APIs)
- Backward compatibility, semver impact (MAJOR / MINOR / PATCH).
- Deprecations and timelines.
- Migration guide for callers.

## 16. Data and migrations (conditional — schema or data changes)
- Schema delta, migration script location, reversibility.
- Backfill plan: idempotency, batch size, throttling.
- Read/write compatibility window during the migration.

## 17. Security and privacy (conditional — auth, PII, secrets, external input)
- New attack surface introduced.
- AuthN / AuthZ changes.
- Data classification, retention, redaction.
- Input validation boundary — where untrusted data becomes trusted.

## 18. Observability (conditional — recommended for production code)
- Logs: events, levels, structured fields.
- Metrics: names, units, labels.
- Traces: spans of interest.
- Alerts and ownership.

## 19. Performance (conditional — when latency, throughput, or cost matters)
- Targets: p50 / p95 / p99 latency, throughput, memory, cost.
- Load characteristics and capacity assumptions.

## 20. Testing (required)
A scenario table with at least five rows. Cover: happy path, dependency unavailable, idempotency / repeat input, invalid input, cancellation / timeout, and any feature-specific concern.

| Scenario | Setup | Expected behavior | Requirement |
|---|---|---|---|
| Happy path | ... | ... | R1 |

State unit / integration / contract test placement, what is mocked, what runs against real infrastructure, and the command to run them. State any coverage target the project has.

## 21. Rollout and rollback (conditional — staged or risky changes)
- Flag name and default.
- Phased rollout plan (cohorts, % traffic).
- Rollback procedure — concrete: a command, a flag flip, a redeploy.
- Removal trigger for the flag and its target date.

## 22. Tasks (required)
Numbered, dependency-ordered. Mark groups that can run in parallel. Each task back-references one or more requirement IDs.

- T1 [parallel-A] [R1] — ...
- T2 [parallel-A] [R2] — ...
- T3 [after T1, T2] [R3] — ...

## 23. Definition of Done (required)
A checklist where every item is independently verifiable — a command passes, a file exists, an observable behavior is met. Group by layer (contracts, implementation, integration, tests, observability) when useful.

- [ ] R1 verified by <test / command / observation>
- [ ] R2 ...
```

## Quality bar for code examples in specs

- **Real names from the project.** No `SomeService`, `MyRepo`, generic `handler`. If you cannot fill in real names, you didn't read the codebase.
- **Pattern, not full implementation.** Show the 15 lines that contain the pattern, not the 60 with boilerplate. Write "remainder follows the standard pattern" for the rest.
- **Comment the why, not the what.** A future reader can read the code; they need the constraint, the trade-off, the reason.
- **Match the project's error-handling convention.** Read two or three existing call sites before drafting examples.
- **Show both failure modes** when degradation matters: a wrong example (propagates failure to the caller) and a correct one (degrades gracefully).

## Patterns to avoid in specs

These produce specs that read well but don't ship:

- Collaborative voice (`we will use X`) instead of imperative (`use X`).
- "Can be extended later" without a concrete trigger or owner.
- "See library docs" as the only guidance for a non-trivial decision.
- Abstractions defined at the implementation layer instead of the consumer.
- A test plan whose only scenario is the happy path.
- A configuration flag without a documented default.
- An external service without a healthcheck or readiness criterion.
- TODO comments without a specific trigger or owner.
- Errors silently swallowed without a documented reason.
- Definition-of-Done items that cannot be objectively verified.
- Placeholder identifiers (`SomeService`, `MyRepository`, generic `Handler`).
- Strawman alternatives — listed only to be dismissed; not real options anyone would have considered.

## Working with multiple specs

A large feature usually wants a **graph of focused specs** rather than one monolith. Decomposition rule: two specs can be drafted in parallel if and only if their **File structure** sections do not share modified files.

When the feature is broad, before drafting any spec, produce a decomposition plan:

1. List sub-components, each as a candidate spec.
2. For each, list the files it modifies.
3. Group specs that share zero modified files (parallel).
4. Order the rest by dependency.

Then draft parallel specs concurrently, dependent specs once their inputs are accepted.
