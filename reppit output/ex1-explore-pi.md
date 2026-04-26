# pi-mono: pi-mom & pi-pods — Exploration Brief

## Overview

This brief covers two packages that handle the outermost integration surfaces of the monorepo:

- **pi-mom** (`packages/mom`) — A self-managing Slack bot powered by Claude Sonnet. Listens for @mentions via Socket Mode, runs bash/file tools in a Docker sandbox, schedules events (cron, one-shot, immediate), and autonomously creates CLI "skills" for recurring workflows.
- **pi-pods** (`packages/pods`) — A CLI tool (`pi`) that provisions vLLM on remote GPU pods over SSH, manages GPU allocation, starts/stops models, and exposes OpenAI-compatible HTTP endpoints.

Neither package is exercised by any existing tests; both are integration-heavy and operator-run tools.

---

## pi-mom

### Purpose

Mom is a Claude-backed Slack agent. Each Slack channel gets its own isolated conversation history, memory, and tool directory. She installs her own dependencies (inside a Docker container), writes reusable scripts ("skills"), and can wake herself up on a cron schedule. The design is explicitly minimal: mom builds what she needs rather than shipping pre-built integrations.

### File & Directory Structure

```
packages/mom/src/
  main.ts          Entry point; arg parsing, ChannelState map, SlackContext adapter, MomHandler
  agent.ts         AgentRunner factory; system prompt builder; event-loop subscriber
  slack.ts         SlackBot (Socket Mode); message logging, backfill, channel/user registry
  context.ts       SessionManager wrapper; log→context sync (syncLogToSessionManager)
  store.ts         ChannelStore; attachment downloads from Slack
  events.ts        EventsWatcher; fs-watch on data/events/; croner for periodic events
  sandbox.ts       Docker/host command executor; path translation container↔host
  fs-watch.ts      Resilient fs.watch wrapper with auto-retry
  log.ts           Structured console logging
  tools/
    bash.ts        Executes shell commands via sandbox executor
    read.ts        Read file contents
    write.ts       Create/overwrite files
    edit.ts        Surgical string replacement
    attach.ts      Upload file to Slack channel
    index.ts       Assembles tool array; setUploadFunction injection point
```

**Data directory layout** (operator-controlled, outside source):
```
data/
  MEMORY.md                 Global memory (all channels)
  SYSTEM.md                 Environment modification log
  settings.json             Compaction and retry config
  skills/                   Global custom CLI tools
  events/                   Scheduled event JSON files
  <channelId>/
    log.jsonl               Source-of-truth message history (append-only)
    context.jsonl           LLM context (synced + compacted from log)
    MEMORY.md               Channel-specific memory
    attachments/            Slack file downloads
    scratch/                Mom's working directory
    skills/                 Channel-specific CLI tools
    last_prompt.jsonl       Debug dump of last LLM request
```

### Key Entry Points

| File | Function | What it does |
|------|----------|--------------|
| [main.ts](packages/mom/src/main.ts#L53) | `parseArgs()` | CLI arg parsing for `--sandbox` and `--download` |
| [main.ts](packages/mom/src/main.ts#L95) | `getState(channelId)` | Lazy-init per-channel `ChannelState` (runner + store) |
| [main.ts](packages/mom/src/main.ts#L114) | `createSlackContext()` | Adapts `SlackEvent` into the `SlackContext` the agent runner expects; manages message accumulation, thread posting, truncation |
| [main.ts](packages/mom/src/main.ts#L299) | `handler.handleEvent()` | Core dispatch: sets running state, runs agent, handles abort/silent/error outcomes |
| [agent.ts](packages/mom/src/agent.ts#L398) | `getOrCreateRunner()` | Returns cached `AgentRunner` per channel (one runner lives for the process lifetime) |
| [agent.ts](packages/mom/src/agent.ts#L411) | `createRunner()` | Wires sandbox executor → tools → `AgentSession` → event subscriber; one-time setup per channel |
| [agent.ts](packages/mom/src/agent.ts#L641) | `AgentRunner.run()` | Per-message execution: syncs log→context, rebuilds system prompt, calls `AgentSession.prompt()`, drains Slack message queue |
| [agent.ts](packages/mom/src/agent.ts#L141) | `buildSystemPrompt()` | Constructs the full system prompt injecting memory, channel/user IDs, skills, event format, workspace layout |
| [events.ts](packages/mom/src/events.ts#L44) | `EventsWatcher` | fs.watch on `data/events/`; parses JSON event files; schedules `setTimeout` (one-shot) or `croner` (periodic); fires synthetic SlackEvents |

### Call Graph

```
main.ts
  SlackBot.start()                          [slack.ts — Socket Mode connection]
    on @mention / DM → handler.handleEvent()
      createSlackContext()                  [message accumulator + Slack API calls]
      AgentRunner.run()
        syncLogToSessionManager()           [context.ts — backfill missed messages]
        sessionManager.buildSessionContext()
        buildSystemPrompt()                 [fresh memory + skills + channel/user IDs]
        AgentSession.prompt()               [pi-coding-agent]
          Agent.step()                      [pi-agent-core]
            LLM call → Anthropic API        [pi-ai — claude-sonnet-4-5, hardcoded]
            tool_execution_start/end events
              createMomTools(executor)      [tools/index.ts → sandbox.ts]
        event subscriber                    [respond/respondInThread/replaceMessage]
  EventsWatcher.start()                     [events.ts]
    fs.watch(data/events/) → handleFile()
      immediate → execute() immediately
      one-shot  → setTimeout()
      periodic  → new Cron()               [croner]
      → slack.enqueueEvent() → handleEvent() (synthetic)
```

### External Libraries / Services

- **`@mariozechner/pi-agent-core`** — `Agent`, `AgentEvent` types, agentic loop
- **`@mariozechner/pi-ai`** — `getModel()` for Anthropic model descriptor
- **`@mariozechner/pi-coding-agent`** — `AgentSession`, `SessionManager`, `AuthStorage`, `ModelRegistry`, `loadSkillsFromDir`, `formatSkillsForPrompt`, `convertToLlm`
- **Slack Bolt / Socket Mode SDK** — real-time Slack events
- **`croner`** — cron scheduling for periodic events
- **Docker** (external) — sandbox execution environment; mom runs commands via `docker exec`

### Data Flow

1. Slack message arrives → `SlackEvent` → `handleEvent()`
2. `log.jsonl` is written (raw message log)
3. `syncLogToSessionManager` replays any unseen entries from `log.jsonl` into `context.jsonl`
4. `context.jsonl` is loaded as `agent.state.messages`
5. System prompt rebuilt from MEMORY.md files + skills
6. `AgentSession.prompt(userMessage)` → Anthropic API
7. Tool calls dispatched via sandbox executor (Docker exec or host shell)
8. Tool args/results posted to Slack thread; text parts posted to main message
9. Context exceeds limit → `AgentSession` triggers compaction (summary inserted into `context.jsonl`)
10. Final text replaces the "Thinking..." placeholder message in Slack
11. If `[SILENT]` returned, the message and thread are deleted

### Setup Requirements

```bash
export MOM_SLACK_APP_TOKEN=xapp-...
export MOM_SLACK_BOT_TOKEN=xoxb-...
export ANTHROPIC_API_KEY=sk-ant-...   # or link auth.json from pi-coding-agent /login

# Recommended: Docker sandbox
docker run -d --name mom-sandbox -v $(pwd)/data:/workspace alpine tail -f /dev/null
npx mom --sandbox=docker:mom-sandbox ./data
```

---

## pi-pods

### Purpose

`pi` is a CLI to deploy and manage LLMs on remote GPU pods (DataCrunch, RunPod, Vast.ai, etc.) entirely over SSH. It handles first-time vLLM installation, model-specific vLLM argument presets (tool-call parsers, tensor parallelism), GPU assignment across multiple models, and log streaming. Each model runs as a detached `setsid` process on the pod, logging to `~/.vllm_logs/<name>.log`.

### File & Directory Structure

```
packages/pods/src/
  cli.ts                 Entry point; routes all subcommands
  config.ts              Load/save ~/.pi/pods.json; getActivePod
  types.ts               GPU, Model, Pod, Config interfaces
  model-configs.ts       Predefined vLLM args per model (Qwen, GPT-OSS, GLM, etc.)
  models.json            JSON catalog of known models with GPU requirements
  ssh.ts                 sshExec (buffered), sshExecStream (piped), scpFile
  commands/
    pods.ts              setupPod, listPods, switchActivePod, removePodCommand
    models.ts            startModel, stopModel, stopAllModels, listModels, viewLogs, showKnownModels
    prompt.ts            promptModel (agent chat against a running vLLM endpoint)

packages/pods/scripts/   (SCP'd to pods at runtime)
  pod_setup.sh           Installs vLLM, sets env vars, configures HF token
  model_run.sh           Template script to launch vLLM; placeholders replaced before upload
```

**Config state** (`~/.pi/pods.json`):
```json
{
  "active": "dc1",
  "pods": {
    "dc1": {
      "ssh": "ssh root@1.2.3.4",
      "gpus": [{"id": 0, "name": "NVIDIA H100", "memory": "80 GiB"}],
      "models": {
        "qwen": {"model": "Qwen/Qwen2.5-Coder-32B-Instruct", "port": 8001, "gpu": [0], "pid": 12345}
      },
      "modelsPath": "/mnt/hf-models",
      "vllmVersion": "release"
    }
  }
}
```

### Key Entry Points

| File | Function | What it does |
|------|----------|--------------|
| [cli.ts](packages/pods/src/cli.ts#L57) | top-level | Parses `command`/`subcommand`/`--pod` and dispatches to the right handler |
| [commands/pods.ts](packages/pods/src/commands/pods.ts#L44) | `setupPod()` | SSH-tests connection, SCPs `pod_setup.sh`, runs it with HF_TOKEN + PI_API_KEY, queries `nvidia-smi`, saves pod to config |
| [commands/models.ts](packages/pods/src/commands/models.ts#L78) | `startModel()` | Selects GPUs, looks up model preset, injects vars into `model_run.sh`, uploads via SSH heredoc, launches via `setsid`, streams startup logs |
| [commands/models.ts](packages/pods/src/commands/models.ts#L36) | `getNextPort()` | Finds first unused port ≥ 8001 across running models |
| [commands/models.ts](packages/pods/src/commands/models.ts#L48) | `selectGPUs()` | Round-robin GPU assignment by current usage count |
| [commands/models.ts](packages/pods/src/commands/models.ts#L418) | `stopModel()` | `pkill -P <pid>` + `kill <pid>` via SSH; removes from config |
| [commands/models.ts](packages/pods/src/commands/models.ts#L481) | `listModels()` | Lists from config; verifies live by SSHing `curl /health` + log grep |
| [model-configs.ts](packages/pods/src/model-configs.ts) | `getModelConfig()` | Returns `{args, env, notes}` for a known model+GPU-count combination |
| [config.ts](packages/pods/src/config.ts#L19) | `loadConfig()` | Reads `~/.pi/pods.json`; returns empty config if absent |
| [ssh.ts](packages/pods/src/ssh.ts) | `sshExec/Stream/scpFile` | Thin wrappers around `child_process.spawn("ssh"/"scp")`; no SSH npm library |

### Call Graph

```
cli.ts
  pods setup → setupPod()
    sshExec("echo SSH OK")             — connection test
    scpFile(pod_setup.sh)              — copy installer
    sshExecStream(pod_setup.sh ...)    — install vLLM (2-5 min, streaming)
    sshExec("nvidia-smi ...")          — detect GPUs
    addPod() → saveConfig()            — write ~/.pi/pods.json

  start <model> → startModel()
    getModelConfig()                   — model-configs.ts preset lookup
    selectGPUs()                       — round-robin allocation
    sshExec(heredoc model_run.sh)      — upload customized launch script
    sshExec(setsid wrapper)            — detached launch, capture PID
    spawn("ssh", [..., "tail -f ..."])  — stream startup logs locally
      detect "Application startup complete" → success path
      detect OOM / RuntimeError        → failure path, remove from config

  stop [name] → stopModel() / stopAllModels()
    sshExec("pkill -P <pid>; kill <pid>")
    delete from config, saveConfig()

  list → listModels()
    for each model: sshExec(ps + curl /health + tail log)

  agent <name> → promptModel()        — commands/prompt.ts
    pi-agent-core over HTTP to vLLM endpoint (OpenAI-compat)
```

### External Libraries / Services

- **SSH / SCP** — raw `child_process.spawn`, no npm SSH library; pod must allow SSH root login
- **vLLM** — runs on the remote pod, exposes `/v1/chat/completions` and `/v1/responses`
- **HuggingFace Hub** — model weights downloaded on the pod using `HF_TOKEN`
- **`@mariozechner/pi-agent-core`** — used by `promptModel` for local agent loop against remote vLLM
- **`chalk`** — terminal color output

### Data Flow

**Pod setup:**
1. `pi pods setup` → test SSH → SCP `pod_setup.sh` → run (installs vLLM, writes `~/.vllm_env`)
2. Query `nvidia-smi` → persist `GPU[]` to `~/.pi/pods.json`

**Model start:**
1. `pi start` → lookup preset in `model-configs.ts` / `models.json`
2. Select GPU IDs (round-robin), next free port
3. Inject `MODEL_ID`, `PORT`, `VLLM_ARGS` into `model_run.sh` template
4. Upload via SSH heredoc → `setsid` launch (detached, logs to `~/.vllm_logs/<name>.log`)
5. `spawn` tail -f → wait for `Application startup complete` in log stream
6. Save `{model, port, gpu[], pid}` to config

**Model stop:**
1. Read PID from config → SSH `pkill -P <pid>; kill <pid>`
2. Remove model entry from config

### Setup Requirements

```bash
export HF_TOKEN=hf_...          # HuggingFace token for model downloads
export PI_API_KEY=any-string    # Becomes the vLLM API key

# GPU pod prerequisites:
#   Ubuntu 22.04/24.04, SSH root access, NVIDIA drivers + CUDA installed

pi pods setup dc1 "ssh root@1.2.3.4" \
  --mount "sudo mount -t nfs ... /mnt/hf-models"

pi start Qwen/Qwen2.5-Coder-32B-Instruct --name qwen
```

---

## Cross-Package Relationship

`pi-mom` and `pi-pods` are independent packages with no direct imports between them. They share upstream dependencies:

- Both use `@mariozechner/pi-agent-core` for agentic execution
- `pi-pods` exposes vLLM endpoints that `pi-mom` could target if its hardcoded model were changed (currently `claude-sonnet-4-5` via Anthropic API)
- Both write their state to `~/.pi/` by default (`pods.json` for pods, `mom/auth.json` for mom auth)
