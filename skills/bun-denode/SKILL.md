---
name: bun-denode
description: Find and replace Node.js APIs, polyfills, and compatibility shims with their native Bun equivalents. Use on a file, directory, or entire project to strip out Node overhead and go native.
argument-hint: "[file, directory, or 'all' for the whole project]"
---

## Denode

You are removing Node.js from this code. Every `fs.readFile` that should be `Bun.file()`, every `http.createServer` that should be `Bun.serve()`, every `node:` import that has a faster Bun-native equivalent — find it and replace it.

### Step 1: Gather Context

The user should provide one of:
- A file or set of files to denode
- A directory to scan
- `all` to scan the entire project

Read the code. Identify the runtime target (check `package.json`, `bunfig.toml`, `tsconfig.json`) to confirm this is a Bun project or is intended to become one.

If the project still needs to support Node as a target (dual runtime), **stop and tell the user** — denoding is destructive to Node compatibility and they should confirm before proceeding.

### Step 2: Spawn Denoding Agents

Launch the following agents **in parallel**.

**API Replacement Agent** (subagent_type: general-purpose)
```
You are scanning code for Node.js APIs that have native Bun replacements. Your job is to find every instance where the code goes through Node's API (or Bun's Node compatibility layer) when a faster, native Bun API exists.

Here is the code to review:
{CODE}

Find every instance of:
- `fs.readFile` / `fs.readFileSync` → `Bun.file().text()` / `Bun.file().arrayBuffer()`
- `fs.writeFile` / `fs.writeFileSync` → `Bun.write()`
- `fs.existsSync` / `fs.access` → `Bun.file().exists()`
- `fs.readdir` → `Bun.Glob` where appropriate
- `http.createServer` / `https.createServer` → `Bun.serve()`
- `express()` / `fastify()` / `koa()` patterns that could be `Bun.serve()` with `Bun.route()` or a lighter framework like Hono/Elysia
- `child_process.exec` / `spawn` → `Bun.spawn()` / `Bun.$`
- `path.join` / `path.resolve` → template literals or `Bun.pathToFileURL` where appropriate
- `crypto.randomUUID` → `Bun.randomUUIDv7()`
- `crypto.createHash` → `Bun.CryptoHasher`
- `Buffer.from` → direct use of `Uint8Array` or Bun's optimised Buffer
- `node:stream` / `Readable` / `Writable` → Web Streams API or `Bun.ArrayBufferSink`
- `node:worker_threads` → `new Worker()` (Bun's native workers)
- `dotenv` / `dotenv.config()` → remove entirely (Bun reads `.env` natively)
- `cross-env` → remove entirely (Bun handles env cross-platform)
- `node-fetch` / `undici` → remove entirely (Bun's native `fetch`)
- `ws` WebSocket library → `Bun.serve()` with `websocket` handler
- `jest` / `vitest` imports → `bun:test`
- Any `node:` prefixed import where Bun has a native equivalent

For each finding, provide:
1. File and line/function
2. Current Node API usage
3. The exact Bun-native replacement code
4. Performance note (why the Bun version is better — fewer allocations, native implementation, no compat overhead, etc.)

Format as a numbered list.
```

**Dependency Purge Agent** (subagent_type: general-purpose)
```
You are scanning a project's dependencies for npm packages that exist only because Node needed them — packages that Bun makes redundant.

Here is the package.json (and lock file if available):
{PACKAGE_JSON}

Here is the code that imports these dependencies:
{CODE}

Find every dependency that can be removed because:
- Bun has the functionality built in (e.g. `dotenv`, `cross-env`, `node-fetch`, `ws`, `jest`, `vitest`, `tsx`, `ts-node`, `nodemon`)
- The package is a polyfill for something Bun supports natively (e.g. `whatwg-url`, `abort-controller`, `web-streams-polyfill`, `formdata-polyfill`)
- The package wraps a Node API that Bun replaces (e.g. `glob`, `rimraf`, `mkdirp`, `which`)
- The package is a Node-specific build tool Bun replaces (e.g. `webpack`, `esbuild`, `rollup` when Bun's bundler suffices)

For each dependency:
1. Package name and current version
2. Why it's redundant in Bun
3. What replaces it (built-in API, native feature, or nothing — just delete)
4. Files that import it (so the user knows what to update)
5. Confidence level (certain / likely / check first) — "check first" means the package might be used for features beyond what Bun replaces

Format as a numbered list, grouped by confidence level.
```

**Config Cleanup Agent** (subagent_type: general-purpose)
```
You are scanning project configuration for Node-specific settings that should be updated or removed for a Bun project.

Here are the config files:
{CONFIG_FILES}

Find every config artefact that needs to change:
- `tsconfig.json`: module resolution, target, types that should include `bun-types`
- `package.json` scripts: `node` calls that should be `bun`, `npx` that should be `bunx`, `ts-node` / `tsx` wrappers that are unnecessary (Bun runs TypeScript natively)
- `.env` loading: any programmatic dotenv setup (Bun loads `.env` automatically)
- `nodemon.json` / watch configs: Bun has `--watch` and `--hot` built in
- Jest/Vitest config files that should be removed in favour of `bun test`
- Dockerfile: `FROM node:` images that should be `FROM oven/bun:`
- CI/CD: Node setup actions that should be Bun setup actions
- `.nvmrc` / `.node-version`: should these become a `bun` version pin?
- `engines` field in package.json

For each finding:
1. File and setting
2. What it currently is
3. What it should be (or "remove")
4. Why

Format as a numbered list.
```

### Step 3: Synthesise

Collect all findings. Then:

1. **Order by safety** — removals that are drop-in safe first (e.g. removing `dotenv`), risky changes last (e.g. replacing an Express app with `Bun.serve()`)
2. **Group by action type** — API swaps, dependency removals, config changes
3. **Flag breaking changes** — anything that changes external behaviour (different error types, different streaming semantics, etc.)

### Step 4: Generate the Report

```markdown
# Denode Report

## Project
{project name, detected runtime, key config}

## Summary
- **API replacements**: {count}
- **Dependencies to remove**: {count}
- **Config changes**: {count}

## Safe to Do Now (drop-in replacements)

### API Swaps
| # | File | Node API | Bun Replacement | Performance Note |
|---|------|----------|-----------------|-----------------|
| 1 | {file} | {node api} | {bun api} | {why faster} |

### Dependencies to Remove
| # | Package | Reason | Replaced By | Confidence |
|---|---------|--------|-------------|------------|
| 1 | {pkg} | {why redundant} | {bun built-in} | certain/likely |

### Config Changes
| # | File | Setting | Change | Reason |
|---|------|---------|--------|--------|
| 1 | {file} | {setting} | {what to do} | {why} |

## Needs Review (behaviour may change)
{List of changes that alter external behaviour, with explanation of what's different}

## Migration Commands
{Exact shell commands to run, in order}
```

### Important Notes

- **Don't assume Bun parity** — if a Node API has no Bun equivalent yet, say so. Don't force a replacement that doesn't exist
- **Check Bun's compatibility table** — some `node:` modules work fine through Bun's compat layer and aren't worth replacing if there's no native gain
- **Order matters** — dependency removal should happen after API replacement, not before
- **Test after each batch** — suggest the user runs `bun test` between groups of changes, not all at once
