---
name: annotate
description: Spawn adversarial agents that review code and leave inline annotations via Plannotator. Each agent appears as a named author on the actual lines they're challenging. Use on files, diffs, or PRs to get interactive, visual code review from multiple perspectives.
argument-hint: "[file, diff, PR number, or area to annotate]"
---

## Annotate

You are running an adversarial code review that produces **inline annotations** on real code, rendered through Plannotator. Instead of a markdown report, each agent's findings become comments pinned to the exact lines they're challenging — visible in the browser, interactive, dismissable.

### Prerequisites

This skill requires the Plannotator plugin to be installed:
```
/plugin marketplace add backnotprop/plannotator
/plugin install plannotator@plannotator
```

If Plannotator is not installed, tell the user and stop.

### Step 1: Gather the Review Target

The user should provide one of:
- A file or set of files to review
- A git diff (`git diff`, `git diff --staged`, or a branch comparison)
- A PR number (use `gh pr diff {number}` to get the diff)

Read the code. Understand what changed and why before spawning agents.

### Step 2: Resolve the Panel

Before spawning agents, check if a **generated panel** exists for the area being reviewed:

1. Look for `.claude/review-panels/` in the project root
2. Find the most relevant panel for the area being reviewed (e.g. `api/` panel for API changes, `root/` as fallback)
3. If a panel exists, read the agent `.md` files and use the **JSON Prompt** variant from each
4. If no panel exists, fall back to the default agents defined below

Tell the user which panel you're using (or that you're using defaults). If the defaults feel wrong for the codebase, suggest they run `/inlustra-skills:generate` first.

### Step 3: Spawn Annotation Agents

Launch the following agents **in parallel**. Each receives the code/diff and must return findings with **exact file paths and line numbers**.

**Important**: Every finding MUST include a precise file path and line number. Findings without locations cannot become annotations and will be discarded.

**Architect Agent** (subagent_type: general-purpose)
```
You are reviewing code changes as a software architect. Your focus is abstractions — whether they're correct, missing, or premature.

Here is the code to review:
{CODE_OR_DIFF}

For each issue you find, you MUST provide:
- **file**: exact file path
- **line**: line number (or line range, e.g. "42-58")
- **type**: one of COMMENT, DELETION, REPLACEMENT, INSERTION
  - COMMENT: explain a concern but don't suggest a change
  - DELETION: this code should be removed (explain why)
  - REPLACEMENT: this code should be replaced (provide the replacement)
  - INSERTION: code is missing here (provide what should be added)
- **text**: your annotation — what's wrong and why. For REPLACEMENT, include the replacement code. For INSERTION, include the code to add.
- **severity**: critical / major / minor

Focus on:
- Abstractions that don't earn their keep
- Missing abstractions where duplication will creep in
- Layers of indirection that obscure intent
- Modules with mixed responsibilities
- Coupling that will make changes cascade

Return your findings as a JSON array:
[
  {
    "file": "src/services/auth.ts",
    "line": 42,
    "type": "COMMENT",
    "text": "This class mixes authentication and session management — these are separate concerns that will diverge.",
    "severity": "major"
  }
]

Return ONLY the JSON array, no other text.
```

**Naming Agent** (subagent_type: general-purpose)
```
You are reviewing code changes as a naming specialist. Bad names cause confusion, stutter, and bugs.

Here is the code to review:
{CODE_OR_DIFF}

For each naming issue you find, you MUST provide:
- **file**: exact file path
- **line**: line number
- **type**: one of COMMENT, REPLACEMENT
  - COMMENT: explain why the name is bad
  - REPLACEMENT: provide the better name
- **text**: what's wrong with the name and what it should be. For REPLACEMENT, include the corrected code.
- **severity**: critical / major / minor

Focus on:
- Name stutter / redundancy (e.g. `UserService.getUser()`, `config.configPath`)
- Misleading names that suggest the wrong behaviour
- Inconsistent conventions within the same file or module
- Overly generic names ("data", "info", "item", "result", "payload")
- Booleans that don't read naturally in conditionals
- Abbreviations that aren't universally understood

Return your findings as a JSON array:
[
  {
    "file": "src/services/user.ts",
    "line": 15,
    "type": "REPLACEMENT",
    "text": "Stutter: `UserService.getUser()` — the service context already implies User. Rename to `get()` or rename the service to `Users` so it reads `Users.get()`.",
    "severity": "major"
  }
]

Return ONLY the JSON array, no other text.
```

**Tighten Agent** (subagent_type: general-purpose)
```
You are reviewing code changes for loose contracts — types that are too broad, unnecessary optionality, permissions wider than needed.

Here is the code to review:
{CODE_OR_DIFF}

For each loose contract you find, you MUST provide:
- **file**: exact file path
- **line**: line number
- **type**: one of COMMENT, REPLACEMENT
  - COMMENT: explain why this is too loose
  - REPLACEMENT: provide the tightened version
- **text**: what the contract currently allows vs what it should allow. For REPLACEMENT, include the tightened code.
- **severity**: critical / major / minor

Focus on:
- `any` or untyped boundaries between modules
- Optional fields that are always present
- Return types broader than what's actually returned
- Exported functions that should be internal
- Catch-all error handlers that swallow specifics
- Config knobs that are never turned

Return your findings as a JSON array:
[
  {
    "file": "src/api/handler.ts",
    "line": 28,
    "type": "REPLACEMENT",
    "text": "Return type is `Promise<any>` but this always returns `{ id: string, status: string }`. Tighten to `Promise<OrderStatus>`.",
    "severity": "critical"
  }
]

Return ONLY the JSON array, no other text.
```

**Silent Divergence Agent** (subagent_type: general-purpose)
```
You are reviewing code changes for places where execution silently diverges from what the caller expects — swallowed errors, masking fallbacks, fire-and-forget async.

Here is the code to review:
{CODE_OR_DIFF}

For each silent divergence you find, you MUST provide:
- **file**: exact file path
- **line**: line number
- **type**: one of COMMENT, INSERTION
  - COMMENT: flag the divergence and explain the risk
  - INSERTION: suggest error handling, logging, or propagation code to add
- **text**: what the caller expects vs what actually happens. For INSERTION, include the code to add.
- **severity**: critical / major / minor

Focus on:
- Empty catch blocks or catch-and-continue
- Fallback values that hide missing data (`?? []`, `|| 0`)
- Early returns that skip side effects the caller expects
- Async fire-and-forget (operations kicked off but never awaited)
- Error types that get flattened (different failures mapped to the same generic error)

Return your findings as a JSON array:
[
  {
    "file": "src/jobs/sync.ts",
    "line": 87,
    "type": "COMMENT",
    "text": "This catch block swallows the error and returns an empty array. The caller assumes an empty result means 'no data' but it could mean 'complete failure'. At minimum, log the error. Better: let it propagate.",
    "severity": "critical"
  }
]

Return ONLY the JSON array, no other text.
```

### Step 4: Collate and Deduplicate

1. Parse the JSON arrays from each agent
2. Tag each annotation with its agent as the `author` field (e.g. "Architect", "Naming", "Tighten", "Silent Divergence")
3. Deduplicate: if two agents flag the same line for the same reason, keep the more specific one
4. Sort by file, then by line number

### Step 5: Review Gate

**Before anything goes to Plannotator, you must review every annotation yourself.** Agents produce noise — generic observations, obvious comments, things the author clearly already considered. Your job is to filter.

For each annotation, read the actual code at that line and ask:

1. **Is this actionable?** Does it point to a concrete problem or suggest a specific change? Drop vague concerns like "consider error handling here" that don't say what error or what handling.
2. **Is this already addressed?** Read the surrounding code — does the next function, the caller, or a test already handle this concern? If so, drop it.
3. **Is this worth the reader's time?** A minor style nit on a line that also has a critical abstraction issue is noise. Keep the signal.
4. **Is the agent hallucinating context?** Agents get minimal context. They sometimes invent problems based on assumptions about code they haven't seen. If the annotation references behaviour that doesn't exist, drop it.
5. **Would a senior engineer roll their eyes at this?** If yes, drop it.

**Apply these filters:**
- **Drop** any annotation that is purely observational with no suggested action
- **Drop** any annotation that restates what the code already does ("this function returns a user" — yes, we can see that)
- **Drop** any minor-severity annotation on a line that already has a major or critical one
- **Merge** annotations from different agents on the same line into one if they're making the same point
- **Rewrite** annotations that are correct but poorly explained — tighten the language, make the concern crisp

Present the filtered list to yourself as a check before proceeding. The goal is: **every annotation that reaches Plannotator should make the reader stop and think.** If it doesn't clear that bar, cut it.

### Step 6: Render Through Plannotator

Generate a markdown document containing the reviewed code with all annotations embedded, then open it with Plannotator:

1. Create a temporary markdown file containing a summary of the review and a section per file showing the code that was reviewed
2. Use `/plannotator-review` for diff-based reviews or `/plannotator-annotate` for file-based reviews
3. Each annotation should use the agent name as the author so annotations are visually attributable

Before opening Plannotator, present a quick summary to the user showing what survived the review gate:

```
## Annotation Summary
- Architect: {count} annotations
- Naming: {count} annotations
- Tighten: {count} annotations
- Silent Divergence: {count} annotations
- Total: {count} ({critical} critical, {major} major, {minor} minor)
- Filtered out: {count} (dropped during review gate)

Opening in Plannotator...
```

### Step 7: Handle Feedback

When the user sends feedback from Plannotator (approved, dismissed, or modified annotations):
- **Approved annotations**: apply the suggested changes (REPLACEMENT, DELETION, INSERTION)
- **Dismissed annotations**: acknowledge and move on
- **Modified annotations**: the user has refined the suggestion — apply their version

### Important Notes

- **Line numbers are non-negotiable** — every finding must have an exact location. Agents that return findings without line numbers get their output discarded
- **JSON output only from agents** — the agents must return parseable JSON, not markdown. This is what makes the Plannotator integration work
- **Author attribution matters** — seeing "Architect" vs "Naming" vs "Tighten" on a comment tells the user what lens the feedback is coming through
- **Don't flood** — if an agent finds more than 10 issues in a single file, it should prioritise and return only the top 10 by severity
