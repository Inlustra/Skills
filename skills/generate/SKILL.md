---
name: generate
description: Inspect a repo or area of a repo and generate a tailored adversarial review panel. Produces markdown agent definitions with rationale, saved to the repo for reuse by adversarial-review and annotate skills. Use when setting up a new project or when the default review agents don't fit.
argument-hint: "[path to area, or 'all' for the whole repo]"
---

## Generate

You are generating a tailored adversarial review panel for a specific codebase or area of a codebase. Instead of using generic reviewers, you inspect the actual code, dependencies, patterns, and history to produce agents that will catch the real bugs in this specific project.

### Step 1: Determine Scope and Depth

The user provides:
- A **path** — a directory, workspace, or `all` for the whole repo
- Optionally a **depth** — `surface` (default) or `deep`

**Surface scan** reads:
- `package.json` / `bunfig.toml` / `Cargo.toml` / `go.mod` (dependencies, scripts)
- File tree structure (what directories exist, naming patterns)
- Config files (`tsconfig.json`, `.eslintrc`, `docker-compose.yml`, CI configs)
- Framework entry points (e.g. `next.config.js`, `app/layout.tsx`, `src/main.ts`)

**Deep scan** additionally reads:
- Key source files (entry points, shared modules, the "god files" that everything imports)
- Git log for hotspots (`git log --format='%H' --follow -- {path} | head -50` to find files that change most)
- Past bug patterns (`git log --grep='fix' --grep='bug' --all-match --oneline` in the target area)
- Test structure and coverage gaps (which areas have tests, which don't)
- Migration history (number and recency of schema changes)

### Step 2: Analyse and Identify Specialisms

Based on what you found, identify **which specific concerns matter for this codebase**. Don't think in roles — think in risks.

For each concern, note:
- **What signal triggered it** — e.g. "47 Prisma migration files, 3 of which were added in the last week"
- **What could go wrong** — e.g. "Schema changes at this velocity risk breaking existing queries or creating inconsistent migration chains"
- **What a reviewer should look for** — e.g. "Check that new migrations don't assume column existence before the migration runs, validate rollback paths"

Common signals and what they suggest (not exhaustive — use judgement):

| Signal | Concern |
|--------|---------|
| GraphQL schema files, resolvers | Schema design, N+1, breaking changes, type drift |
| REST routes, OpenAPI specs | Contract consistency, versioning, error response shapes |
| Prisma/Drizzle/TypeORM | Migration safety, query performance, schema drift |
| Redis/queue/pub-sub usage | Ordering guarantees, retry semantics, dead letter handling |
| Auth middleware, JWT handling | Token lifecycle, permission boundaries, refresh flows |
| Stripe/payment integrations | Idempotency, webhook reliability, state machine correctness |
| WebSocket/real-time code | Connection lifecycle, reconnection, state sync |
| SSR/RSC patterns (Next.js etc) | Server/client boundary, hydration mismatches, data fetching waterfalls |
| Monorepo workspaces | Phantom dependencies, circular imports, build ordering |
| Docker/k8s configs | Image size, secret handling, health check correctness |
| Feature flags | Stale flags, flag-dependent code paths, cleanup |
| i18n/l10n files | Missing translations, interpolation bugs, RTL handling |
| Large shared utility files | God-module risk, unclear ownership, import coupling |
| Minimal or no test files | Coverage gaps, which areas are most dangerous untested |

### Step 3: Present Rationale

Before writing any agent prompts, present your findings to the user:

```markdown
## Panel Rationale for {path}

### What I Found
{Brief summary of the codebase/area — stack, structure, key patterns}

### Proposed Agents

#### 1. {Agent Name}
**Triggered by**: {what signals you saw}
**Risk**: {what could go wrong}
**Focus**: {what this agent will look for}

#### 2. {Agent Name}
**Triggered by**: {what signals you saw}
**Risk**: {what could go wrong}
**Focus**: {what this agent will look for}

...

### Agents I Considered But Skipped
{List any concerns you noticed but decided weren't worth a dedicated agent, and why}
```

**Wait for the user to approve, modify, or reject agents before proceeding.** They might say "drop the payment agent, we're sunsetting Stripe" or "add one for accessibility, we just got audited." Adjust the panel accordingly.

### Step 4: Generate Agent Definitions

For each approved agent, write a full agent definition as a markdown file. Save them to the target repo at:

```
.claude/review-panels/{panel-name}/{agent-name}.md
```

Where `{panel-name}` is derived from the scope (e.g. `api`, `web`, `shared`, `root` for the whole repo).

Each agent file follows this format:

```markdown
# {Agent Name}

> {One-line description of what this agent challenges}

## Triggered By

{What signals in the codebase justify this agent's existence}

## Severity Guidance

- **Critical**: {what counts as critical for this agent}
- **Major**: {what counts as major}
- **Minor**: {what counts as minor}

## Prompt

You are a {role description}. You are reviewing code changes in a {brief codebase description}.

{CODE_OR_DIFF}

Generate 3-5 hard questions or challenges. Focus on:
- {focus area 1}
- {focus area 2}
- {focus area 3}
- {focus area 4}

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — reference concrete scenarios, not vague concerns.

Format your output as a numbered list with severity tags.

## JSON Prompt

You are a {role description}. You are reviewing code changes in a {brief codebase description}.

{CODE_OR_DIFF}

For each issue you find, provide:
- **file**: exact file path
- **line**: line number or range
- **type**: COMMENT, DELETION, REPLACEMENT, or INSERTION
- **text**: your annotation
- **severity**: critical / major / minor

Focus on:
- {focus area 1}
- {focus area 2}
- {focus area 3}
- {focus area 4}

Return ONLY a JSON array of findings.
```

Each agent file contains **two prompt variants**:
- **Prompt** — used by `adversarial-review` (produces text for the report)
- **JSON Prompt** — used by `annotate` (produces structured annotations for Plannotator)

### Step 5: Write the Panel Index

Create an index file at `.claude/review-panels/{panel-name}/PANEL.md`:

```markdown
# {Panel Name} Review Panel

> Generated for `{path}` on {date}

## Agents

| Agent | File | Focus |
|-------|------|-------|
| {name} | `./{agent-name}.md` | {one-line focus} |

## How This Panel Was Generated

### Scan Type
{surface / deep}

### Key Signals
{Bulleted list of what was detected}

### Skipped Concerns
{What was considered but not included, and why}

## Usage

This panel is automatically picked up by the `adversarial-review` and `annotate` skills when reviewing code in `{path}`.

To regenerate: `/inlustra-skills:generate {path}`
To edit: modify the agent `.md` files directly.
```

### Step 6: Confirm and Summarise

Tell the user:
- What files were created and where
- How to edit the panel (just edit the markdown)
- How to use it (the other skills will pick it up automatically)
- How to regenerate if the codebase evolves

### Important Notes

- **Signals, not assumptions** — every agent must be justified by something you actually found in the codebase, not by what you think a project "probably" has
- **Fewer is better** — 4 focused agents beat 8 generic ones. If you can't articulate a specific risk, don't create the agent
- **Two prompts per agent** — the text prompt for adversarial-review and the JSON prompt for annotate. Both must be present
- **Monorepo-aware** — a monorepo might have `review-panels/api/`, `review-panels/web/`, `review-panels/shared/`. Each area gets its own panel. Don't try to cover the whole monorepo with one panel unless it's genuinely homogeneous
- **The user edits these** — write them clearly. Someone who's never seen this skill should be able to read an agent file and understand what it does and why it exists
- **Regeneration is cheap** — if the codebase changes significantly (new major dependency, architectural shift), tell the user to regenerate rather than trying to make the panel future-proof
