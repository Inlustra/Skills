---
name: fight
description: Run your adversarial fighters against a plan or code. Auto-detects whether you're fighting a plan (pre-implementation) or fighting code (post-implementation). Uses saved fighters from choose-your-fighter if available, otherwise uses defaults. If Plannotator is installed, annotations are rendered inline.
argument-hint: "[plan, file, diff, PR number, or area]"
---

## Fight

You are starting a fight — running adversarial agents against a plan or code to surface problems before they become bugs.

### Step 1: Detect What We're Fighting

Examine what the user provided and classify it:

**Fighting a Plan** (pre-implementation):
- The input is a text description, design doc, todo list, or RFC
- The input references things that don't exist yet ("we will create...", "the new service will...")
- The user explicitly says "plan", "proposal", "design", or "approach"

**Fighting Code** (post-implementation):
- The input is a file path, directory, or set of files
- The input is a git diff, staged changes, or branch comparison
- The input is a PR number
- The code already exists on disk

**If ambiguous**, ask the user: "Are we fighting a plan or fighting code?"

Tell the user what mode you're in: `Fighting plan` or `Fighting code`.

### Step 2: Gather the Target

**For plans:**
- Read/receive the plan content
- Summarise it back to the user in bullet points
- This summary becomes `{PLAN_SUMMARY}` for the fighters

**For code:**
- If given a file or directory: read the files
- If given a diff: capture it (`git diff`, `git diff --staged`, `git diff {branch}`)
- If given a PR number: `gh pr diff {number}`
- This becomes `{CODE_OR_DIFF}` for the fighters

### Step 3: Resolve the Arena

Find the right fighters for this fight:

1. Determine the area from the target path (e.g. `src/api/users.ts` → look for `api` arena, then `root` as fallback)
2. Look for `.claude/fighters/{arena-name}/ARENA.md` in the project root
3. If an arena exists, read the fighter `.md` files within it
4. If no arena exists, fall back to the **default fighters** below

Tell the user which arena you're using:
- `Using fighters from .claude/fighters/api/` or
- `No arena found — using default fighters. Run /inlustra-skills:choose-your-fighter to generate fighters tailored to this codebase.`

**Which prompt variant to use:**
- Fighting a plan → use the **Plan Prompt** from each fighter
- Fighting code, Plannotator available → use the **JSON Prompt** from each fighter
- Fighting code, no Plannotator → use the **Code Prompt** from each fighter

### Step 4: Launch the Fighters

Launch all fighters **in parallel** using the Agent tool. Each fighter receives:
- The target content (plan summary or code/diff)
- Their specific prompt (resolved in Step 3)
- **Minimal context** — 2-3 sentences of background max. This is deliberate.

#### Default Fighters

Used when no arena has been generated for this area.

**Architect** (subagent_type: general-purpose)

*Plan Prompt:*
```
You are a software architect reviewing a proposed implementation. Your primary concern is whether the abstractions are right — not too many, not too few, and each one earning its keep.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges. Focus on:
- Are the abstractions correct? Does each one represent a genuine concept, or is it a premature generalisation?
- Are there missing abstractions — places where concrete code will be duplicated because a shared concept wasn't identified?
- Abstraction depth: are there too many layers of indirection? Can you trace a request from entry to exit without getting lost?
- Cohesion and coupling: clean boundaries, single responsibilities
- At a high level, does this plan actually deliver what it claims to?

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific.

Format your output as a numbered list with severity tags.
```

*Code Prompt:*
```
You are a software architect reviewing code changes. Your primary concern is whether the abstractions are right.

Here is the code to review:
{CODE_OR_DIFF}

Generate 3-5 hard questions or challenges. Focus on:
- Abstractions that don't earn their keep or are premature
- Missing abstractions where duplication will creep in
- Too many layers of indirection
- Modules with mixed responsibilities
- Coupling that will make changes cascade

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — name the abstractions you're questioning.

Format your output as a numbered list with severity tags.
```

*JSON Prompt:*
```
You are a software architect reviewing code changes. Focus on abstractions — whether they're correct, missing, or premature.

Here is the code to review:
{CODE_OR_DIFF}

For each issue, return a JSON object with:
- "file": exact file path
- "line": line number or range
- "type": COMMENT, DELETION, REPLACEMENT, or INSERTION
- "text": your annotation
- "severity": critical / major / minor

Return ONLY a JSON array.
```

**Naming** (subagent_type: general-purpose)

*Plan Prompt:*
```
You are a naming specialist. Your entire job is to review whether things are named correctly.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 naming challenges. Focus on:
- Name stutter / redundancy (e.g. `UserService.getUser()`)
- Misleading names that suggest the wrong thing
- Inconsistent conventions
- Overly generic names ("data", "info", "item", "result", "payload")
- Booleans that don't read naturally in conditionals

For each challenge, rate its severity and explain WHY the name is wrong and what it should be.

Format your output as a numbered list with severity tags.
```

*Code Prompt:*
```
You are a naming specialist reviewing code changes. Bad names cause confusion, stutter, and bugs.

Here is the code to review:
{CODE_OR_DIFF}

Generate 3-5 naming challenges. Focus on:
- Name stutter / redundancy (e.g. `UserService.getUser()`, `config.configPath`)
- Misleading names that suggest the wrong behaviour
- Inconsistent conventions within the same file or module
- Overly generic names ("data", "info", "item", "result", "payload")
- Booleans that don't read naturally in conditionals

For each challenge, rate its severity and explain WHY the name is wrong and what it should be.

Format your output as a numbered list with severity tags.
```

*JSON Prompt:*
```
You are a naming specialist reviewing code changes. Find naming issues: stutter, misleading names, inconsistent conventions, overly generic identifiers, booleans that don't read naturally.

Here is the code to review:
{CODE_OR_DIFF}

For each issue, return a JSON object with:
- "file": exact file path
- "line": line number
- "type": COMMENT or REPLACEMENT
- "text": what's wrong and what it should be
- "severity": critical / major / minor

Return ONLY a JSON array.
```

**Tighten** (subagent_type: general-purpose)

*Plan Prompt:*
```
You are reviewing a plan for loose contracts — types that will be too broad, unnecessary optionality baked in from the start, interfaces wider than needed.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 challenges about precision. Focus on:
- Interfaces or data shapes that are too permissive for what they actually carry
- Unnecessary optionality that will propagate null checks everywhere
- Generic "any"-style designs where the actual domain is known and narrow
- Config or feature toggles being added that will only ever have one value

For each challenge, rate its severity and explain what should be tighter and why.

Format your output as a numbered list with severity tags.
```

*Code Prompt:*
```
You are reviewing code changes for loose contracts — overly broad types, unnecessary optionality, permissions wider than needed.

Here is the code to review:
{CODE_OR_DIFF}

Generate 3-5 challenges. Focus on:
- `any` or untyped boundaries between modules
- Optional fields that are always present in practice
- Return types broader than what's actually returned
- Exported functions that should be internal
- Catch-all error handlers that swallow specifics

For each challenge, rate its severity and explain what should be tighter and why.

Format your output as a numbered list with severity tags.
```

*JSON Prompt:*
```
You are reviewing code for loose contracts: broad types, unnecessary optionality, wide permissions.

Here is the code to review:
{CODE_OR_DIFF}

For each issue, return a JSON object with:
- "file": exact file path
- "line": line number
- "type": COMMENT or REPLACEMENT
- "text": what's too loose and what it should be
- "severity": critical / major / minor

Return ONLY a JSON array.
```

**Devil's Advocate** (subagent_type: general-purpose)

*Plan Prompt:*
```
You are a senior staff engineer who is skeptical of new work. Your job is to challenge whether this plan is the right approach at all.

Here is a brief summary of the plan:
{PLAN_SUMMARY}

Generate 3-5 challenges. Focus on:
- Is this the simplest solution? What's the boring alternative?
- What's being over-engineered?
- What existing functionality could be reused instead?
- What are the maintenance costs of this approach?
- Are there organisational/team concerns (too much scope, unclear ownership)?

For each challenge, rate its severity. Be blunt but constructive.

Format your output as a numbered list with severity tags.
```

*Code Prompt:*
```
You are a senior staff engineer who is skeptical. Your job is to challenge whether this code is the right approach.

Here is the code to review:
{CODE_OR_DIFF}

Generate 3-5 challenges. Focus on:
- Is there a simpler way to achieve the same thing?
- What's being over-engineered or abstracted prematurely?
- What existing code/library could replace this?
- What will be painful to maintain?

For each challenge, rate its severity. Be blunt but constructive.

Format your output as a numbered list with severity tags.
```

*JSON Prompt:*
```
You are a skeptical senior engineer. Challenge whether this code is the right approach. Find over-engineering, unnecessary complexity, and simpler alternatives.

Here is the code to review:
{CODE_OR_DIFF}

For each issue, return a JSON object with:
- "file": exact file path
- "line": line number
- "type": COMMENT, DELETION, or REPLACEMENT
- "text": your challenge
- "severity": critical / major / minor

Return ONLY a JSON array.
```

### Step 5: Review Gate

**Before presenting results, review every finding yourself.** Fighters produce noise. Your job is to filter.

For each finding, ask:
1. **Is this actionable?** Does it point to a concrete problem or suggest a specific change?
2. **Is this already addressed?** Does the plan or surrounding code already handle this?
3. **Is the fighter hallucinating context?** Fighters get minimal context. They sometimes invent problems about code they haven't seen.
4. **Would a senior engineer roll their eyes at this?** If yes, drop it.

**Filters:**
- **Drop** purely observational findings with no suggested action
- **Drop** findings that restate what the code/plan already says
- **Drop** minor findings on lines that already have major/critical ones
- **Merge** findings from different fighters on the same point
- **Rewrite** findings that are correct but poorly explained

### Step 6: Present Results

**For plan fights** — present a structured report:

```markdown
# Fight Report

## What Was Fought
{Brief summary}

## Mode
Plan fight — pre-implementation review

## Results by Fighter

### {Fighter Name}
| # | Challenge | Severity | Response | Status |
|---|-----------|----------|----------|--------|
| 1 | {challenge} | {severity} | {your defence} | Addressed / Gap / Accepted Risk |

## Summary
- **Total challenges**: {count} ({count} filtered out)
- **Addressed**: {count}
- **Acknowledged gaps**: {count}
- **Accepted risks**: {count}

### Confidence Scores
| Area | Score | Notes |
|------|-------|-------|
| {fighter area} | {low/medium/high} | {brief note} |

### Recommended Actions
{Numbered list of changes based on acknowledged gaps}
```

For plan fights, **defend the plan** for each challenge:
- **Addressed**: the plan already handles this
- **Acknowledged Gap**: valid concern, plan should be updated
- **Accepted Risk**: valid concern, but the trade-off is intentional

**For code fights with Plannotator** — collate JSON annotations and render through Plannotator:

1. Parse JSON arrays from each fighter
2. Tag each annotation with the fighter name as `author`
3. Deduplicate, apply review gate filters
4. Present summary, then open with `/plannotator-review` or `/plannotator-annotate`

```
## Fight Summary
- {Fighter}: {count} annotations
- Total: {count} ({critical} critical, {major} major, {minor} minor)
- Filtered out: {count}

Opening in Plannotator...
```

**For code fights without Plannotator** — present the same structured report as plan fights, but using the code prompt output instead. Suggest installing Plannotator for inline annotations.

### Step 7: Handle Feedback

**For plan fights**: if the user wants to update the plan based on gaps, help them revise. Offer to re-fight the updated plan.

**For code fights with Plannotator**: when the user sends feedback:
- **Approved annotations**: apply the changes (REPLACEMENT, DELETION, INSERTION)
- **Dismissed annotations**: acknowledge and move on
- **Modified annotations**: apply the user's version

### Important Notes

- **Minimal context is deliberate** — it surfaces hidden assumptions
- **Record everything** — every challenge and every response, even filtered ones should be counted in the summary
- **Be honest in defence** — if a concern is valid, say so. The point is to improve the plan/code, not win
- **Auto-detect should be obvious** — if it's not clear whether this is a plan or code, just ask
- **The review gate matters** — every finding that reaches the user should make them stop and think. If it doesn't clear that bar, cut it
