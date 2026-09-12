# agentmemory — Agent Instructions

## Architecture

agentmemory is a persistent memory system for AI coding agents, built on iii-engine's three primitives (Worker/Function/Trigger). Everything goes through `registerFunction`/`registerTrigger`/`sdk.trigger()` — never bypass iii-engine with standalone SQLite or in-process alternatives.

- **Engine**: iii-sdk (WebSocket to iii-engine on port 49134)
- **State**: File-based SQLite via iii-engine's StateModule (`./data/state_store.db`)
- **Build**: TypeScript → ESM via tsdown, output to `dist/`
- **Test**: vitest (`npm test` excludes integration tests)

## Consistency Rules

**When adding or removing MCP tools, you MUST update ALL of the following:**
1. `src/mcp/tools-registry.ts` — tool definition + `getAllTools()` array
2. `src/mcp/server.ts` — handler case in the `mcp::tools::call` switch
3. `src/triggers/api.ts` — REST endpoint registration
4. `src/index.ts` — function registration + endpoint count in the log line
5. `test/mcp-standalone.test.ts` — tool count assertion
6. `README.md` — tool counts (search for "MCP tools")
7. `plugin/.claude-plugin/plugin.json` — tool count in description

**When adding REST endpoints, you MUST update:**
1. `src/triggers/api.ts` — endpoint registration
2. `src/index.ts` — endpoint count in the log line
3. `README.md` — endpoint count (search for "REST endpoints" and "endpoints on port")

**When bumping version, you MUST update ALL of the following:**
1. `package.json` — version field
2. `src/version.ts` — VERSION constant and type union
3. `src/types.ts` — ExportData version union
4. `src/functions/export-import.ts` — supportedVersions set
5. `test/export-import.test.ts` — version assertion
6. `plugin/.claude-plugin/plugin.json` — version field

**When adding new KV scopes:**
1. `src/state/schema.ts` — add to the KV object
2. `src/types.ts` — add the corresponding interface

**When adding new audit operations:**
1. `src/types.ts` — add to AuditEntry.operation union type

## Code Patterns

### Function Registration
```typescript
sdk.registerFunction(
  "mem::your-function",
  async (data: { ... }) => {
    // validate inputs
    // do work via kv.get/kv.set/kv.list
    // record audit via recordAudit()
    return { success: true, ... };
  },
);
```

### REST Endpoint Registration
```typescript
sdk.registerFunction("api::your-endpoint", async (req: ApiRequest) => {
  const denied = checkAuth(req, secret);
  if (denied) return denied;
  const body = req.body as Record<string, unknown>;
  // validate + whitelist fields (never pass raw body to sdk.trigger)
  const result = await sdk.trigger({
    function_id: "mem::your-function",
    payload: { ... },
  });
  return { status_code: 200, body: result };
});
sdk.registerTrigger({
  type: "http",
  function_id: "api::your-endpoint",
  config: { api_path: "/agentmemory/your-path", http_method: "POST" },
});
```

### MCP Tool Handler
```typescript
case "memory_your_tool": {
  // validate args with typeof checks
  // parse CSV args: args.field.split(",").map(t => t.trim()).filter(Boolean)
  const result = await sdk.trigger({
    function_id: "mem::your-function",
    payload: { ... },
  });
  return { status_code: 200, body: { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] } };
}
```

### Hook Scripts
Hook scripts in `src/hooks/` are standalone Node.js scripts (no iii-sdk import). They read JSON from stdin, make HTTP calls to the REST API, and exit. Always use `try/catch` with `AbortSignal.timeout()` for best-effort calls.

## Coding Standards

- TypeScript, ESM only (`"type": "module"`)
- No code comments explaining WHAT — use clear naming instead
- Use `fingerprintId()` for content-addressable dedup, `generateId()` for unique IDs
- Parallel operations where possible (`Promise.all` for independent kv writes/reads)
- Input validation at system boundaries (MCP handlers, REST endpoints)
- REST endpoints must whitelist fields — never pass raw request body to `sdk.trigger()`
- Use `recordAudit()` for state-changing operations
- Timestamps: capture once with `new Date().toISOString()` and reuse

## Testing

- All tests must pass before PR: `npm test` (950+ tests)
- Mock pattern: `vi.mock("iii-sdk")` with mock `sdk.trigger`, `kv.get/set/list`
- Test files go in `test/` with `.test.ts` extension
- Follow existing patterns in `test/crystallize.test.ts` for function tests

## Current Stats (v0.9.16)

- 53 MCP tools (8 visible by default, `AGENTMEMORY_TOOLS=all` for all)
- 124 REST endpoints
- 6 MCP resources, 3 MCP prompts
- 12 hooks, 4 skills
- 50+ iii functions
- 950+ tests

## Local development instructions

### Services
- **iii-engine** (native binary `~/.local/bin/iii`, v0.11.2): the runtime bus on `ws://localhost:49134`; also serves REST on `:3111` and streams on `:3112`. `~/.local/bin` is on PATH in login shells.
- **agentmemory worker**: the Node process registering all `mem::*`/`api::*` functions. `iii-config.yaml` includes an `iii-exec` worker that auto-spawns the built worker (`node dist/index.mjs`), so `iii --config iii-config.yaml` runs the engine *and* worker together.
- **web viewer**: served by the worker on `:3113` (REST port + 2).

### Non-obvious setup/run caveats
- `npm install` fails with `ERESOLVE` (peer conflict: `@anthropic-ai/claude-agent-sdk` wants `@anthropic-ai/sdk>=0.93` while root pins `^0.39`). Install with `npm install --legacy-peer-deps`. The checked-in CI uses the same legacy-peer-deps option.
- You MUST run `npm run build` before starting the engine, because `iii-config.yaml`'s `iii-exec` runs `node dist/index.mjs`.
- Start the full app (engine + worker + viewer) with: `iii --config iii-config.yaml` (run from repo root; writes state to `./data/`). Then REST is at `http://127.0.0.1:3111/agentmemory/*` and the viewer at `http://127.0.0.1:3113`.
- `npm run dev` (tsx, hot reload) starts ONLY the worker and expects an engine already listening on `:49134`. Do NOT run it alongside `iii --config iii-config.yaml`, which already spawns a worker via `iii-exec` — two workers double-register the same function ids.
- No API keys are required: with no LLM/embedding keys it degrades to noop compression + BM25-only search (fully functional for save/recall).
- `npm test` mocks `iii-sdk` and needs no running services. `npm run test:integration` requires a live engine + worker.
- Run development services only in an isolated local checkout, never on a Pi or
  on the ports of an existing user daemon. Do not ingest personal history for tests.
- For a disposable development instance, these commands exercise the REST API.
  The POST examples change that instance's data; do not run them against the user's daemon.

```sh
curl --fail-with-body http://127.0.0.1:3111/agentmemory/health
curl --fail-with-body -X POST -H "Content-Type: application/json" -d '{"content":"hello world"}' http://127.0.0.1:3111/agentmemory/remember
curl --fail-with-body -X POST -H "Content-Type: application/json" -d '{"query":"hello"}' http://127.0.0.1:3111/agentmemory/search
```
