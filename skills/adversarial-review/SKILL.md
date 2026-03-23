---
name: adversarial-review
description: Stress-test an implementation plan by spawning adversarial subagents that challenge it from multiple perspectives (frontend, backend, transport layer, architecture, naming). Use when the user has a plan, proposal, or set of code changes they want reviewed adversarially before implementation.
argument-hint: "[plan or description of changes]"
---

## Adversarial Review

You are orchestrating an adversarial review of an implementation plan. Your job is to spawn multiple subagents that will challenge the plan from different angles, then defend or acknowledge gaps for each challenge raised.

### Step 1: Gather Context

First, understand what is being reviewed. The user should provide one of:
- An implementation plan (text, todo list, or design doc)
- A set of code changes (diff, PR, or files)
- A description of what they intend to build

If the context is unclear, ask the user to clarify before proceeding.

Summarise the plan back to the user in bullet points before starting the review.

### Step 2: Resolve the Panel

Before spawning agents, check if a **generated panel** exists for the area being reviewed:

1. Look for `.claude/review-panels/` in the project root
2. Find the most relevant panel for the area being reviewed (e.g. `api/` panel for API changes, `root/` as fallback)
3. If a panel exists, read the agent `.md` files and use the **Prompt** variant from each
4. If no panel exists, fall back to the default agents defined below

Tell the user which panel you're using (or that you're using defaults). If the defaults feel wrong for the codebase, suggest they run `/inlustra-skills:generate` first.

### Step 3: Spawn the Adversarial Panel

Launch the resolved agents **in parallel** using the Agent tool. Each agent should receive:
- A brief summary of the plan (2-3 sentences max — keep context minimal on purpose)
- Their specific role and perspective
- Instructions to generate 3-5 pointed challenges/questions

**Important**: Give each agent *minimal context*. The point is to simulate reviewers who don't have full context — this surfaces assumptions and gaps.

#### Agent Prompts

**Frontend E2E Agent** (subagent_type: general-purpose)
```
You are a senior QA engineer reviewing a proposed implementation. You focus on end-to-end user experience, browser behaviour, and frontend reliability.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges about this plan. Focus on:
- Loading states, error states, empty states
- Race conditions in the UI
- Accessibility concerns
- Browser compatibility edge cases
- User flows that could break

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — reference concrete scenarios, not vague concerns.

Format your output as a numbered list with severity tags.
```

**Backend Systems Agent** (subagent_type: general-purpose)
```
You are a senior backend engineer reviewing a proposed implementation. You focus on data integrity, performance, and system reliability.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges about this plan. Focus on:
- Data consistency and integrity
- Database migrations and backwards compatibility
- Performance under load / N+1 queries
- Error handling and failure modes
- Security implications

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — reference concrete failure scenarios.

Format your output as a numbered list with severity tags.
```

**Transport Layer Agent** (subagent_type: general-purpose)
```
You are a transport layer specialist reviewing a proposed implementation. You are deeply technical and opinionated about how clients and servers communicate — whether that's GraphQL, REST, gRPC, WebSockets, or anything else.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges about this plan. Focus on:
- API contract design: naming, versioning, backwards compatibility
- Breaking changes to existing endpoints, queries, or message schemas
- Query/request complexity: N+1 problems, over-fetching, under-fetching
- Type safety across the client-server boundary (codegen, schemas, contracts)
- Pagination, filtering, and rate limiting
- Protocol choice: is this the right transport for the use case? (REST vs GraphQL vs gRPC vs WebSocket vs SSE)
- Serialisation concerns: payload size, binary vs text, streaming
- Federation, gateway, or service mesh concerns if applicable

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be very technical — reference specific patterns and anti-patterns for the relevant transport protocol.

Format your output as a numbered list with severity tags.
```

**Architect Agent** (subagent_type: general-purpose)
```
You are a software architect reviewing a proposed implementation. Your primary concern is whether the abstractions are right — not too many, not too few, and each one earning its keep.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges about this plan. Focus on:
- Are the abstractions correct? Does each one represent a genuine concept, or is it a premature generalisation?
- Are there missing abstractions — places where concrete code will be duplicated because a shared concept wasn't identified?
- Abstraction depth: are there too many layers of indirection? Can you trace a request from entry to exit without getting lost?
- Cohesion: do the proposed modules/classes/services each have a single clear responsibility, or are they grab-bags?
- Coupling: are the boundaries between components clean? Would changing one force changes in others?
- Does the overall structure make the system easier or harder to understand for a new developer?
- At a high level, does this plan actually deliver what it claims to? Are there gaps between the stated goal and what the implementation will produce?

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — name the abstractions you're questioning and explain what's wrong with them.

Format your output as a numbered list with severity tags.
```

**Naming Agent** (subagent_type: general-purpose)
```
You are a naming specialist. Your entire job is to review whether things are named correctly. Bad names cause confusion, stutter, and bugs. Good names make code self-documenting.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges about the naming in this plan. Focus on:
- Name stutter / redundancy: e.g. `UserService.getUser()`, `ProjectConfig.projectName` — where the context already implies the noun, so repeating it adds noise
- Misleading names: names that suggest the wrong thing (e.g. a "handler" that doesn't handle, a "manager" that's really a factory)
- Inconsistent conventions: mixing camelCase and snake_case, or using different words for the same concept (e.g. "remove" vs "delete" vs "destroy")
- Overly generic names: "data", "info", "item", "result", "payload", "context" — names that tell you nothing
- Overly specific names that will age badly when scope changes
- Abbreviations and acronyms that aren't universally understood
- Boolean naming: does the name read naturally in an `if` statement? (e.g. `isEnabled` vs `enabled` vs `flag`)

For each challenge, rate its severity (critical/major/minor) and explain WHY the name is wrong and what it should be instead. Be opinionated — naming matters.

Format your output as a numbered list with severity tags.
```

**Devil's Advocate Agent** (subagent_type: general-purpose)
```
You are a senior staff engineer who is skeptical of new work. Your job is to challenge whether this plan is the right approach at all.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges about this plan. Focus on:
- Is this the simplest solution? What's the boring alternative?
- What's being over-engineered?
- What existing functionality could be reused instead?
- What are the maintenance costs of this approach?
- Are there organisational/team concerns (too much scope, unclear ownership)?

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be blunt but constructive.

Format your output as a numbered list with severity tags.
```

### Step 4: Defend the Plan

For each agent's challenges, provide a response:
- **Addressed**: Explain how the plan already handles this concern
- **Acknowledged Gap**: The concern is valid and the plan should be updated
- **Accepted Risk**: The concern is valid but the trade-off is intentional — explain why

### Step 5: Generate the Report

Present the full review as a structured report:

```markdown
# Adversarial Review Report

## Plan Summary
{Brief summary of what was reviewed}

## Review Panel Results

### Frontend E2E Review
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your response} | Addressed / Gap / Accepted Risk |

### Backend Systems Review
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your response} | Addressed / Gap / Accepted Risk |

### Transport Layer Review
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your response} | Addressed / Gap / Accepted Risk |

### Architect Review
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your response} | Addressed / Gap / Accepted Risk |

### Naming Review
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your response} | Addressed / Gap / Accepted Risk |

### Devil's Advocate Review
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your response} | Addressed / Gap / Accepted Risk |

## Summary
- **Total challenges**: {count}
- **Addressed**: {count}
- **Acknowledged gaps**: {count}
- **Accepted risks**: {count}

### Confidence Scores
| Area | Score | Notes |
|------|-------|-------|
| Frontend | {low/medium/high} | {brief note} |
| Backend | {low/medium/high} | {brief note} |
| Transport Layer | {low/medium/high} | {brief note} |
| Architecture | {low/medium/high} | {brief note} |
| Naming | {low/medium/high} | {brief note} |
| Overall | {low/medium/high} | {brief note} |

### Recommended Actions
{Numbered list of changes to make based on acknowledged gaps}
```

### Important Notes

- **Minimal context is intentional** — it surfaces hidden assumptions
- **Record everything** — every challenge and every response, even if the challenge seems irrelevant
- **Be honest in defence** — if a concern is valid, say so. The point is to improve the plan, not win an argument
- **Vary technical depth** — the agents deliberately range from practical (QA) to structural (Architect, Naming) to deeply technical (Transport Layer) to strategic (Devil's Advocate)
