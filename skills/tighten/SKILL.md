---
name: tighten
description: Find and tighten loose contracts in code — overly broad types, unnecessary optionality, permissions that are too wide, vague return types, any-typed boundaries. Use on a file, diff, or area of the codebase.
argument-hint: "[file, diff, or area to tighten]"
---

## Tighten

You are tightening loose contracts in code. Your job is to find places where types, permissions, configs, or interfaces are broader than they need to be — and make them more precise.

### Step 1: Gather Context

The user should provide one of:
- A file or set of files to review
- A diff or PR
- A description of an area of the codebase

Read the code. Understand what it actually does, not just what its types claim.

### Step 2: Spawn Tightening Agents

Launch the following agents **in parallel**. Each receives the code (or a focused excerpt) and their specific lens.

**Types Agent** (subagent_type: general-purpose)
```
You are reviewing code for loose or imprecise types. Your job is to find every place where a type is broader than what the code actually needs.

Here is the code to review:
{CODE}

Find and list every instance of:
- `any`, `unknown`, or equivalent untyped boundaries
- Union types that include cases the code never handles
- Optional fields (`?`) that are always present in practice
- Return types that are broader than what's actually returned (e.g. returns `string | null` but never returns null)
- Generic type parameters that could be constrained further
- Function parameters typed as a wide interface when only 1-2 fields are used
- Index signatures (`[key: string]: any`) hiding actual structure

For each finding, provide:
1. The exact location (file + line or function name)
2. What the type currently is
3. What it should be tightened to
4. Severity (critical/major/minor) — critical means this looseness could cause a runtime bug

Format as a numbered list.
```

**Permissions Agent** (subagent_type: general-purpose)
```
You are reviewing code for overly broad permissions and access patterns. Your job is to find every place where access is wider than necessary.

Here is the code to review:
{CODE}

Find and list every instance of:
- Exported functions/classes that are only used internally
- Public methods that should be private or protected
- Broad file/network/DB permissions when narrower ones would work
- API endpoints missing auth or with overly permissive auth
- Environment variables or config values with no validation on their shape
- Catch-all error handlers that swallow specifics
- Wildcard imports or re-exports that expose more than intended

For each finding, provide:
1. The exact location
2. What the current access level is
3. What it should be tightened to
4. Severity (critical/major/minor)

Format as a numbered list.
```

**Optionality Agent** (subagent_type: general-purpose)
```
You are reviewing code for unnecessary optionality — places where the code pretends something might not exist when in practice it always does, or where config has knobs that nobody turns.

Here is the code to review:
{CODE}

Find and list every instance of:
- Config options that only ever have one value in practice
- Feature flags that are always on (or always off)
- Null checks guarding values that are guaranteed by the call site
- Default values that are never overridden
- Parameters with defaults that every caller passes explicitly anyway
- Optional chaining (`?.`) on values that are never nullish at that point
- Fallback logic (`?? defaultValue`) protecting against cases that can't happen

For each finding, provide:
1. The exact location
2. What the current optionality is
3. Why it's unnecessary (explain what guarantees the value's presence)
4. Severity (critical/major/minor) — critical means the false optionality hides a real bug path

Format as a numbered list.
```

### Step 3: Synthesise and Prioritise

Collect all findings from the three agents. Deduplicate any overlaps. Then:

1. **Group by file** — present findings organised by location, not by agent
2. **Rank by impact** — critical findings first (looseness that could cause bugs), then major (looseness that causes confusion), then minor (cosmetic tightness)
3. **Provide the fix** — for each finding, show the exact code change. Don't just say "tighten this" — show the tightened version

### Step 4: Generate the Report

```markdown
# Tighten Report

## Summary
- **Files reviewed**: {count}
- **Total findings**: {count}
- **Critical**: {count} | **Major**: {count} | **Minor**: {count}

## Findings by File

### {file_path}
| # | Location | Category | Current | Tightened To | Severity |
|---|----------|----------|---------|-------------|----------|
| 1 | {line/function} | Type / Permission / Optionality | {current} | {proposed} | {severity} |

## Recommended Changes
{For each critical/major finding, show the exact code diff}
```

### Important Notes

- **Read the actual code** — don't guess from types alone. A type might be `string | null` but if the only call site always passes a string, the union is a lie
- **Context matters** — an `any` at a system boundary (parsing JSON from an API) might be acceptable; an `any` between two internal modules is not
- **Don't over-tighten** — if something is optional for a good reason (backwards compatibility, upcoming feature), leave it alone and note why
