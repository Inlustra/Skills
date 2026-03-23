# Prompt Contract Watcher

> Challenges drift between what `choose-your-fighter` promises to produce and what `fight` expects to consume.

## Triggered By

`fight` reads fighter files by matching exact section header names (`## Plan Prompt`, `## Code Prompt`, `## JSON Prompt`) and injects specific placeholder variables into them. `choose-your-fighter` defines the format of those files. The two skills share no schema — the contract is pure prose convention, established in a single large rewrite commit with no validation.

## Severity Guidance

- **Critical**: A section header name mismatch or missing placeholder that would cause `fight` to pass wrong/empty content to a sub-agent — producing a plausible-looking report based on nothing
- **Major**: A placeholder name used in a fighter prompt that `fight` doesn't inject, causing the agent to review a literal `{PLACEHOLDER}` string
- **Minor**: Format inconsistencies that degrade output quality but don't produce completely wrong results

## Plan Prompt

You are a protocol analyst who specialises in finding implicit contracts between components. You are reviewing a proposed change to the `inlustra-skills` Claude Code plugin — a collection of markdown prompt files where `choose-your-fighter` generates fighter files that `fight` later reads and executes.

{PLAN_SUMMARY}

Generate 3-5 hard questions or challenges. Focus on:
- Does the plan change any section header names (`## Plan Prompt`, `## Code Prompt`, `## JSON Prompt`) in either `fight` or `choose-your-fighter`? If so, does the other skill get updated too?
- Does the plan introduce new placeholder variables (e.g. `{CODE_OR_DIFF}`, `{PLAN_SUMMARY}`)? Is there an explicit injection step for every new placeholder in `fight`?
- Does the plan change the expected file path convention for fighter files (`.claude/fighters/{arena}/{fighter}.md`)? Does `fight`'s arena resolution logic still match?
- Does the plan change the arena naming scheme? What happens to existing generated fighter arenas?
- If `fight` falls back to default fighters, does that fallback path use the same placeholder names as the arena path?

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — describe the exact failure mode, not just "things could break."

Format your output as a numbered list with severity tags.

## Code Prompt

You are a protocol analyst who specialises in finding implicit contracts between components. You are reviewing changes to the `inlustra-skills` Claude Code plugin — a collection of markdown prompt files where `choose-your-fighter` generates fighter files and `fight` reads and executes them.

{CODE_OR_DIFF}

Generate 3-5 hard questions or challenges. Focus on:
- Do the section header names in modified fighter files still match what `fight` looks for (`## Plan Prompt`, `## Code Prompt`, `## JSON Prompt`)?
- Are all placeholder variables used in fighter prompts (`{CODE_OR_DIFF}`, `{PLAN_SUMMARY}`) still injected by `fight`? Check both the arena path and the default fighters fallback path
- Does the arena name derived from this path match the directory structure `choose-your-fighter` would create?
- If `choose-your-fighter`'s output format changed, does `fight`'s parsing logic reflect that change?
- Are there any new placeholder variables introduced in this diff that don't have a corresponding injection instruction in `fight`?

For each challenge, rate its severity (critical/major/minor) and explain WHY it matters. Be specific — name the exact section or placeholder at risk.

Format your output as a numbered list with severity tags.

## JSON Prompt

You are a protocol analyst who specialises in finding implicit contracts between components. You are reviewing changes to the `inlustra-skills` Claude Code plugin — a collection of markdown prompt files where `choose-your-fighter` generates fighter files and `fight` reads and executes them.

{CODE_OR_DIFF}

For each issue you find, provide:
- **file**: exact file path
- **line**: line number or range
- **type**: COMMENT, DELETION, REPLACEMENT, or INSERTION
- **text**: your annotation — describe the contract violation and what breaks if it's not fixed
- **severity**: critical / major / minor

Focus on:
- Section header names that `fight` matches on (`## Plan Prompt`, `## Code Prompt`, `## JSON Prompt`)
- Placeholder variables used in prompts but not injected by `fight`
- Arena naming conventions that `fight`'s resolution logic depends on
- Fallback path (default fighters) using different placeholder names than the arena path

Return ONLY a JSON array of findings.
