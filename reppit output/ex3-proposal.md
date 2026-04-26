# Solution Proposals: MCP Extension for Pi Coding Agent

## Solution Proposals

**Context:**
- **Request:** Build an extension for pi that connects to MCP (Model Context Protocol) servers, discovers their tools, and exposes those tools to the LLM so it can call them during coding sessions.
- **Research Source:** `research/mcp-extension.md` — sections 2 (Extension Architecture), 4 (Tool Definition), 5 (Tool Registration Timing), 7 (Subprocess Utilities), 9 (Closest Examples), 12 (TypeBox Constraint), 14 (Constraints and Boundaries).

---

## Proposal 1 — Distributable Pi Package Using the Official MCP SDK

### Overview

A full pi package published to npm (e.g., `@user/pi-mcp`) that uses `@modelcontextprotocol/sdk` as a production dependency. The extension connects to one or more configured MCP servers at session start (supporting both stdio and SSE transports), discovers their tools via `tools/list`, registers each as a first-class pi tool via `pi.registerTool()`, and injects MCP resource content into the system prompt. A `/mcp` command lets users inspect and toggle servers at runtime. The package is installed with `pi install npm:@user/pi-mcp` and survives across pi updates.

### Key Changes

**New files (self-contained pi package):**

```
pi-mcp/
  package.json          "pi": { "extensions": ["./index.ts"] }, dependencies: { "@modelcontextprotocol/sdk": "^1.x" }
  index.ts              Extension factory — reads config, connects servers, registers tools
  config.ts             Reads ~/.pi/agent/mcp.json and .pi/mcp.json; merges global + project
  transport.ts          createStdioTransport() / createSseTransport() using MCP SDK Client
  schema.ts             convertMcpSchema(inputSchema) → Type.Unsafe<Record<string,unknown>>
  commands.ts           /mcp command: list servers, show tools per server, toggle server on/off
```

**Configuration file** (not part of pi's `Settings` interface — read directly by the extension):

```json
// ~/.pi/agent/mcp.json  (global) or .pi/mcp.json (project, overrides global)
{
  "servers": [
    { "name": "filesystem", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"] },
    { "name": "github",     "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" } },
    { "name": "remote",     "url": "http://localhost:3000/sse" }
  ]
}
```

**Extension factory skeleton** (`index.ts`):

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";
import { Type } from "typebox";
import { loadConfig } from "./config.js";

// Keyed by server name
const clients = new Map<string, Client>();

export default async function (pi: ExtensionAPI) {
  const config = loadConfig();        // reads mcp.json; safe during factory
  if (!config.servers?.length) return;

  // Register flags (values readable in session_start)
  pi.registerFlag("mcp-servers", { type: "string", description: "Comma-separated server names to enable" });

  pi.on("session_start", async (_event, ctx) => {
    for (const server of config.servers) {
      try {
        const transport = server.url
          ? new SSEClientTransport(new URL(server.url))
          : new StdioClientTransport({ command: server.command, args: server.args, env: server.env });
        const client = new Client({ name: "pi-mcp", version: "1.0.0" }, { capabilities: {} });
        await client.connect(transport);
        clients.set(server.name, client);

        const { tools } = await client.listTools();
        for (const tool of tools) {
          const toolName = `mcp__${server.name}__${tool.name}`;
          pi.registerTool({
            name: toolName,
            label: `[MCP:${server.name}] ${tool.name}`,
            description: tool.description ?? tool.name,
            parameters: Type.Unsafe(tool.inputSchema ?? { type: "object", properties: {} }),
            prepareArguments: (args) => (typeof args === "object" && args !== null ? args : {}) as any,
            async execute(_id, params, signal, onUpdate) {
              const result = await client.callTool({ name: tool.name, arguments: params as Record<string,unknown> });
              const text = result.content
                .filter((c: any) => c.type === "text")
                .map((c: any) => c.text)
                .join("\n");
              return { content: [{ type: "text", text }], details: result };
            },
          });
        }
        ctx.ui.notify(`MCP: connected ${server.name} (${tools.length} tools)`, "info");
        ctx.ui.setStatus(`mcp-${server.name}`, ctx.ui.theme.fg("success", `MCP:${server.name}`));
      } catch (err) {
        ctx.ui.notify(`MCP: failed to connect ${server.name}: ${err}`, "error");
      }
    }
  });

  pi.on("session_shutdown", async () => {
    for (const client of clients.values()) await client.close().catch(() => {});
    clients.clear();
  });

  // /mcp command: list servers and tools
  pi.registerCommand("mcp", {
    description: "List MCP servers and tools",
    handler: async (_args, ctx) => {
      const lines = [`MCP servers (${clients.size} connected):`];
      for (const [name, client] of clients) {
        const { tools } = await client.listTools();
        lines.push(`  ${name}: ${tools.length} tools — ${tools.map(t => t.name).join(", ")}`);
      }
      ctx.ui.notify(lines.join("\n"), "info");
    },
  });
}
```

**Cited anchors from research:**
- Factory + async init pattern: `research/mcp-extension.md §2.1` — pi awaits async factory before startup
- `session_start` for flags & connection: `research/mcp-extension.md §9.2` (ssh.ts pattern), `§14` (flags not available during factory)
- `registerTool()` in `session_start`: `research/mcp-extension.md §5` — valid, `refreshTools()` is live
- `Type.Unsafe()` for JSON Schema: `research/mcp-extension.md §12`
- Tool naming: `research/mcp-extension.md §14` — names globally unique; use `mcp__<server>__<tool>`
- npm deps via package.json: `research/mcp-extension.md §9.3`, `§14` — must be in `dependencies`, not `devDependencies`
- `session_shutdown` for cleanup: `research/mcp-extension.md §10`
- `/mcp` command registration: `core/extensions/types.ts:1126`

### Trade-offs

| Benefit | Risk |
|---------|------|
| Full MCP SDK — stdio + SSE transports, robust protocol handling, future-proof against MCP spec changes | Requires `npm install` after `pi install`; users must have Node + npm available |
| Published as pi package — `pi install npm:@user/pi-mcp`; upgradeable via `pi update` | MCP SDK is an external dep; pi's production install (`--omit=dev`) means SDK must be in `dependencies` not `devDependencies` (see `settings-manager.ts:88`, README packages section) |
| Multi-server, multi-transport (stdio + SSE) in one extension | Async factory increases pi startup time proportional to the number of servers and their `initialize` latency |
| `/mcp` command for runtime inspection and `/reload` re-discovery | Tool name collisions if two servers expose tools with the same base name — namespace with `mcp__<server>__` mitigates this but makes LLM tool names verbose |
| MCP resources could feed into `before_agent_start` for context injection | `resources_discover` event cannot currently inject arbitrary text content — only paths to skill/prompt/theme files; resource injection would require the `before_agent_start` hook and manual fetching |

### Validation

- **Unit:** Mock `@modelcontextprotocol/sdk` Client; assert `registerTool()` is called N times for N tools returned by `listTools()`.
- **Integration:** Spin up a real MCP server (`@modelcontextprotocol/server-filesystem`); run `pi -e ./index.ts -p "list files in /tmp"` and assert the `mcp__filesystem__list_directory` tool is called.
- **Connection failure:** Assert the extension logs an error and does not crash pi if a server is unreachable at `session_start`.
- **Session shutdown:** Assert `client.close()` is called for each connected server on `session_shutdown`.
- **Tool name uniqueness:** Assert no two servers' tools collide after namespacing.
- **Reload:** Assert `/reload` triggers reconnection (`session_start` fires after reload per `types.ts:512`).

### Open Questions

1. **MCP resources vs. pi tools:** MCP servers expose both `tools` and `resources`. Pi has no built-in resource abstraction. Should resources be injected into the system prompt via `before_agent_start`, or skipped in v1?
2. **Authentication per server:** MCP servers may require Bearer tokens or other auth. Should the extension read `env` vars from the config, from pi's `AuthStorage`, or from the system environment?
3. **Bun binary compatibility:** `@modelcontextprotocol/sdk` is not in `VIRTUAL_MODULES` (`loader.ts:45-57`). If pi is run from its compiled Bun binary, jiti uses `virtualModules` with `tryNative: false`, meaning the SDK must be resolvable from the extension's `node_modules`. This needs verification against the Bun binary path.
4. **Tool schema validation:** `Type.Unsafe()` bypasses TypeBox validation. Should the extension add a `prepareArguments` shim that validates params against the raw JSON Schema at runtime?
5. **MCP prompts:** MCP servers also expose `prompts` — reusable prompt templates. Should these be registered with pi's prompt template system via `resources_discover`?

---

## Proposal 2 — Zero-Dependency Single-File Extension via Raw stdio/JSON-RPC

### Overview

A single `.ts` file that implements the MCP stdio transport manually using `node:child_process` and `node:readline`, requiring no npm install. The user drops the file into `~/.pi/agent/extensions/mcp.ts`, creates a `~/.pi/agent/mcp.json` config, and the extension works immediately. Only stdio servers are supported (the most common case: `filesystem`, `github`, `puppeteer`, `git`, etc.). This trades protocol robustness for zero-setup friction.

### Key Changes

**Single file** (`~/.pi/agent/extensions/mcp.ts`):

```typescript
import { spawn, type ChildProcess } from "node:child_process";
import { createInterface } from "node:readline";
import { readFileSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "typebox";

// ---- Minimal MCP stdio client ----

interface McpRequest { jsonrpc: "2.0"; id: number; method: string; params?: unknown }
interface McpResponse { jsonrpc: "2.0"; id: number; result?: unknown; error?: { message: string } }

class StdioMcpClient {
  private proc: ChildProcess;
  private pending = new Map<number, { resolve(v: unknown): void; reject(e: Error): void }>();
  private nextId = 1;

  constructor(command: string, args: string[], env?: Record<string, string>) {
    this.proc = spawn(command, args, {
      stdio: ["pipe", "pipe", "inherit"],
      env: { ...process.env, ...env },
      shell: false,
    });
    const rl = createInterface({ input: this.proc.stdout! });
    rl.on("line", (line) => {
      try {
        const msg = JSON.parse(line) as McpResponse;
        const p = this.pending.get(msg.id);
        if (!p) return;
        this.pending.delete(msg.id);
        if (msg.error) p.reject(new Error(msg.error.message));
        else p.resolve(msg.result);
      } catch { /* ignore malformed lines */ }
    });
    this.proc.on("exit", () => {
      for (const p of this.pending.values()) p.reject(new Error("MCP server exited"));
      this.pending.clear();
    });
  }

  call(method: string, params?: unknown): Promise<unknown> {
    return new Promise((resolve, reject) => {
      const id = this.nextId++;
      this.pending.set(id, { resolve, reject });
      const msg: McpRequest = { jsonrpc: "2.0", id, method, params };
      this.proc.stdin!.write(JSON.stringify(msg) + "\n");
    });
  }

  async initialize() {
    await this.call("initialize", {
      protocolVersion: "2024-11-05",
      clientInfo: { name: "pi-mcp", version: "1.0.0" },
      capabilities: {},
    });
    await this.call("notifications/initialized");
  }

  async listTools(): Promise<Array<{ name: string; description?: string; inputSchema: object }>> {
    const result = await this.call("tools/list") as { tools: any[] };
    return result.tools ?? [];
  }

  async callTool(name: string, args: unknown): Promise<string> {
    const result = await this.call("tools/call", { name, arguments: args }) as { content: any[] };
    return (result.content ?? [])
      .filter((c: any) => c.type === "text")
      .map((c: any) => c.text)
      .join("\n");
  }

  close() { this.proc.kill("SIGTERM"); }
}

// ---- Config ----

interface ServerConfig {
  name: string;
  command: string;
  args?: string[];
  env?: Record<string, string>;
}

function loadConfig(cwd: string): ServerConfig[] {
  const paths = [
    join(homedir(), ".pi", "agent", "mcp.json"),
    join(cwd, ".pi", "mcp.json"),  // project overrides global
  ];
  let servers: ServerConfig[] = [];
  for (const p of paths) {
    try {
      const parsed = JSON.parse(readFileSync(p, "utf-8"));
      if (Array.isArray(parsed.servers)) servers = [...servers, ...parsed.servers];
    } catch { /* file absent or invalid — skip */ }
  }
  return servers;
}

// ---- Extension ----

export default function (pi: ExtensionAPI) {
  const clients = new Map<string, StdioMcpClient>();

  pi.on("session_start", async (_event, ctx) => {
    const servers = loadConfig(ctx.cwd);
    for (const server of servers) {
      try {
        const client = new StdioMcpClient(server.command, server.args ?? [], server.env);
        await client.initialize();
        const tools = await client.listTools();
        clients.set(server.name, client);

        for (const tool of tools) {
          const toolName = `mcp__${server.name}__${tool.name}`;
          pi.registerTool({
            name: toolName,
            label: `[MCP:${server.name}] ${tool.name}`,
            description: tool.description ?? tool.name,
            parameters: Type.Unsafe(tool.inputSchema),
            prepareArguments: (a) => (a && typeof a === "object" ? a : {}) as any,
            async execute(_id, params, signal) {
              const text = await client.callTool(tool.name, params);
              return { content: [{ type: "text", text }], details: {} };
            },
          });
        }
        ctx.ui.setStatus(`mcp-${server.name}`, `MCP:${server.name}(${tools.length})`);
      } catch (err) {
        ctx.ui.notify(`MCP: ${server.name} failed: ${err}`, "error");
      }
    }
  });

  pi.on("session_shutdown", () => {
    for (const client of clients.values()) client.close();
    clients.clear();
  });
}
```

**Cited anchors from research:**
- Direct `spawn()` for long-lived processes: `research/mcp-extension.md §7.2` — same pattern as `ssh.ts` and `subagent/index.ts`
- `Type.Unsafe()` for JSON Schema wrapping: `research/mcp-extension.md §12`
- `session_start` for connection init: `research/mcp-extension.md §5` and `§10`
- `session_shutdown` for cleanup: `research/mcp-extension.md §10`, `§14`
- Single-file, no npm install: `research/mcp-extension.md §2.1` — jiti resolves Node.js built-ins natively; `typebox` is a virtual module
- `ctx.exec()` uses `shell: false`: `research/mcp-extension.md §14` — the manual `spawn()` approach here is consistent

### Trade-offs

| Benefit | Risk |
|---------|------|
| **Zero setup** — drop the `.ts` file, create `mcp.json`, done. No `npm install`, no package.json | Manual JSON-RPC 2.0 implementation: MCP spec changes (new transport fields, error codes, streaming) require manual updates to the extension |
| Works identically in Bun binary and Node.js — no `node_modules` resolution path needed | **stdio only** — cannot connect to remote SSE/HTTP MCP servers; rules out hosted MCP endpoints |
| Single file is auditable, copy-pasteable, and easy to fork/modify | No SDK-level retries, reconnection, or protocol negotiation beyond basic `initialize` |
| `typebox` is already a virtual module (`loader.ts:45-57`) — the only import besides Node built-ins | `AbortSignal` from `execute()` is not plumbed into the `callTool` request — a long-running MCP tool call cannot be cancelled mid-flight without killing the server process |
| Faster startup — no async SDK initialization overhead | JSON parsing of MCP responses is naive; malformed partial lines from a misbehaving server could break the readline queue |

### Validation

- **Unit:** Stub `spawn()` to return a mock process that emits preset JSON-RPC responses; assert the `StdioMcpClient` resolves `listTools()` correctly and rejects on error responses.
- **Integration:** Spawn a real `@modelcontextprotocol/server-filesystem` as a subprocess; verify tool registration and a real `tools/call` round-trip.
- **Config merge:** Assert that `.pi/mcp.json` servers are appended after global `~/.pi/agent/mcp.json` servers (not replaced).
- **Crash recovery:** Kill the MCP server process mid-call; assert the extension logs an error, does not crash pi, and that pending promises reject cleanly.
- **AbortSignal gap:** Manually verify (or add a test) that Escape/abort during a long `callTool()` does not hang pi's agent loop.
- **Session shutdown:** Assert `client.close()` (SIGTERM) is called on `/reload` and quit.

### Open Questions

1. **Cancellation gap:** `AbortSignal` from `execute()` is available but the raw stdio client has no way to cancel an in-flight `tools/call` without killing the server process (which would break all other pending calls). Should the extension kill and restart the server on abort, or accept that MCP tool calls are non-cancellable in this implementation?
2. **Protocol version:** The manual `initialize` hardcodes `"protocolVersion": "2024-11-05"`. How should version negotiation be handled if servers return a different version or reject the handshake?
3. **Notifications and sampling:** MCP servers can send unsolicited notifications (e.g., `tools/list_changed`). The `readline` loop silently drops these because they have no `id`. Should the extension handle `tools/list_changed` to re-register tools at runtime?
4. **Project config scoping:** `loadConfig(ctx.cwd)` merges global and project servers. If a project-local server uses a relative command path, should the command be resolved relative to `ctx.cwd` or the config file location?
5. **Upgrade path:** If users start with Proposal 2 and want to migrate to Proposal 1 (SDK-based), tool names are identical (`mcp__<server>__<tool>`). Is the session JSONL format forward-compatible if tool names are preserved?

---

## Comparison at a Glance

| Dimension | Proposal 1 (SDK package) | Proposal 2 (zero-dep single file) |
|-----------|--------------------------|-------------------------------------|
| Install | `pi install npm:@user/pi-mcp` | Copy `.ts` + create `mcp.json` |
| Transports | stdio + SSE/HTTP | stdio only |
| Protocol robustness | MCP SDK handles versioning, retries | Manual JSON-RPC; brittle to spec drift |
| MCP resources | Extensible (add `before_agent_start`) | Not included |
| Bun binary compat | Needs verification (`node_modules` outside virtualModules) | Confirmed — only Node built-ins + `typebox` |
| Startup latency | Higher (SDK init per server) | Lower (direct spawn) |
| Cancellation | SDK may support it | Not supported without killing server |
| Upgrade/share | npm versioning, `pi update` | Manual file replacement |
| Recommended for | Production use, remote servers, teams | Local tools only, quick experiments |
