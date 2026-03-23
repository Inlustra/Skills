# @inlustra/claude-skills

A collection of Claude Code skills for use across all your repositories, distributed as a Claude Code plugin and installable via the plugin marketplace.

## Installation

```bash
# Add the marketplace
/plugin marketplace add https://github.com/Inlustra/Skills.git

# Install the plugin
/plugin install inlustra-skills@inlustra-skills
```

## Skills

### `/inlustra-skills:adversarial-review`

An adversarial review skill that stress-tests your implementation plans before you write a single line of code.

**How it works:**

Given an implementation plan (or code changes + context), the skill spawns a panel of adversarial subagents — each with a different perspective and level of technical depth. These agents are given *minimal context on purpose* — they ask hard questions, poke holes, and surface edge cases you haven't considered.

**The adversarial panel:**

| Agent | Role | Focus |
|-------|------|-------|
| **Frontend E2E** | QA Engineer | "How will this break in the browser? What about loading states, race conditions, accessibility?" |
| **Backend Systems** | Backend Engineer | "What about data integrity, migrations, performance under load, error propagation?" |
| **Transport Layer** | API/Protocol Specialist | "API contracts, protocol choice (REST/GraphQL/gRPC/WS), N+1 queries, breaking changes, type safety across the boundary, serialisation?" |
| **Architect** | Software Architect | "Are the abstractions right? Too many layers? Missing concepts? Does this actually deliver what it claims?" |
| **Naming** | Naming Specialist | "Name stutter, misleading names, inconsistent conventions, overly generic identifiers, booleans that don't read naturally?" |
| **Devil's Advocate** | Senior Engineer | "Why build this at all? What's the simplest alternative? What are you over-engineering?" |

Each agent generates challenges at varying levels of technical depth. Your job (as the orchestrating Claude) is to defend the plan — explaining how each concern is addressed or acknowledging gaps.

**Output:**

Everything generated during the review is recorded and presented as a structured report:

- Each agent's challenges and questions
- Your responses/defences for each
- Unresolved concerns flagged for human review
- A confidence score per area (frontend, backend, API)

**Usage:**

```
/inlustra-skills:adversarial-review
```

Then provide or reference your implementation plan. The skill handles the rest.

### `/inlustra-skills:generate`

Inspect a repo (or area of a repo) and **generate a tailored review panel**. Instead of using the default agents, this skill scans your actual code, dependencies, and patterns to produce agents that catch the real bugs in your specific project.

- **Surface scan** (default): reads package.json, file tree, configs
- **Deep scan**: reads key source files, git hotspots, past bug patterns, test coverage gaps
- **Monorepo-aware**: generates separate panels per area (`api/`, `web/`, `shared/`)

Outputs markdown agent files to `.claude/review-panels/{area}/` — human-readable, editable. The `adversarial-review` and `annotate` skills automatically pick these up.

```
/inlustra-skills:generate src/api
```

### `/inlustra-skills:annotate`

Adversarial code review that outputs **inline annotations via Plannotator**. Same multi-agent approach, but instead of a report, findings are pinned to exact lines in your code — visible in the browser, interactive, dismissable. Each agent appears as a named author.

Includes a **review gate** that filters agent noise before anything reaches Plannotator. Requires the [Plannotator plugin](https://github.com/backnotprop/plannotator).

```
/inlustra-skills:annotate
```

### `/inlustra-skills:tighten`

Find and tighten loose contracts — overly broad types, unnecessary optionality, permissions wider than needed. Three agents (Types, Permissions, Optionality) scan for `any` types, always-present optional fields, exported internals, config knobs nobody turns, and null checks guarding guaranteed values.

```
/inlustra-skills:tighten src/services/auth.ts
```

### `/inlustra-skills:trace`

Follow a request, event, or data flow end-to-end through the codebase. Three agents trace the same flow through different lenses: Happy Path, Failure Path, and Silent Divergences (swallowed errors, masking fallbacks, fire-and-forget async). The silent divergences are the prize.

```
/inlustra-skills:trace "POST /api/orders"
```

### `/inlustra-skills:bun-denode`

Strip Node.js APIs, polyfills, and compat shims from a Bun project. Replaces them with native Bun equivalents (`fs.readFile` → `Bun.file()`, `dotenv` → delete, `node-fetch` → delete). Three agents: API Replacement, Dependency Purge, Config Cleanup.

```
/inlustra-skills:bun-denode all
```

## Development

```bash
# Clone the repo
git clone https://github.com/Inlustra/Skills.git
cd Skills

# Skills are plain markdown — edit directly
# Each skill lives in skills/<skill-name>/SKILL.md
```

### Repository Structure

```
Skills/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest
│   └── marketplace.json         # Marketplace definition
├── skills/
│   ├── adversarial-review/
│   │   └── SKILL.md             # Stress-test plans
│   ├── generate/
│   │   └── SKILL.md             # Generate tailored review panels
│   ├── annotate/
│   │   └── SKILL.md             # Inline annotations via Plannotator
│   ├── tighten/
│   │   └── SKILL.md             # Tighten loose contracts
│   ├── trace/
│   │   └── SKILL.md             # End-to-end flow tracing
│   └── bun-denode/
│       └── SKILL.md             # Strip Node, go native Bun
├── package.json
├── README.md
└── LICENSE
```

### Adding a New Skill

1. Create a directory under `skills/` with your skill name
2. Add a `SKILL.md` with YAML frontmatter and instructions
3. Commit and push — the marketplace picks it up automatically

## License

MIT
