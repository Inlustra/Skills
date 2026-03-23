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

### `/inlustra-skills:choose-your-fighter`

Inspect a repo (or area of a repo) and **assemble a tailored fight card**. Scans your code, dependencies, and patterns to generate fighters that know where the bugs live in your specific project.

- **Surface scan** (default): package.json, file tree, configs
- **Deep scan**: key source files, git hotspots, past bug patterns, test coverage gaps
- **Monorepo-aware**: separate arenas per area (`fighters/api/`, `fighters/web/`, `fighters/shared/`)

Presents a rationale first ("I created a Prisma fighter because you have 47 migrations"), you approve/edit, then it saves editable markdown fighter files to `.claude/fighters/{arena}/`.

```
/inlustra-skills:choose-your-fighter src/api
```

### `/inlustra-skills:fight`

Run your fighters against a plan or code. **Auto-detects** what you're fighting:

- **Fighting a plan**: challenges the idea before code exists. Fighters poke holes in your design, abstractions, naming, and approach. You defend or acknowledge gaps. Output: structured fight report.
- **Fighting code**: challenges the implementation after code exists. If [Plannotator](https://github.com/backnotprop/plannotator) is installed, findings are rendered as **inline annotations** pinned to exact lines — interactive, dismissable, author-attributed. Without Plannotator, falls back to a text report.

Automatically resolves the right arena from the file path. Includes a **review gate** that filters fighter noise — every finding that reaches you should make you stop and think.

**Default fighters** (used when no arena has been generated):

| Fighter | Fights For |
|---------|-----------|
| **Architect** | Are the abstractions right? Too many layers? Missing concepts? |
| **Naming** | Stutter, misleading names, inconsistent conventions, generic identifiers |
| **Tighten** | Loose types, unnecessary optionality, wide permissions |
| **Devil's Advocate** | Is this the simplest solution? What's over-engineered? |

```
/inlustra-skills:fight
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
│   ├── choose-your-fighter/
│   │   └── SKILL.md             # Assemble tailored fight card
│   ├── fight/
│   │   └── SKILL.md             # Run fighters against plans or code
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
