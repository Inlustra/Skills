---
name: trace
description: Follow a request, event, or data flow end-to-end through the codebase. Traces the happy path, failure path, and silent divergences. Use when you need to understand how something actually moves through the system.
argument-hint: "[entry point, request, or behaviour to trace]"
---

## Trace

You are tracing a request, event, or data flow through the codebase from entry to exit. Your job is to document every hop, transformation, and decision point — and surface places where the path diverges silently.

### Step 1: Gather Context

The user should provide one of:
- An entry point (e.g. "trace a POST to /api/users")
- A behaviour (e.g. "trace what happens when a user clicks 'submit'")
- A data flow (e.g. "trace how `orderTotal` gets calculated")

Identify the starting point in the code. If ambiguous, ask.

### Step 2: Spawn Tracing Agents

Launch the following agents **in parallel**. Each traces the same flow but through a different lens.

**Happy Path Agent** (subagent_type: general-purpose)
```
You are tracing the happy path — the successful, expected flow through the system. No errors, no edge cases, everything works as intended.

The entry point is:
{ENTRY_POINT}

Read the code starting from the entry point and trace every step:
1. What function/method is called first?
2. What does it call next?
3. Where does data get transformed? What shape is it at each step?
4. What external systems are touched (DB, API, cache, queue)?
5. What is the final output/response?

For each step, document:
- File and function name
- What happens (1 sentence)
- Data shape in → data shape out
- Any side effects (writes, sends, logs)

Present as a numbered sequence. Be exhaustive — don't skip "obvious" steps.
```

**Failure Path Agent** (subagent_type: general-purpose)
```
You are tracing every failure mode in a code path. Your job is to find every place where things can go wrong and document what happens when they do.

The entry point is:
{ENTRY_POINT}

Read the code starting from the entry point and find every:
1. Try/catch block — what's caught? What happens in the catch? Is the error re-thrown, logged, or swallowed?
2. Null/undefined checks — what happens on the falsy branch?
3. Validation failures — what error is returned? What HTTP status?
4. External call failures — timeout, connection refused, unexpected response shape
5. Auth/permission failures — what gets returned? Is it distinguishable from other errors?

For each failure point, document:
- File and function name
- What triggers the failure
- What the caller sees (error type, message, status code)
- Whether the failure is recoverable or terminal
- Whether the failure is logged/reported or silent

Present as a numbered list ordered by how early in the flow they occur.
```

**Silent Divergence Agent** (subagent_type: general-purpose)
```
You are looking for places where the code path silently diverges — where something unexpected happens but no error is raised, no log is written, and the caller has no idea.

The entry point is:
{ENTRY_POINT}

Read the code starting from the entry point and find every:
1. Swallowed errors — empty catch blocks, catch-and-continue, catch-and-return-default
2. Fallback values that mask problems — `?? []`, `|| 0`, default returns that hide missing data
3. Early returns that skip important logic — guard clauses that bail out before side effects that the caller expects
4. Conditional branches where one path does significantly less than the other (feature flags, A/B tests, permission checks that silently degrade)
5. Async fire-and-forget — operations kicked off but never awaited, so failures vanish
6. Middleware/interceptors that transform the request/response in ways downstream code doesn't expect

For each divergence, document:
- File and function name
- What triggers the silent divergence
- What the caller expects to happen vs what actually happens
- How a developer would discover this divergence (or wouldn't)
- Severity (critical/major/minor) — critical means data loss or corruption could go unnoticed

Present as a numbered list.
```

### Step 3: Assemble the Trace

Combine all three agents' outputs into a single unified trace:

1. **Merge into a timeline** — interleave the happy path steps with the failure points and silent divergences that occur at each step
2. **Annotate decision points** — where the flow branches, show all paths
3. **Flag the dangerous spots** — highlight silent divergences in the final output, these are the highest-value findings

### Step 4: Generate the Report

```markdown
# Trace Report

## What was traced
{Description of the flow being traced}

## Entry Point
`{file}:{function}` — {brief description}

## Happy Path Sequence
| Step | Location | Action | Data Shape | Side Effects |
|------|----------|--------|-----------|-------------|
| 1 | `{file}:{fn}` | {what happens} | {in} → {out} | {writes/sends/none} |

## Failure Points
| Step | Location | Trigger | Caller Sees | Recoverable | Logged |
|------|----------|---------|-------------|-------------|--------|
| 1 | `{file}:{fn}` | {what fails} | {error/status} | yes/no | yes/no |

## Silent Divergences
| # | Location | Trigger | Expected | Actual | Severity |
|---|----------|---------|----------|--------|----------|
| 1 | `{file}:{fn}` | {condition} | {what caller expects} | {what actually happens} | {severity} |

## Flow Diagram
{ASCII or mermaid diagram showing the full flow with branches}

## Key Findings
{Numbered list of the most important discoveries, ordered by severity}
```

### Important Notes

- **Read the actual code** — don't infer from function names. A function called `validateUser` might not actually validate anything
- **Follow the data** — track the exact shape of data at every transformation. Many bugs live in the gaps between shapes
- **Async boundaries matter** — note every await, callback, event emission, or queue publish. These are where ordering assumptions break
- **The silent divergences are the prize** — the happy and failure paths are usually known. The silent divergences are what catch teams off guard in production
