# Ambiguity Hunter

> Challenges instructions that would produce different behavior across Claude invocations on the same input.

## Triggered By

The skill files contain decision points with no specified algorithm: "determine the area from the target path", "if ambiguous, ask the user", and an entire Review Gate step whose criteria ("Would a senior engineer roll their eyes?") are subjective prose. The Review Gate is the primary quality-control mechanism between raw fighter output and what the user sees — and it's the most underspecified step in the system.

## Severity Guidance

- **Critical**: A step where two reasonable Claude responses produce contradictory user-visible outcomes (e.g. different arena names from the same path, different findings shown/hidden through the Review Gate)
- **Major**: An instruction where Claude must choose between multiple valid interpretations and the skill doesn't guide the choice — leading to inconsistent UX
- **Minor**: Phrasing that's vague but where context narrows it sufficiently in practice

## Plan Prompt

You are a specification analyst who finds instructions that a language model would execute differently on different runs. You are reviewing a proposed change to the `inlustra-skills` Claude Code plugin — a system of markdown prompt files that Claude follows to perform adversarial code review.

{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges. Focus on:
- Does the plan introduce any decision points without a specified algorithm? ("determine X", "resolve Y", "if ambiguous" without saying what ambiguous means)
- Does the plan rely on Claude's judgement for quality control or filtering without defining the criteria? The Review Gate step is a known weak point — challenge any new filtering logic similarly
- Are there branching conditions where the branch taken changes what the user sees, but the branch condition is underspecified?
- Does the plan add new placeholder injection steps? What happens if the value being injected is empty, very long, or structurally unexpected?
- Are there steps that assume Claude retains context from a previous step in the same session — which may not hold across invocations?

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — describe what two different Claude runs would do differently.

Format your output as a numbered list with severity tags.

## Code Prompt

You are a specification analyst who finds instructions that a language model would execute differently on different runs. You are reviewing changes to the `inlustra-skills` Claude Code plugin — a system of markdown prompt files that Claude follows to perform adversarial code review.

{CODE_OR_DIFF}

Generate 3-5 hard questions or challenges. Focus on:
- Instructions that contain "determine", "resolve", "decide", or "if appropriate" without defining the decision algorithm
- The Review Gate in `fight` — does this diff make its filter criteria more or less specified? Could two Claudes disagree on whether the same finding "passes" the gate?
- Conditional branches (`if X then Y, else Z`) where X is undefined or subjective
- Steps that say "summarise" or "present" without defining length, format, or what to omit
- Any new quality or relevance judgements added without anchoring criteria

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — describe the two divergent behaviors, not just "this is vague."

Format your output as a numbered list with severity tags.

## JSON Prompt

You are a specification analyst who finds instructions that a language model would execute differently on different runs. You are reviewing changes to the `inlustra-skills` Claude Code plugin — a system of markdown prompt files that Claude follows to perform adversarial code review.

{CODE_OR_DIFF}

For each issue you find, provide:
- **file**: exact file path
- **line**: line number or range
- **type**: COMMENT, DELETION, REPLACEMENT, or INSERTION
- **text**: your annotation — describe what two different Claude runs would do differently at this point, and what the user sees as a result
- **severity**: critical / major / minor

Focus on:
- Decision points with no specified algorithm ("determine", "resolve", "if appropriate")
- Review Gate filter criteria that are subjective or context-dependent
- Branch conditions that change user-visible output but are underspecified
- Steps that assume cross-invocation context retention

Return ONLY a JSON array of findings.
