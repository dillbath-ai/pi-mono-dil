# Module 2 Homework — MCP Extension for Pi



---

## What was built

A pi extension that connects agent sessions to MCP servers. When pi starts a session it reads `.pi/mcp.json`, connects to each configured server over **stdio** (subprocess) or **SSE** (remote URL), discovers available tools, and registers them with the LLM automatically. The LLM can then call any MCP tool as naturally as a built-in pi tool.

### Key design highlights

- **Auto-discovery** — extension lives in `.pi/extensions/mcp/` and is picked up by pi without any explicit configuration
- **Tool naming** — MCP tools surface as `mcp__<server>__<tool>` so they're namespaced and unambiguous
- **Resilience** — 10 s connect timeout + 5 s `listTools` timeout; transport closed on expiry so subprocesses are not abandoned
- **Session safety** — `execute()` reads the live client from the Map at call time, not from the closure, so session restarts don't call closed clients
- **`/mcp` command** — lists connected servers and tool counts at runtime
- **Input validation** — params checked against the MCP tool's JSON Schema before forwarding

### Sample config (`.pi/mcp.json`)

```json
{
  "servers": [
    { "name": "context7", "command": "npx", "args": ["-y", "@upstash/context7-mcp@latest"] },
    { "name": "github",   "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"], "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" } },
    { "name": "remote",   "url": "http://localhost:3000/sse" }
  ]
}
```

---

## Files changed

### Extension source

| File | Purpose |
|------|---------|
| [.pi/extensions/mcp/index.ts](../.pi/extensions/mcp/index.ts) | Factory — wires config, transport, tool registration, lifecycle |
| [.pi/extensions/mcp/config.ts](../.pi/extensions/mcp/config.ts) | Config loading + `${ENV_VAR}` interpolation |
| [.pi/extensions/mcp/transport.ts](../.pi/extensions/mcp/transport.ts) | stdio / SSE transport factory |
| [.pi/extensions/mcp/schema.ts](../.pi/extensions/mcp/schema.ts) | TypeBox wrapping + param validation |
| [.pi/extensions/mcp/commands.ts](../.pi/extensions/mcp/commands.ts) | `/mcp` slash command |
| [.pi/extensions/mcp/package.json](../.pi/extensions/mcp/package.json) | Package manifest (`pi-package` keyword, peer deps) |
| [.pi/extensions/mcp/.gitignore](../.pi/extensions/mcp/.gitignore) | Excludes `node_modules/` |
| [.pi/mcp.json](../.pi/mcp.json) | Sample config pointing to context7 |

### Tests

| File | Purpose |
|------|---------|
| [packages/coding-agent/test/mcp-extension.test.ts](../packages/coding-agent/test/mcp-extension.test.ts) | 26 unit tests — config, schema, transport, factory |

### Dev environment

| File | Purpose |
|------|---------|
| [.devcontainer/devcontainer.json](../.devcontainer/devcontainer.json) | Node 24 devcontainer with pi + Claude Code |

---

## Research trail

| Doc | What it covers |
|-----|---------------|
| [ex1-explore-pi.md](ex1-explore-pi.md) | Initial exploration of pi-mom and pi-pods packages |
| [ext2-mcp-extension.md](ext2-mcp-extension.md) | Deep-dive into pi's extension system, API surface, and TypeBox constraints |
| [ex3-proposal.md](ex3-proposal.md) | Two solution proposals (SDK-based vs zero-dep) with open questions |
| [ex4-plan.md](ex4-plan.md) | Detailed implementation plan (design doc format) |
| [ex5-build.md](ex5-build.md) | Build findings — SDK export map quirks, vi.mock cross-directory limits, and 6 other discoveries |
| [ex6-code review.md](ex6-code review.md) | Iterative code review output (4 rounds, 20+ action items addressed) |
| [design_doc_tempalate.md](design_doc_tempalate.md) | Template used for the plan |
