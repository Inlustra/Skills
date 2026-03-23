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
| **GraphQL Specialist** | API Designer | "Schema design, N+1 queries, breaking changes, resolver complexity, type safety across the boundary?" |
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
│   └── adversarial-review/
│       └── SKILL.md             # Skill definition
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
