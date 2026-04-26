# MCP Extension for Pi Coding Agent

## Current Context
- Pi has no built-in MCP support; README explicitly calls this out as an extension opportunity (`packages/coding-agent/README.md:454`)
- Extension system loads TypeScript files at runtime via jiti — no compilation step
- Extensions receive an `ExtensionAPI` object and call `pi.registerTool()` to surface tools to the LLM
- No existing files are modified; this is a pure addition following the `with-deps/` extension pattern
- Primary runtime is the Bun binary — MCP SDK must be in `node_modules/` next to the extension (outside jiti's virtual modules list)
- Pi auto-discovers extensions from `.pi/extensions/` (project-local) and `~/.pi/agent/extensions/` (global) — no settings config needed for those paths
- Pi can also install extensions directly from a GitHub repo via `pi install git:github.com/user/repo`, reading the `"pi": { "extensions": [...] }` field in the repo's `package.json` to find entry points

## Requirements

### Functional Requirements
- Connect to one or more MCP servers at `session_start` (stdio and SSE/HTTP transports)
- Discover tools from each server via `tools/list` and register each as a first-class pi tool via `pi.registerTool()`
- Namespace tool names as `mcp__<server>__<tool>` to prevent collisions across servers
- Validate LLM-supplied arguments against the tool's raw JSON Schema before forwarding to the MCP server
- Return MCP tool results as `text` content to the LLM
- Close all server connections cleanly on `session_shutdown` (fires on quit, `/reload`, and session replacement)
- `/mcp` command: list connected servers and their tool counts at runtime
- Config: merge global (`~/.pi/agent/mcp.json`) and project-local (`.pi/mcp.json`) server lists — project appends to global
- Env var interpolation in server config (`${ENV_VAR}`) via Pi's existing `resolveConfigValue()` utility

### Non-Functional Requirements
- No modifications to any existing pi source files
- Must work from Bun binary: MCP SDK resolved from `mcp/node_modules/`, not jiti virtual modules
- Connection failures are non-fatal: log error per server and continue with remaining servers
- Tool registration happens inside `session_start` handler (not factory body) — flags not available until then
- Every registered MCP tool must include `promptSnippet` so it appears in the LLM's "Available tools" system prompt section — tools without it are invisible to the LLM
- Validation failures and MCP call errors must `throw new Error(...)` from `execute()` to set `isError: true` on the tool result; returning a value never sets the error flag

## Design Decisions

### 1. MCP SDK vs. manual JSON-RPC
Will use `@modelcontextprotocol/sdk` because:
- Handles both stdio and SSE transports under one API — both are required
- Future-proof against MCP spec changes (protocol versioning, error codes)
- SDK goes in `dependencies`, not `devDependencies` — present in production installs

### 2. Extension location in monorepo
Will place in `.pi/extensions/mcp/` (repo root) because:
- `.pi/extensions/` is one of pi's two auto-discovery paths — no settings config or `-e` flag needed
- Mirrors the `with-deps/` structural pattern (own folder + `package.json`)
- The `package.json` must include `"pi": { "extensions": ["./index.ts"] }` — this is what tells pi which file is the entry point when loading from a package or git install
- Future distribution via `pi install git:github.com/user/repo` requires this same `package.json` structure — designing to it now means zero rework to share later
- `examples/extensions/` is for illustrative examples only; active extensions belong in `.pi/extensions/`

### 3. TypeBox schema wrapping + runtime validation
Will use `Type.Unsafe(inputSchema)` for TypeBox compatibility plus `Value.Check()` for runtime validation because:
- TypeBox is already a virtual module — no extra dependency
- `Value.Check(Type.Unsafe(schema), params)` validates params against the raw JSON Schema
- Malformed LLM arguments surface early (before the MCP server call) with a clear error message returned to the LLM

### 4. Auth / env var handling
Will use `resolveConfigValue()` from `packages/coding-agent/src/core/resolve-config-value.ts` (line 17) because:
- Consistent with how pi resolves API keys and provider credentials across the codebase
- Supports `${ENV_VAR}` syntax in `mcp.json` `env` fields; values come from `process.env` directly — no shell expansion
- No new auth abstraction needed

### 5. MCP resources and prompt templates
Skipped for v1:
- Resources require `before_agent_start` injection — out of scope
- Prompt templates skipped per explicit decision

## Technical Design

### 1. Core Components

```typescript
// config.ts
interface McpServerConfig {
  name: string;
  command?: string;               // stdio: executable path
  args?: string[];
  env?: Record<string, string>;   // supports ${ENV_VAR} interpolation
  url?: string;                   // SSE: http(s) endpoint
}

interface McpConfig {
  servers: McpServerConfig[];
}

function loadConfig(cwd: string): McpConfig
// Reads ~/.pi/agent/mcp.json, then .pi/mcp.json (project appends to global, not replaces)
// Applies resolveConfigValue() to each value in server.env
// Returns { servers: [] } if no config files found — non-fatal

// transport.ts
function createTransport(server: McpServerConfig): Transport
// stdio → new StdioClientTransport({ command, args, env })
// SSE   → new SSEClientTransport(new URL(server.url))
// Throws if neither command nor url is present

// schema.ts
function wrapSchema(inputSchema: object): TSchema
// Returns Type.Unsafe<Record<string, unknown>>(inputSchema)

function validateParams(inputSchema: object, params: unknown): { valid: boolean; errors: string[] }
// Uses Value.Check(Type.Unsafe(inputSchema), params)
// Uses Value.Errors() to collect human-readable error strings on failure

// commands.ts
function registerMcpCommand(pi: ExtensionAPI, clients: Map<string, Client>): void
// Registers /mcp command — prints connected servers and tool count per server

// index.ts — extension factory
export default async function (pi: ExtensionAPI): Promise<void>
```

### 2. Data Flow

```
Extension factory (sync entry point)
  └─ Reads mcp.json config (sync, safe during factory)
  └─ Registers /mcp command
  └─ Registers session_start handler
  └─ Registers session_shutdown handler

session_start
  └─ For each server in config:
       └─ createTransport(server)
       └─ new Client({ name: "pi-mcp", version: "1.0.0" }, { capabilities: {} })
       └─ client.connect(transport)
       └─ client.listTools()
       └─ For each tool:
            └─ pi.registerTool({
                 name: "mcp__<server>__<tool>",
                 label: "[MCP:<server>] <tool>",
                 description: tool.description,
                 promptSnippet: tool.description,   ← required or LLM won't see this tool
                 parameters: wrapSchema(tool.inputSchema),
                 execute: validateThenCall
               })
       └─ ctx.ui.notify("MCP: connected <server> (N tools)", "info")
       └─ ctx.ui.setStatus("mcp-<server>", "MCP:<server>(N)")
       └─ On failure: ctx.ui.notify("MCP: <server> failed: <err>", "error") — non-fatal

LLM tool call → execute(toolCallId, params, signal, onUpdate, ctx)
  └─ if (signal?.aborted) throw new Error("Cancelled")
  └─ validateParams(tool.inputSchema, params)
       └─ If invalid: throw new Error(`Invalid arguments: <errors>`)  ← sets isError: true; do NOT return
  └─ client.callTool({ name: tool.name, arguments: params as Record<string, unknown> })
       └─ If call throws: re-throw so isError: true is set on the result
  └─ Map result.content[] text items → joined string
  └─ Return { content: [{ type: "text", text }], details: result }

session_shutdown
  └─ client.close() for each client in map (errors swallowed)
  └─ clients.clear()
```

### 3. Integration Points
- `pi.registerTool()` — `packages/coding-agent/src/core/extensions/types.ts` (lines 424–471)
- `pi.registerCommand()` — `packages/coding-agent/src/core/extensions/types.ts` (line 1126)
- `pi.on("session_start" | "session_shutdown")` — `packages/coding-agent/src/core/extensions/types.ts` (lines 1074–1110)
- `resolveConfigValue()` — `packages/coding-agent/src/core/resolve-config-value.ts` (line 17)
- `Type.Unsafe()` + `Value` — `typebox` virtual module (`loader.ts:45-57`)
- `@modelcontextprotocol/sdk` — external npm dep, resolved from `mcp/node_modules/`

### 4. Files Changed

All files are new — no existing files are modified.

- `.pi/extensions/mcp/package.json` — package manifest with `"pi": { "extensions": ["./index.ts"] }`, `keywords: ["pi-package"]`, `@modelcontextprotocol/sdk` in `dependencies`, and `@mariozechner/pi-coding-agent` + `typebox` in `peerDependencies: { "*" }`
- `.pi/extensions/mcp/package-lock.json` — committed alongside `package.json` for reproducible installs
- `.pi/extensions/mcp/index.ts` — extension factory entry point
- `.pi/extensions/mcp/config.ts` — mcp.json loader + env resolution
- `.pi/extensions/mcp/transport.ts` — stdio/SSE transport factory
- `.pi/extensions/mcp/schema.ts` — TypeBox wrapping + param validation
- `.pi/extensions/mcp/commands.ts` — /mcp slash command

Existing files referenced (read-only, no changes):
- `packages/coding-agent/src/core/resolve-config-value.ts` (line 17) — `resolveConfigValue()`
- `packages/coding-agent/src/core/extensions/types.ts` (lines 424–471, 1074–1110, 1126) — `ToolDefinition`, `ExtensionAPI`, event types
- `packages/coding-agent/examples/extensions/with-deps/package.json` (lines 1–22) — `package.json` structure to mirror (note: must add `"pi": { "extensions": ["./index.ts"] }` field)

## Implementation Plan

1. Phase 1 — Scaffold
   - Create `.pi/extensions/mcp/package.json` with: `"pi": { "extensions": ["./index.ts"] }`, `keywords: ["pi-package"]`, `@modelcontextprotocol/sdk` in `dependencies`, `@mariozechner/pi-coding-agent` and `typebox` in `peerDependencies` (both `"*"`)
   - Create `.pi/extensions/mcp/config.ts`: `loadConfig(cwd)` reads and merges global + project mcp.json; applies `resolveConfigValue()` to env values; silent on missing files
   - Run `npm install` inside `.pi/extensions/mcp/` to populate `node_modules/` and generate `package-lock.json`; commit both

2. Phase 2 — Core Extension Logic
   - Create `.pi/extensions/mcp/transport.ts`: `createTransport(server)` returns `StdioClientTransport` or `SSEClientTransport` based on which config field is present
   - Create `.pi/extensions/mcp/schema.ts`: `wrapSchema()` with `Type.Unsafe` and `validateParams()` using TypeBox `Value.Check` + `Value.Errors`
   - Create `.pi/extensions/mcp/commands.ts`: `/mcp` command handler that prints connected servers and tool counts via `ctx.ui.notify`
   - Create `.pi/extensions/mcp/index.ts`: export default factory; register `session_start` (connect → discover → registerTool per tool), `session_shutdown` (close all clients), and `/mcp` command

3. Phase 3 — Tests
   - Unit tests for `config.ts`, `schema.ts`, `transport.ts`, and the extension factory (mock MCP SDK Client)
   - Integration test: spin up real `@modelcontextprotocol/server-filesystem`, run full session, assert tool registration and invocation

## Testing Strategy

### Unit Tests

`config.ts`:
- Merges global and project mcp.json — project servers appended after global servers
- Applies `resolveConfigValue()` to each env value (mock `process.env`)
- Returns `{ servers: [] }` when no config files exist
- Skips a config file with invalid JSON without throwing

`schema.ts`:
- `wrapSchema()` returns a TypeBox schema object for an input with properties
- `validateParams()` returns `{ valid: true, errors: [] }` for valid input
- `validateParams()` returns `{ valid: false, errors: [...] }` for missing required fields
- `validateParams()` returns `{ valid: false, errors: [...] }` for wrong property type

`transport.ts`:
- Returns `StdioClientTransport` when config has `command` field
- Returns `SSEClientTransport` when config has `url` field
- Throws descriptive error if neither `command` nor `url` is present

Extension factory + session_start (mock `Client`):
- `pi.registerTool()` is called exactly once per tool returned by `listTools()`
- Tool names follow `mcp__<server>__<tool>` pattern
- Each registered tool has a non-empty `promptSnippet` field
- Connection failure for one server does not prevent other servers from connecting
- `client.close()` is called for each client on `session_shutdown`
- Validation failure in `execute()` throws `Error` (not returns) so `isError: true` is set on the result
- MCP `callTool()` error propagates as a thrown error, not a returned value

### Integration Tests

Using context7 + wiki patterns for test harness:
- Spin up `@modelcontextprotocol/server-filesystem` as a real subprocess; write `.pi/mcp.json` pointing to it
- Load extension via test harness with `-e ./mcp/index.ts`
- Assert: tools from the server appear in `pi.getAllTools()` after `session_start`
- Assert: `mcp__filesystem__<tool>` is invoked when LLM is prompted to list files in a known directory
- Assert: `/reload` triggers reconnection (`session_start` fires again, tools re-registered)
- Assert: connection failure for one server (invalid command) does not crash the session or block other servers
- Assert: `session_shutdown` calls `client.close()` — verify server process exits cleanly

## Observability

- `ctx.ui.notify("MCP: connected <server> (N tools)", "info")` on successful connection per server
- `ctx.ui.notify("MCP: <server> failed: <error>", "error")` on connection failure per server
- `ctx.ui.setStatus("mcp-<server>", "MCP:<server>(N)")` shows live server status in the TUI status bar

## Future Considerations

### Potential Enhancements
- MCP resources: inject via `before_agent_start` for resource-aware context
- MCP prompt templates: surface as `/mcp <template>` slash commands
- Reconnection / retry on server crash mid-session
- AbortSignal wiring: kill and restart stdio server on abort to unblock hanging `callTool()`
- `tools/list_changed` notification handling to re-register tools dynamically at runtime

### Known Limitations
- AbortSignal from `execute()` is not plumbed into in-flight `callTool()` — cancellation not supported in v1
- Bun binary requires `npm install` inside `.pi/extensions/mcp/` before first use — not automated; must be re-run after `@modelcontextprotocol/sdk` version bumps
- MCP SDK protocol version negotiation is handled by the SDK; servers rejecting that version will produce an error at connect time, not a clean fallback
- `.pi/extensions/` is gitignored by default in many setups — verify it is committed if the intent is to share via the repo; for team/cross-project sharing, `pi install git:github.com/user/repo` is the proper path

## Dependencies

- `@modelcontextprotocol/sdk` — production `dependency` in `mcp/package.json`; version `^1.x`; resolved from `mcp/node_modules/` at runtime (required for Bun binary compatibility)
- `@mariozechner/pi-coding-agent` — `peerDependency: "*"` (bundled by pi; do not include in `dependencies`)
- `typebox` — `peerDependency: "*"` (bundled by pi as a virtual module; do not include in `dependencies`)

## Security Considerations
- Env var values in `mcp.json` are resolved via `resolveConfigValue()` — values come from `process.env` directly, no shell expansion
- Stdio server processes spawned by the MCP SDK default to `shell: false` — no shell injection risk from `command`/`args` fields
- Server `env` fields merge onto the existing environment (`{ ...process.env, ...resolvedEnv }`) — Pi's own env vars are not replaced
- `mcp.json` is user-controlled; extension trusts `command`/`args` as-is — no sandboxing in v1

## References
- [research/ext2-mcp-extension.md](ext2-mcp-extension.md) — full extension system architecture notes
- [research/ex3-proposal.md](ex3-proposal.md) — Proposal 1 skeleton code
- [packages/coding-agent/examples/extensions/with-deps/](../packages/coding-agent/examples/extensions/with-deps/) — package structure to mirror
- [packages/coding-agent/src/core/resolve-config-value.ts](../packages/coding-agent/src/core/resolve-config-value.ts) — env var resolution utility
- [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts) — `ExtensionAPI`, `ToolDefinition` contracts
