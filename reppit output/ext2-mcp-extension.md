# Research: MCP Extension for Pi Coding Agent

**Date:** 2026-04-26  
**Scope:** Document the existing codebase as it is today in relation to building an MCP (Model Context Protocol) extension for pi.

---

## Summary of Findings

There is **no MCP code in the repository today**. The README explicitly states "No MCP — build an extension that adds MCP support." The extension system is the designed integration point. Extensions are TypeScript modules loaded at runtime via `jiti` (no compilation step) that receive an `ExtensionAPI` object and can: register tools the LLM can call, subscribe to lifecycle events, spawn child processes, and manage async connections. An MCP extension would connect to MCP servers at startup (via stdio, SSE, or HTTP), discover the tools they expose, convert MCP's JSON Schema tool specs into TypeBox schemas, and call `pi.registerTool()` for each, bridging LLM tool calls to the MCP server at execution time.

---

## 1. MCP References in the Codebase Today

**Grep result:** `grep -rn "mcp\|MCP" packages/ --include="*.ts" -l` returns:
- `packages/coding-agent/test/settings-manager-bug.test.ts` — unrelated (contains the string "mcp" in a mock path string, not MCP protocol)
- `packages/ai/src/utils/oauth/anthropic.ts` — unrelated

The README (`packages/coding-agent/README.md:454`) states:

> **No MCP.** Build CLI tools with READMEs (see Skills), or build an extension that adds MCP support.

The same README lists "MCP server integration" under the "What's possible" list for extensions (`README.md:351`).

---

## 2. Extension System Architecture

### 2.1 Entry Point and Loading

**File:** `packages/coding-agent/src/core/extensions/loader.ts`

Extensions are loaded by `discoverAndLoadExtensions()` (line 560) and `loadExtensions()` (line 422). The loader:

1. Discovers extension files from three locations in order (`loader.ts:579-603`):
   - `.pi/extensions/` (project-local, relative to `cwd`)
   - `~/.pi/agent/extensions/` (global)
   - Explicitly configured paths from `settings.json` / CLI `-e` flags

2. Resolves each file using `@mariozechner/jiti` (`loader.ts:342-353`) — TypeScript is executed directly without a build step. Two resolution modes:
   - **Bun binary**: uses `virtualModules` for bundled packages (no filesystem lookup)
   - **Node.js/dev**: uses path `aliases` pointing to built dist files

3. Virtual modules available to extension authors (`loader.ts:45-57`):
   ```
   typebox, @sinclair/typebox
   @mariozechner/pi-agent-core
   @mariozechner/pi-ai
   @mariozechner/pi-ai/oauth
   @mariozechner/pi-tui
   @mariozechner/pi-coding-agent
   ```
   External npm packages (e.g., the MCP TypeScript SDK) are resolved from the extension's own `node_modules/`.

4. Each extension file must export a default function (`ExtensionFactory`, `types.ts:1357`):
   ```typescript
   export type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
   ```
   If the factory returns a `Promise`, pi awaits it before continuing startup — before `session_start`, before `resources_discover`, and before queued provider registrations are flushed.

### 2.2 Extension Runtime Lifecycle

**File:** `packages/coding-agent/src/core/extensions/loader.ts:134-180`

A single `ExtensionRuntime` object is shared across all extensions in a session. It is created with throwing stubs for action methods (`.sendMessage`, `.setModel`, etc.) by `createExtensionRuntime()`. These stubs are replaced by real implementations when `ExtensionRunner.bindCore()` is called (`runner.ts:262-332`).

Key state:
- `runtime.pendingProviderRegistrations` — queued `registerProvider()` calls made before `bindCore()`; flushed when the runner binds to a session.
- `runtime.flagValues` — CLI flag values registered by extensions via `registerFlag()`.
- `runtime.assertActive()` / `runtime.invalidate()` — stale-instance detection after reload or session replacement.

**`Extension` object** (per loaded file, `types.ts:1516-1526`):
```typescript
interface Extension {
  path: string;           // original (possibly relative) path
  resolvedPath: string;   // absolute
  sourceInfo: SourceInfo;
  handlers:       Map<string, HandlerFn[]>;
  tools:          Map<string, RegisteredTool>;
  messageRenderers: Map<string, MessageRenderer>;
  commands:       Map<string, RegisteredCommand>;
  flags:          Map<string, ExtensionFlag>;
  shortcuts:      Map<KeyId, ExtensionShortcut>;
}
```

### 2.3 Extension Runner

**File:** `packages/coding-agent/src/core/extensions/runner.ts`

`ExtensionRunner` manages all loaded extensions and dispatches events:
- `bindCore(actions, contextActions, providerActions)` (line 262) — connects to session.
- `bindCommandContext(actions)` (line 334) — adds session-switch/fork/reload actions for command handlers.
- `setUIContext(uiContext)` (line 353) — attaches mode-specific UI (TUI in interactive mode; no-op stubs in print/RPC mode).
- `emit(event)` (line 676) — generic event emission, calls handlers on all extensions in load order.
- `emitToolCall(event)` (line 760) — can block tool; returns `{ block?: boolean, reason?: string }`.
- `emitToolResult(event)` (line 710) — can modify tool result content/details/isError.
- `emitContext(messages)` (line 812) — allows message mutation before each LLM call.
- `emitBeforeAgentStart(...)` (line 878) — can prepend messages and replace system prompt.
- `getAllRegisteredTools()` (line 370) — first-registration-wins when names collide.

---

## 3. ExtensionAPI — Public Surface for Extension Authors

**File:** `packages/coding-agent/src/core/extensions/types.ts:1069-1295`

```typescript
interface ExtensionAPI {
  // Event subscription
  on(event: "session_start",      handler): void;
  on(event: "session_shutdown",   handler): void;
  on(event: "session_before_compact", handler): void;
  on(event: "context",            handler): void;  // mutate messages before LLM call
  on(event: "before_agent_start", handler): void;  // replace system prompt
  on(event: "tool_call",          handler): void;  // block or mutate args in-place
  on(event: "tool_result",        handler): void;  // mutate result content/details
  on(event: "resources_discover", handler): void;  // add dynamic skill/prompt/theme paths
  on(event: "input",              handler): void;  // transform or intercept user input
  // ... (full list: 25 event types, types.ts:1074-1110)

  // Registration
  registerTool(tool: ToolDefinition): void;
  registerCommand(name: string, options): void;
  registerShortcut(shortcut: KeyId, options): void;
  registerFlag(name: string, options): void;
  registerProvider(name: string, config: ProviderConfig): void;
  registerMessageRenderer(customType: string, renderer): void;

  // Actions (require session to be active)
  sendMessage(message, options?): void;
  sendUserMessage(content, options?): void;
  appendEntry(customType, data?): void;          // persist state to JSONL session
  setSessionName(name): void;
  setActiveTools(toolNames: string[]): void;
  getActiveTools(): string[];
  getAllTools(): ToolInfo[];
  setModel(model): Promise<boolean>;
  exec(command, args, options?): Promise<ExecResult>;  // spawn subprocess

  // Shared event bus for inter-extension communication
  events: EventBus;
}
```

---

## 4. Tool Definition and Execution Contract

**File:** `packages/coding-agent/src/core/extensions/types.ts:424-471`

```typescript
interface ToolDefinition<TParams extends TSchema, TDetails, TState> {
  name: string;           // LLM tool name (must be unique)
  label: string;          // human-readable UI label
  description: string;    // description sent to LLM
  promptSnippet?: string; // one-line blurb injected into system prompt
  promptGuidelines?: string[]; // guideline bullets appended to system prompt
  parameters: TParams;    // TypeBox schema for LLM-validated params
  executionMode?: "sequential" | "parallel";
  prepareArguments?: (args: unknown) => Static<TParams>; // compat shim

  execute(
    toolCallId: string,
    params: Static<TParams>,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<TDetails> | undefined,  // streaming updates
    ctx: ExtensionContext,
  ): Promise<AgentToolResult<TDetails>>;

  renderCall?(...):   Component;  // TUI rendering of the call
  renderResult?(...): Component;  // TUI rendering of the result
}
```

**`AgentToolResult<T>`** (`packages/agent/src/types.ts:292-302`):
```typescript
interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];  // returned to the LLM
  details: T;                               // structured data for logs/UI
  terminate?: boolean;                      // hint agent to stop after this batch
}
```

**Streaming updates** (`agent/src/types.ts:305`):
```typescript
type AgentToolUpdateCallback<T> = (partialResult: AgentToolResult<T>) => void;
```
Call `onUpdate({ content: [...], details: ... })` during execution to stream intermediate progress to the TUI.

---

## 5. Tool Registration Timing

**File:** `packages/coding-agent/src/core/extensions/loader.ts:202-209`

`registerTool()` can be called:
- **During factory execution** — tools are stored in `extension.tools` immediately; `runtime.refreshTools()` is a no-op at this stage (before bind).
- **In `session_start` handler** — valid; `runtime.refreshTools()` is live and triggers immediate tool refresh in the session.
- **From a command handler or event callback** — valid; tools take effect immediately.

First registration wins when tool names collide across extensions (`runner.ts:371-380`).

---

## 6. ExtensionContext (Available in All Event Handlers and Tool execute)

**File:** `packages/coding-agent/src/core/extensions/types.ts:296-325`

```typescript
interface ExtensionContext {
  ui: ExtensionUIContext;           // select, confirm, input, notify, setStatus, setWidget, ...
  hasUI: boolean;                   // false in print/RPC mode
  cwd: string;                      // current working directory
  sessionManager: ReadonlySessionManager;
  modelRegistry: ModelRegistry;
  model: Model<any> | undefined;
  isIdle(): boolean;
  signal: AbortSignal | undefined;  // live only during streaming
  abort(): void;
  hasPendingMessages(): boolean;
  shutdown(): void;
  getContextUsage(): ContextUsage | undefined;
  compact(options?): void;
  getSystemPrompt(): string;
}
```

---

## 7. Subprocess and Process Communication Utilities

Extensions can spawn external processes using two mechanisms:

### 7.1 `pi.exec()` — Simple One-Shot Commands

**File:** `packages/coding-agent/src/core/exec.ts`

```typescript
pi.exec(command: string, args: string[], options?: ExecOptions): Promise<ExecResult>
// ExecResult: { stdout, stderr, code, killed }
// ExecOptions: { signal?, timeout?, cwd? }
```
Internally uses `spawn()` with `stdio: ["ignore", "pipe", "pipe"]` and `shell: false`. Supports abort signal and timeout.

### 7.2 Direct `node:child_process` spawn

Used in examples for long-running connections. The `ssh.ts` example (`examples/extensions/ssh.ts`) and `subagent/index.ts` (`examples/extensions/subagent/index.ts`) both use `spawn()` directly for processes that stay alive across multiple tool invocations.

---

## 8. Settings and Configuration for Extensions

**File:** `packages/coding-agent/src/core/settings-manager.ts:72-108`

```typescript
interface Settings {
  packages?: PackageSource[];  // npm/git packages
  extensions?: string[];       // local extension file paths or directories
  // ...
}
```

Extensions are configured in:
- `~/.pi/agent/settings.json` (global)
- `.pi/settings.json` (project-local; project overrides global)

Example settings.json for an MCP extension installed as an npm package:
```json
{
  "packages": ["npm:@user/pi-mcp-extension"]
}
```

Or for a local development file:
```json
{
  "extensions": ["~/.pi/agent/extensions/mcp.ts"]
}
```

CLI override (one-shot, no settings.json):
```bash
pi -e ./mcp-extension.ts
```

---

## 9. Examples Most Relevant to an MCP Extension

All in `packages/coding-agent/examples/extensions/`.

### 9.1 `dynamic-tools.ts` — Registering Tools at Runtime

Shows registering tools inside `session_start` and from command handlers.  
Pattern: call `pi.registerTool(...)` inside the `session_start` handler after discovering available tools from an external source.

### 9.2 `ssh.ts` — Long-Running External Connection + Tool Wrapping

Shows:
- Registering a CLI flag (`pi.registerFlag("ssh", ...)`) to configure the remote target.
- Reading the flag in `session_start` (flags are not available during factory execution).
- Creating a long-lived connection (SSH) and proxying built-in tool operations through it.
- Replacing `read`, `write`, `edit`, `bash` tools by re-registering them with custom `Operations` objects.

This is the closest structural analog to an MCP extension: establish connection on `session_start`, intercept/proxy tool execution.

### 9.3 `with-deps/index.ts` — Extension with npm Dependencies

Shows that external npm packages (e.g., the `@modelcontextprotocol/sdk` package) can be imported if present in a `package.json` alongside the extension file.

### 9.4 `subagent/index.ts` — Spawning and Streaming from a Subprocess

Shows:
- Spawning a long-running process and reading its JSON-line stdout stream.
- Using `onUpdate(partialResult)` to stream intermediate results to the TUI.
- Using `AbortSignal` to cancel the subprocess.

---

## 10. Key Events for an MCP Extension

| Event | When fired | Relevance for MCP |
|-------|-----------|-------------------|
| `session_start` | Once per session init | Connect to MCP servers, discover tools, call `registerTool()` for each |
| `session_shutdown` | Before session ends/reload | Close MCP connections, clean up |
| `resources_discover` | After `session_start`, on `/reload` | Could re-discover MCP tools on reload |
| `tool_call` | Before each tool execution | Intercept to route to MCP (alternative: register MCP tools directly) |
| `before_agent_start` | Before each LLM turn starts | Inject MCP resource content into system prompt |
| `context` | Before each LLM API call | Could inject MCP resource URIs into message context |

---

## 11. Data Flow for an MCP Extension (Conceptual)

```
Extension factory (async) runs at startup
  └─ Connect to MCP server (stdio/SSE/HTTP)
       └─ Call MCP initialize + tools/list
            └─ For each MCP tool:
                 └─ Convert MCP JSON Schema → TypeBox schema
                 └─ pi.registerTool({ name, description, parameters, execute: callMcpTool })

session_start event
  └─ Confirm connection active; notify user

Each LLM tool call (via agent-loop):
  └─ Agent.execute(toolCallId, params, signal, onUpdate, ctx)
       └─ extension execute() handler
            └─ Call MCP server: tools/call { name, arguments: params }
                 └─ Stream partial results via onUpdate() if server supports progress
            └─ Return { content: [{ type: "text", text: result }], details: mcpResult }

session_shutdown event
  └─ Close MCP transport
```

---

## 12. TypeBox Schema Requirement

MCP tool input schemas are JSON Schema objects. Pi tool parameters must be TypeBox `TSchema` objects. The MCP TypeScript SDK (`@modelcontextprotocol/sdk`) exposes tool schemas as plain JSON Schema. TypeBox's `Type.Unsafe()` can wrap an existing JSON Schema object so it satisfies the `TSchema` constraint without manual conversion:

```typescript
import { Type } from "typebox";
const params = Type.Unsafe<Record<string, unknown>>(mcpTool.inputSchema);
```

This is not an existing pattern in the codebase; it would be introduced by the extension.

---

## 13. File and Directory Reference

```
packages/coding-agent/
  src/core/extensions/
    types.ts          ExtensionAPI, ToolDefinition, all event types, ExtensionContext
    loader.ts         jiti-based loader, discoverAndLoadExtensions(), VIRTUAL_MODULES
    runner.ts         ExtensionRunner class, event dispatch, bindCore/bindCommandContext
    index.ts          Re-exports public types

  src/core/
    exec.ts           execCommand() — subprocess spawn utility
    settings-manager.ts  Settings interface (extensions, packages fields)
    event-bus.ts      EventBus for inter-extension communication

  examples/extensions/
    dynamic-tools.ts  Register tools in session_start and from commands
    ssh.ts            Long-running external connection + tool wrapping + flags
    subagent/index.ts Spawn subprocess, stream JSON output, onUpdate pattern
    with-deps/        Extension with its own package.json and npm dependencies

packages/agent/src/
  types.ts            AgentToolResult<T>, AgentToolUpdateCallback<T>, AgentTool
  agent-loop.ts       agentLoop() — LLM ↔ tool call cycle
```

---

## 14. Constraints and Boundaries

- **No TypeScript compilation** — extensions are executed as-is by jiti. TypeScript syntax and imports from bundled virtual modules work out of the box. For external npm packages (like the MCP SDK), a `package.json` with `dependencies` must exist in or near the extension file, and `npm install` must have been run.

- **Tool names must be globally unique** — first registration across all loaded extensions wins. MCP tool names should be namespaced (e.g., `mcp__<server>__<tool>`).

- **Flags are not available during factory execution** — only available in and after `session_start`. Read flags inside the event handler, not during factory setup.

- **`registerTool()` during factory** is valid but `refreshTools()` is a no-op until `bindCore()` runs. This is fine — the runtime picks up all tools at bind time.

- **Session shutdown vs. extension reload** — `session_shutdown` fires on quit, reload (`/reload`), and session replacement. Close any MCP transport in this handler, since the extension instance is invalidated afterward.

- **`ctx.exec()` uses `shell: false`** — no shell metacharacter expansion. Pass MCP server command as `[command, ...args]` array.
