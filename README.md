# spec-writer

> A portable, language-agnostic skill that turns "build feature X" into an implementation spec an AI coding agent can actually execute — grounded in your project, with explicit goals, non-goals, decisions, alternatives considered, testable requirements, and a verifiable definition of done.

---

## The problem

You ask an agent to write a spec. It writes one. It just never read the codebase first.

The result is always the same: placeholder identifiers (`SomeService`, `MyRepository`), generic justifications ("Redis is widely adopted"), missing non-goals, and a Definition of Done that amounts to a feeling — *"the feature works."*

Then come the mid-task questions, the wrong wiring, the ignored edge cases, and a PR that doesn't match the spec.

**spec-writer** fixes this by making the agent follow a five-phase process where reading the project, surfacing open questions, and justifying every decision come *before* the first line of spec text.

---

## What it is

A single `SKILL.md` file. The body is plain Markdown that any AI coding tool can load as instructions — Claude Code, Cursor, Windsurf, GitHub Copilot, Aider, Codex, Continue, and similar systems. The frontmatter is consumed by Claude Code for routing; other tools ignore unknown YAML harmlessly.

The skill is informed by:

- **GitHub Spec Kit** — the constitution / specify / plan / tasks pattern, with `[NEEDS CLARIFICATION]` markers
- **AWS Kiro** — EARS notation for testable requirements, requirement-to-task traceability
- **ADRs** (Michael Nygard) — Context / Decision / Consequences with rejected alternatives
- **Google design doc culture** — explicit non-goals, alternatives considered, cross-cutting concerns
- **Shape Up** (Basecamp) — fat-marker sketches, scope budgets, no-gos
- **Anthropic Agent Skills format** — frontmatter, progressive disclosure, "explain the why" over rigid MUST walls
- **RFC 2119 + Rust RFCs + Python PEPs** — normative keywords, rejected ideas, two-level explanation

---

## Compatibility

Any tool that loads instruction Markdown:

| Tool | How it picks it up |
|---|---|
| Claude Code | Place at project root or under `.claude/skills/spec-writer/SKILL.md`. The frontmatter routes activation. |
| Cursor | Drop into `.cursor/rules/spec-writer.mdc` (or `.cursorrules` for the legacy format). |
| Windsurf | Drop into `.windsurf/rules/`. |
| GitHub Copilot Chat | Reference in the project's `.github/copilot-instructions.md`. |
| Aider | `aider --read SKILL.md` |
| Codex / Continue / others | Any tool that lets you load a project-level prompt. |

**Language-agnostic.** Works for Go, TypeScript, Python, Rust, Java, Kotlin, Swift, C#, Ruby, PHP, Elixir, ML pipelines, infra, mobile, frontend — any stack.

---

## Installation

### Manual (recommended for portability)

Copy `SKILL.md` to wherever your agent loads instructions from. Some common targets:

```
.claude/skills/spec-writer/SKILL.md
.cursor/rules/spec-writer.mdc
.windsurf/rules/spec-writer.md
docs/SKILL.md  # generic location for any tool
```

### Project root

Drop `SKILL.md` at the repo root. Reference it from your `AGENTS.md` / `CLAUDE.md` / `.cursorrules` so the agent knows it exists and when to use it.

---

## How to use

### Activation phrase

Any of these will trigger the skill if loaded:

```
use the spec-writer skill to create a spec for ...
write a spec for ...
spec out ...
plan the implementation of ...
```

### Three prompt levels

The difference between levels is **what you've already decided vs. what's still open** — not length.

#### Level 1 — Minimal

Use when the project has a well-documented `AGENTS.md` / `CLAUDE.md`.

```
Use the spec-writer skill to create a spec for
rate limiting on the API.
```

The agent reads the primary context document and infers constraints.

#### Level 2 — Standard *(recommended)*

Use when the main decisions are made.

```
Use the spec-writer skill to create a spec for
rate limiting on the API.

Context:
- Token bucket, per IP and per authenticated user
- Redis in production (already in the stack), in-memory fallback
- Apply only to /api/v1/*, not /health or /metrics
- 100 req/min unauthenticated, 600 req/min authenticated
- Client receives 429 with Retry-After header

Output: specs/rate-limiting/SPEC.md
```

#### Level 3 — Complete

Use when there are open decisions or ambiguous scope.

```
Use the spec-writer skill to create a spec for
rate limiting on the API.

Decided:
- Token bucket, per IP and per user (JWT claims)
- Redis already available in the stack
- Only /api/v1/*, not /health /swagger /metrics
- 429 with Retry-After required

Still undecided — surface trade-offs in the spec:
- Sliding window vs fixed window within token bucket
- Global state vs per-route state

What this is NOT:
- Not DDoS protection / WAF
- Not granular per-endpoint limits (future)

Output: specs/rate-limiting/SPEC.md
```

What's open is marked `[NEEDS CLARIFICATION]` in the resulting spec, not silently resolved.

---

## The five-phase process

When the skill is activated, the agent follows these phases. None can be skipped.

### Phase 1 — Investigate the project

Read primary context (`AGENTS.md` / `CLAUDE.md` / `.cursorrules` / `README.md` — first match), the dependency manifest, the entry point, the environment contract, and at least three files in the area being modified.

Detect the **feature class**: backend service, frontend / UI, library / SDK, CLI / script, data / pipeline / ML, or refactor. The class determines which sections of the spec template apply.

### Phase 2 — Clarify before drafting

Identify what's decided, what's assumed, what's unknown. Anything still open is marked inline as `[NEEDS CLARIFICATION: ...]`. The spec is not "ready" while any clarification marker remains.

### Phase 3 — Justify the approach

For every material decision: why this for **this project** (referencing real constraints), and why not the obvious alternative (one sentence, real trade-off). Generic justifications are rejected.

### Phase 4 — Draft the spec

Use the template (below). Include only sections that apply to the feature class; mark omitted optional sections `N/A — <reason>`.

Requirements get IDs (`R1`, `R2`, …) and are written as testable statements — EARS patterns where useful. Tasks back-reference requirement IDs, so traceability is one grep away.

### Phase 5 — Self-check

Ten questions before saving — read the project, real names, alternatives named, non-goals explicit, criteria testable, DoD verifiable, open questions surfaced, failure modes specified, rollback documented, right-sized.

---

## Spec template — what gets generated

23 sections, **conditional by feature class**. Required sections always appear; conditional sections appear only when relevant — otherwise a one-line `N/A — <reason>` is left so reviewers know it was considered.

| # | Section | Status |
|---|---|---|
| 1 | Summary | required |
| 2 | Goals and Non-goals | required |
| 3 | Open questions | required while non-empty |
| 4 | Requirements (EARS-style, with IDs) | required |
| 5 | Decisions and Alternatives Considered | required |
| 6 | Design | required |
| 7 | Dependencies | conditional |
| 8 | Configuration | conditional |
| 9 | File structure | required |
| 10 | Implementation outline | required |
| 11 | Failure modes and degradation | conditional (any external dep) |
| 12 | Wiring / Bootstrap | conditional (backend / service) |
| 13 | Infrastructure | conditional (new manifests) |
| 14 | UX / Accessibility | conditional (UI) |
| 15 | Public API / SDK impact | conditional (libraries) |
| 16 | Data and migrations | conditional (schema / data) |
| 17 | Security and privacy | conditional (auth / PII / external input) |
| 18 | Observability | conditional (production code) |
| 19 | Performance | conditional (latency / throughput / cost matters) |
| 20 | Testing | required |
| 21 | Rollout and rollback | conditional (staged / risky) |
| 22 | Tasks (with R-ID back-references) | required |
| 23 | Definition of Done | required |

Sections **5, 10, and 23** are where specs typically fail. They are the most controlled by the skill.

---

## Output

Default location: `specs/<feature-name>/SPEC.md` — lowercase, hyphenated, one spec per directory. The skill creates `specs/` if absent.

If `specs/README.md` does not exist, an index is created listing each spec with: name | status (`draft` / `ready` / `in progress` / `done`) | dependencies | primary files touched.

---

## What a good spec looks like (vs a bad one)

### Definition of Done

Rejected:

```
- [ ] The feature works
- [ ] Tests pass
```

Accepted:

```
**Contracts**
- [ ] R1 verified: rate limit counter increments per IP on successful request
- [ ] R2 verified: 429 response includes Retry-After header

**Implementation**
- [ ] Limiter abstraction defined at consumer layer (api/middleware), not infra
- [ ] In-memory fallback used when REDIS_URL is empty; no startup failure

**Integration**
- [ ] /health and /metrics excluded from limiter (verified via integration test)
- [ ] FEATURE_RATE_LIMITING_ENABLED=false → zero infra initialized, no metrics emitted

**Tests**
- [ ] All scenarios from §20 pass under race detection / concurrent execution
- [ ] Integration test verifies degradation when Redis is unavailable
```

### Decisions section

Rejected:

> We will use Redis because it's the most popular caching layer.

Accepted:

> **D2 — Storage backend for the limiter.**
> **Chosen:** Redis with INCR + PEXPIRE.
> **Why this for this project:** Redis is already in the stack for session storage (`platform/cache/`) and the rate-limit data has the same lifecycle properties. Reuses the existing connection pool defined at `platform/cache/pool.go`.
> **Alternatives considered:**
> - In-memory only — rejected: doesn't survive process restart and doesn't share state across replicas, both of which break the per-user limit.
> - PostgreSQL row-level — rejected: row contention under spike traffic is the exact failure mode rate limiting is meant to absorb.
> **Consequences:** Adds a hard dependency on Redis for one more code path. Mitigated by an in-memory fallback (R5) when `REDIS_URL` is empty.

---

## Plan mode vs direct execution

The skill works in both modes.

| Situation | Plan first | Direct |
|---|---|---|
| New feature, ambiguous scope | ✅ | ⚠️ may need rewrite |
| New integration (Redis, Kafka, gRPC) | ✅ | ✅ with Level 2+ prompt |
| Well-documented project (`AGENTS.md`) | optional | ✅ preferred |
| Open architectural decision in spec | ✅ | ⚠️ may miss the trade-off |
| Tight deadline, clear constraints | ⚠️ extra round-trip | ✅ |
| Agent unfamiliar with the codebase | ✅ | ❌ risk of placeholders |

In Claude Code: `Shift+Tab` before submitting (or `/plan`). The agent surfaces an outline first; no files are created until you approve.

---

## Orchestrating large work — the spec graph

A large feature usually wants a **graph of focused specs** rather than one monolith. The decomposition rule: two specs can be drafted in parallel if and only if their **File structure** sections do not share modified files.

Before drafting any spec for broad work:

```
Read the AGENTS.md and the current specs/ directory.

I need to implement a multi-tenant billing system. Before writing any spec,
produce a decomposition plan:

1. List each sub-component that needs its own spec.
2. For each: state which existing files it modifies.
3. Draw the dependency graph.
4. Group specs that can be written in parallel (no shared file modifications).

Do not write any spec yet.
```

Then run parallel specs concurrently, dependent specs once their inputs are accepted.

### Anti-pattern: the monolith spec

A single spec covering subscriptions, payments, invoicing, quota enforcement, and the admin dashboard is too long to verify section by section, impossible to parallelize, and full of unresolved cross-cutting decisions. Break it down — always. Smaller specs are more precise, easier to review, and faster to implement.

---

## What the skill does **not** do

- **Does not replace the developer's judgment.** Wrong constraints in → wrong spec out, just better formatted.
- **Is not a code-generation framework.** It instructs the agent on the *process* of speccing. Implementation is still the agent's, guided by the spec.
- **Is not suitable for tactical tasks.** Bug fixes, local refactors, config tweaks — the overhead is disproportionate.
- **Does not work without project context.** The better the `AGENTS.md` / `CLAUDE.md`, the better the spec. The skill amplifies what already exists; it doesn't invent.

---

## A good `AGENTS.md` makes specs better

The skill works best when the project has a well-documented primary context document (`AGENTS.md`, `CLAUDE.md`, or `.cursor/rules/`). It is read in Phase 1 and feeds the entire spec.

A good context doc has:

- **File map** — every significant file with a one-line description of its responsibility.
- **Architecture rules** — non-negotiable constraints (e.g., "interfaces at the consumer, never at the infra layer").
- **Wiring sequence** — initialization order in the entry point.
- **Key patterns** — where each pattern lives in the code.
- **Configuration reference** — environment variables with defaults.

Without it, the agent infers from code — which works, but is slower and less precise.

---

## Anti-patterns the skill rejects

If any of these appear in the generated spec, it is rewritten before delivery:

| Anti-pattern | Why it's a problem |
|---|---|
| `"We will use X"` | Collaborative, not imperative — write `"Use X"`. |
| `"Can be extended later"` without trigger | No concrete trigger = immediate tech debt. |
| Interface defined in the infra layer | Violates dependency inversion — infra doesn't define contracts. |
| `"Test the happy path"` only | Unhappy paths are where bugs live. |
| Env var without a documented default | Ambiguity at deploy = incident in production. |
| External service without a healthcheck | Unverifiable dependency = startup race. |
| `// TODO: implement` in spec code blocks | TODOs without owners become permanent debt. |
| Placeholder identifiers (`SomeService`, `MyRepo`) | Proof the agent didn't read the code. |
| Strawman alternatives | Listed only to be dismissed; not real options. |
| DoD items that can't be objectively verified | A feeling, not a check. |

---

## Repository structure

```
spec-writer/
├── SKILL.md      ← The skill — install this
├── EXAMPLES.md   ← Copy-paste prompts for common scenarios
├── README.md     ← This file
└── LICENSE
```

Intentionally simple. One skill, one file, zero dependencies.

For copy-paste prompts covering all feature classes, prompt levels, and workflows (decomposition, plan mode, retrofit, draft revision), see [EXAMPLES.md](EXAMPLES.md).

---

## Contributing

Issues and PRs welcome, especially for:

- Worked examples in non-backend domains (mobile, embedded, ML, data engineering).
- Refinements to phase 2 (clarification batching) for ambiguous prompts.
- A documented gallery of negative examples — specs that failed and why.

---

## License

MIT
