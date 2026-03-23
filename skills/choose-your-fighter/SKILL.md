---
name: choose-your-fighter
description: Inspect a repo or area of a repo and generate a tailored adversarial review panel. Scans your code, dependencies, and patterns to produce fighters that catch the real bugs in your specific project. Run this before using fight.
argument-hint: "[path to area, or 'all' for the whole repo]"
---

## Choose Your Fighter

You are assembling a fight card — a tailored panel of adversarial agents built for a specific codebase or area. Instead of generic reviewers, you inspect the actual code to produce fighters that know where the bodies are buried in this project.

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

### Step 2: Analyse and Identify Risks

Based on what you found, identify **which specific concerns matter for this codebase**. Don't think in roles — think in risks.

For each concern, note:
- **What signal triggered it** — e.g. "47 Prisma migration files, 3 of which were added in the last week"
- **What could go wrong** — e.g. "Schema changes at this velocity risk breaking existing queries or creating inconsistent migration chains"
- **What a fighter should look for** — e.g. "Check that new migrations don't assume column existence before the migration runs, validate rollback paths"

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

### Step 3: Present the Fight Card

Before writing any prompts, present your proposed fighters to the user:

```markdown
## Fight Card for {path}

### Recon
{Brief summary of the codebase/area — stack, structure, key patterns}

### Fighters

#### 1. {Fighter Name}
**Triggered by**: {what signals you saw}
**Risk**: {what could go wrong}
**Fights for**: {what this fighter will challenge}

#### 2. {Fighter Name}
**Triggered by**: {what signals you saw}
**Risk**: {what could go wrong}
**Fights for**: {what this fighter will challenge}

...

### Considered But Cut
{List any concerns you noticed but decided weren't worth a dedicated fighter, and why}
```

**Wait for the user to approve, modify, or reject fighters before proceeding.** They might say "drop the payment fighter, we're sunsetting Stripe" or "add one for accessibility, we just got audited." Adjust accordingly.

### Step 4: Write Fighter Definitions

For each approved fighter, write a full definition as a markdown file. Save them to the target repo at:

```
.claude/fighters/{arena-name}/{fighter-name}.md
```

Where `{arena-name}` is derived from the scope (e.g. `api`, `web`, `shared`, `root` for the whole repo).

Each fighter file follows this format:

```markdown
# {Fighter Name}

> {One-line description of what this fighter challenges}

## Triggered By

{What signals in the codebase justify this fighter's existence}

## Severity Guidance

- **Critical**: {what counts as critical for this fighter}
- **Major**: {what counts as major}
- **Minor**: {what counts as minor}

## Plan Prompt

You are a {description}. You are reviewing a proposed implementation plan for a {brief codebase description}.

{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges. Focus on:
- {focus area 1}
- {focus area 2}
- {focus area 3}
- {focus area 4}

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — reference concrete scenarios, not vague concerns.

Format your output as a numbered list with severity tags.

## Code Prompt

You are a {description}. You are reviewing code changes in a {brief codebase description}.

{CODE_OR_DIFF}

Generate 3-5 hard questions or challenges. Focus on:
- {focus area 1}
- {focus area 2}
- {focus area 3}
- {focus area 4}

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — reference concrete scenarios, not vague concerns.

Format your output as a numbered list with severity tags.

## JSON Prompt

You are a {description}. You are reviewing code changes in a {brief codebase description}.

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

Each fighter file contains **three prompt variants**:
- **Plan Prompt** — used by `fight` when fighting a plan (produces text challenges)
- **Code Prompt** — used by `fight` when fighting code (produces text challenges)
- **JSON Prompt** — used by `fight` when Plannotator is available (produces structured annotations)

### Step 5: Write the Arena Index

Create an index file at `.claude/fighters/{arena-name}/ARENA.md`:

```markdown
# {Arena Name}

> Generated for `{path}` on {date}

## Fighters

| Fighter | File | Fights For |
|---------|------|-----------|
| {name} | `./{fighter-name}.md` | {one-line focus} |

## Recon

### Scan Type
{surface / deep}

### Key Signals
{Bulleted list of what was detected}

### Cut From the Card
{What was considered but not included, and why}

## Usage

Run `/inlustra-skills:fight` in this area — the fighters are picked up automatically.

To regenerate: `/inlustra-skills:choose-your-fighter {path}`
To edit: modify the fighter `.md` files directly.
```

### Step 6: Confirm

Tell the user:
- What files were created and where
- How to edit the fighters (just edit the markdown)
- How to start a fight (`/inlustra-skills:fight`)
- How to regenerate if the codebase evolves

### Important Notes

- **Signals, not assumptions** — every fighter must be justified by something you actually found in the codebase
- **Fewer is better** — 4 focused fighters beat 8 generic ones. If you can't articulate a specific risk, don't create the fighter
- **Three prompts per fighter** — plan prompt, code prompt, and JSON prompt. All three must be present
- **Monorepo-aware** — a monorepo might have `fighters/api/`, `fighters/web/`, `fighters/shared/`. Each area gets its own arena
- **The user edits these** — write them clearly. Someone who's never seen this skill should read a fighter file and immediately get it
- **Regeneration is cheap** — if the codebase changes significantly, regenerate rather than trying to future-proof
