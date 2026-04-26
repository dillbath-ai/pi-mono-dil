Code Review
Summary: The MCP extension is well-structured and handles the core lifecycle correctly. The most actionable issues are: a signal-abort gap that leaves in-flight MCP calls uninterruptible, a duplicate-server-name collision that silently orphans clients, and several missing edge cases in validation and result handling.

Action Items:

🔴 Wire signal into client.callTool() so in-flight calls can be cancelled in index.ts:63. The signal is checked at entry (signal?.aborted) but not passed to the MCP call — a cancellation mid-flight hangs until the server responds. Pass { signal } as the second argument to client.callTool() if the SDK supports it, or wrap with Promise.race([call, abortPromise]).

🔴 Guard against duplicate server names across global + project config in index.ts:36. If both ~/.pi/agent/mcp.json and .pi/mcp.json define a server named "filesystem", the second clients.set(server.name, client) silently overwrites the first — but both have already registered tools under the same mcp__filesystem__* prefix. The first client leaks (never closed). Fix: check for name collision in loadConfig or before clients.set, and either error or deduplicate with a suffix.

🟡 Add a connection timeout to client.connect() in index.ts:33. A subprocess that starts but never sends a ready message will hang session_start indefinitely. Wrap with Promise.race([client.connect(transport), timeout(10_000)]) and throw a descriptive error on timeout.

🟡 session_start re-registers tools on every call without deduplication in index.ts:42. Each session_start clears clients but calls pi.registerTool() again for every discovered tool. If pi does not deduplicate by name, the tool list grows on each session cycle. Check whether pi.registerTool() is idempotent; if not, track registered tool names and skip re-registration.

🟡 env spread silently includes undefined values from process.env in transport.ts:17. { ...process.env, ...server.env } casts to Record<string, string> but process.env values are string | undefined. Some subprocess APIs reject undefined env values. Filter before spreading: Object.fromEntries(Object.entries(process.env).filter(([, v]) => v !== undefined)).

🟡 Non-text MCP content is silently dropped in index.ts:68. If an MCP server returns image or resource content, the filter .filter(c => c.type === "text") discards it with no indication to the LLM. Add a fallback line like "(non-text content omitted)" when text extraction produces nothing from a non-empty result, so the LLM knows data existed.

🟡 Empty-string tool result is replaced with "(no text output)" in index.ts:74. text || "(no text output)" treats an intentional empty string as missing output. Use text.length > 0 ? text : "(no text output)" after extracting — or better, check result.content.length === 0 to distinguish "no content items" from "empty text content".

🟡 ${ENV_VAR} interpolation is only applied to env values, not to url or command in config.ts:22-28. A user writing "url": "http://${MCP_HOST}/sse" in their config will get the literal string. Apply resolveEnvValue() to command, args entries, and url in resolveServerEnv().

🟡 package-lock.json is untracked — not committed to git. Without it, npm install in the extension directory may resolve @modelcontextprotocol/sdk@^1.29.0 to a different patch version on another machine. Add it to the commit.

🟢 validateParams only checks required — no type checking of field values in schema.ts:29-33. A missing-required error catches the most common mistake, but passing { name: 42 } when name is typed as string passes validation silently. Consider using TypeBox's Value.Check(wrapSchema(inputSchema), params) for full structural validation instead of the manual check.

🟢 /mcp command calls client.listTools() live on every invocation in commands.ts:17. Fine for debugging, but each call is a synchronous IPC round-trip per server. Cache the tool list on clients (e.g., a parallel Map<string, Tool[]>) and refresh only on demand.

🟢 No test for session_shutdown handler or the /mcp command handler in mcp-extension.test.ts. The three factory tests cover setup and the error path, but session_shutdown (client.close loop) and /mcp with zero/multiple servers are untested. These are good candidates for the integration test suite noted in the known gaps.

🟢 result.content cast is structurally unsafe in index.ts:68. result.content as Array<{ type: string; text?: string }> will not throw if the MCP server returns a non-conforming shape — it will silently produce undefined values in the join. Add a guard: Array.isArray(result.content) ? ... : [].