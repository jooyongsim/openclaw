# OpenClaw Deep Dive: Shell Execution, Planning, Entry Points, Daemon & Agent

---

## Table of Contents

1. [Where Shell Commands Are Actually Executed](#1-where-shell-commands-are-actually-executed)
2. [Where Planning / Reasoning Happens](#2-where-planning--reasoning-happens)
3. [What Are .js and .ts Files?](#3-what-are-js-and-ts-files)
4. [Commander.js and entry.ts Explained](#4-commanderjs-and-entryts-explained)
5. [daemon-cli Explained](#5-daemon-cli-explained)
6. [node-cli Explained](#6-node-cli-explained)
7. [agent.ts Explained](#7-agentts-explained)

---

## 1. Where Shell Commands Are Actually Executed

When the AI agent decides to run a shell command (like `ls`, `git status`, `npm install`, etc.), the command goes through a multi-layered pipeline before it actually touches the OS.

### The Full Pipeline

```
Agent decides to run a command
    ↓
┌─────────────────────────────────────────────┐
│ 1. EXEC TOOL (bash-tools.exec.ts)           │
│    Agent calls the "exec" tool with a       │
│    command string, workdir, env, timeout     │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│ 2. HOST RESOLUTION                          │
│    Where should this command run?            │
│    ├── "sandbox"  → Docker container         │
│    ├── "gateway"  → Host machine             │
│    └── "node"     → Remote node device       │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│ 3. SECURITY CHECK                           │
│    Is this command allowed?                  │
│    ├── deny       → blocked entirely         │
│    ├── allowlist  → only approved commands   │
│    └── full       → all commands allowed     │
│    + Human approval gate (if configured)     │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│ 4. PROCESS SUPERVISOR (supervisor.ts)       │
│    Manages process lifecycle                 │
│    ├── Child Process mode (spawn)            │
│    └── PTY mode (pseudo-terminal)            │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│ 5. ACTUAL OS EXECUTION                      │
│    Node.js child_process.spawn()            │
│    or @lydell/node-pty for terminal mode     │
└─────────────────────────────────────────────┘
```

### Step 1: The Exec Tool

**File:** `src/agents/bash-tools.exec.ts`

The AI agent has access to a tool called `exec`. When the agent decides to run a command, it calls this tool with parameters:

```typescript
// What the agent sends:
{
  command: "git status",       // The actual shell command
  workdir: "/project",         // Working directory
  env: { NODE_ENV: "test" },   // Environment variables
  timeout: 30000,              // Timeout in ms
  pty: true,                   // Use pseudo-terminal?
  background: false,           // Run in background?
}
```

The `createExecTool()` function in this file defines the tool schema and the `execute` function that handles the call.

### Step 2: Three Execution Hosts

The exec tool determines WHERE to run the command:

#### Host A: Sandbox (Docker Container)

**File:** `src/agents/sandbox/docker.ts`

```typescript
// Builds a command like:
spawn("docker", [
  "exec", "-i", "-t",
  "-w", "/workspace",
  "-e", "KEY=VAL",
  "container_name",
  "/bin/sh", "-lc", "git status"
])
```

- Runs inside an isolated Docker container
- Blocked paths: `/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/var/run/docker.sock`
- Dangerous environment variables stripped (PATH, DOCKER_*, etc.)
- The safest execution mode

#### Host B: Gateway (Host Machine)

**File:** `src/agents/bash-tools.exec-host-gateway.ts`

- Runs directly on your machine
- Uses an **allowlist** — only pre-approved command patterns can run
- May require **human approval** before execution (configurable)
- Obfuscation detection prevents tricks like base64-encoded commands

#### Host C: Node (Remote Device)

**File:** `src/agents/bash-tools.exec-host-node.ts`

- Sends the command to a remote node device via the gateway
- Uses the `node.invoke` gateway tool with `system.run.prepare`
- Useful for running commands on companion devices

### Step 3: Security Gates

**File:** `src/agents/bash-tools.exec-approval-request.ts`

Before executing, the system checks:

1. **Security level**: `deny` (blocked), `allowlist` (pre-approved only), or `full` (everything)
2. **Approval mode**: `off` (no approval), `on-miss` (approval if not in allowlist), `always` (always ask)
3. **Approval flow**: Creates a UUID request, notifies via Discord/web UI, waits up to 120s for human approval

### Step 4: Process Supervisor

**File:** `src/process/supervisor/supervisor.ts`

The `ProcessSupervisor` is a singleton that manages ALL process execution. It:
- Creates and tracks running processes
- Manages timeouts (overall + no-output timeout)
- Handles graceful termination (SIGTERM then SIGKILL)
- Streams stdout/stderr back to the agent

### Step 5: Actual OS Execution

Two modes for actually spawning the process:

#### Child Process Mode

**File:** `src/process/supervisor/adapters/child.ts`

```typescript
import { spawn } from "node:child_process";

const child = spawn(command, args, {
  stdio: ["pipe", "pipe", "pipe"],
  cwd: workdir,
  env: environment,
  detached: true,        // POSIX: process group isolation
  windowsHide: true,     // Windows: hide console window
});
```

This is Node.js's built-in `child_process.spawn()` — the standard way to run OS commands from JavaScript.

#### PTY Mode (Pseudo-Terminal)

**File:** `src/process/supervisor/adapters/pty.ts`

```typescript
import pty from "@lydell/node-pty";

const process = pty.spawn(shell, [...shellArgs, command], {
  cwd: workdir,
  env: environment,
  cols: 80,
  rows: 24,
});
```

PTY mode creates a full terminal emulation. This is needed for commands that expect an interactive terminal (like `vim`, `less`, or anything that uses ANSI colors).

### Process Termination

**File:** `src/process/kill-tree.ts`

```typescript
// Unix: Kill the entire process group
process.kill(-pid, "SIGTERM");
// Wait, then force:
process.kill(-pid, "SIGKILL");

// Windows: Use taskkill
spawn("taskkill", ["/T", "/PID", String(pid), "/F"]);
```

### Output Tracking

**File:** `src/agents/bash-process-registry.ts`

Every running command is tracked in a `ProcessSession`:

```typescript
interface ProcessSession {
  id: string;              // Unique ID
  command: string;         // "git status"
  pid: number;             // OS process ID
  aggregated: string;      // All output so far
  tail: string;            // Last N chars
  exitCode: number;        // Exit code when done
  exited: boolean;         // Has it finished?
  backgrounded: boolean;   // Running in background?
}
```

---

## 2. Where Planning / Reasoning Happens

OpenClaw does **not** have a separate "planner" module. Instead, planning emerges from three mechanisms:

### Mechanism 1: The Agent Loop (LLM Decides What To Do)

**File:** `src/agents/pi-embedded-runner/run/attempt.ts` (~1920 lines)

The agent runs in a loop:

```
1. Send prompt + system prompt + tool list to LLM
2. LLM generates a response (may include tool calls)
3. For each tool call:
   a. Validate tool is allowed
   b. Execute tool
   c. Capture result
4. If tool calls were made → go to step 1 (with new context)
5. If no tool calls → done (final response)
```

The LLM (Claude, GPT, etc.) **is** the planner. It sees the available tools, the conversation history, and decides what to do next. There's no separate planning layer — the LLM's reasoning IS the plan.

### Mechanism 2: Extended Thinking (Deeper Reasoning)

**File:** `src/auto-reply/thinking.ts`

The `--thinking` flag controls how much reasoning the LLM does:

| Level | Description |
|-------|-------------|
| `off` | No extended thinking |
| `minimal` | Light reasoning |
| `low` | Basic step-by-step |
| `medium` | Moderate reasoning |
| `high` | Deep multi-step reasoning |
| `xhigh` | Maximum reasoning (only certain models) |
| `adaptive` | System picks based on task complexity |

This maps to provider-specific features:
- **Anthropic Claude**: Native `thinking` parameter (extended thinking blocks)
- **OpenAI o1/o3**: `reasoning_effort` parameter
- **Others**: Provider-specific or ignored

**Thinking visibility** (`src/agents/pi-embedded-runner/thinking.ts`):
- `"off"` — Don't show reasoning
- `"on"` — Show reasoning after completion
- `"stream"` — Stream reasoning as it's generated

**Fallback logic** (`src/agents/pi-embedded-helpers/thinking.ts`):
If a model rejects a thinking level, the system automatically tries lower levels:
`xhigh → high → medium → low → minimal → off`

### Mechanism 3: System Prompt (Instructions That Guide Planning)

**File:** `src/agents/system-prompt.ts` (724 lines)

The system prompt tells the LLM how to plan. Key directives:

```
## Tooling
[Lists all available tools with descriptions]
Tool names are case-sensitive. Call tools exactly as listed.

## Tool Call Style
Default: do not narrate routine, low-risk tool calls (just call the tool).
Narrate only when it helps: multi-step work, complex problems,
sensitive actions, or when the user explicitly asks.

## Skills (mandatory planning step)
Before replying: scan available_skills entries.
- If exactly one skill clearly applies: read its SKILL.md
- If multiple could apply: choose the most specific one
- If none clearly apply: do not read any SKILL.md
Constraint: never read more than one skill up front.

## Safety
You have no independent goals: do not pursue self-preservation,
replication, resource acquisition, or power-seeking.
```

The system prompt includes:
- Identity ("You are a personal assistant running inside OpenClaw")
- Tool list (filtered by policy)
- Skills instructions
- Memory recall instructions
- Messaging instructions
- Runtime info (OS, shell, channel capabilities)
- Owner identity
- Time zone

### Tool Policy Pipeline

**File:** `src/agents/tool-policy-pipeline.ts`

Tools are filtered before the LLM sees them:
1. **Owner-only policies** — Some tools only available to the device owner
2. **Channel-level policies** — Different tools per messaging channel
3. **Agent-level policies** — Per-agent tool restrictions
4. **Subagent depth restrictions** — Spawned agents get fewer tools

### Model Fallback (Recovery Planning)

**File:** `src/agents/model-fallback.ts`

When a model fails, the system "re-plans":
1. Classify failure (auth error, rate limit, context overflow, thinking unsupported, etc.)
2. Try fallback model from configured list
3. On retry: replace prompt with "Continue where you left off..."
4. Prevents duplicate user messages in retries

---

## 3. What Are .js and .ts Files?

### JavaScript (.js)

- **JavaScript** is the programming language that runs in web browsers and Node.js
- `.js` files contain plain JavaScript code
- They can be executed directly by Node.js or browsers
- Example: `openclaw.mjs` (the `.mjs` extension means "ES Module JavaScript")

### TypeScript (.ts)

- **TypeScript** is JavaScript with **type annotations** added on top
- `.ts` files cannot be run directly — they must be **compiled** to `.js` first
- TypeScript catches bugs at compile time (before code runs)
- Example showing the difference:

```javascript
// JavaScript (.js) — no types, errors found at runtime
function add(a, b) {
  return a + b;
}
add("hello", 5);  // Returns "hello5" — no error, just a bug
```

```typescript
// TypeScript (.ts) — types catch bugs at compile time
function add(a: number, b: number): number {
  return a + b;
}
add("hello", 5);  // ERROR: "hello" is not a number ← caught before running
```

### In OpenClaw's Context

```
Source code (what developers write):
  src/entry.ts
  src/cli/program/build-program.ts
  src/commands/agent.ts
  ... all .ts files

     ↓ pnpm build (TypeScript compiler)

Compiled output (what Node.js runs):
  dist/entry.js
  dist/cli/program/build-program.js
  dist/commands/agent.js
  ... all .js files

     ↓ Node.js executes

Running program
```

The `openclaw.mjs` bootstrap file is plain JavaScript because it needs to run BEFORE the TypeScript compiler output is loaded. It checks the Node.js version and then loads `dist/entry.js` (the compiled version of `src/entry.ts`).

---

## 4. Commander.js and entry.ts Explained

### What is Commander.js?

**Commander.js** is a popular Node.js library for building command-line interfaces (CLIs). It handles:
- Parsing command-line arguments (`openclaw gateway --port 8080`)
- Defining commands and subcommands
- Generating help text automatically
- Option validation

Think of it as the "framework" that turns your code into a proper CLI tool, similar to how Express turns your code into a web server.

**Example:**

```typescript
import { Command } from "commander";

const program = new Command();

program
  .command("gateway")                           // "openclaw gateway"
  .description("Run the WebSocket Gateway")
  .option("--port <port>", "Listen port")       // "--port 8080"
  .option("--verbose", "Enable verbose logging")
  .action((opts) => {
    console.log(`Starting gateway on port ${opts.port}`);
    // ... actual gateway logic
  });

program.parse(process.argv);  // Parse "openclaw gateway --port 8080"
```

When a user types `openclaw gateway --port 8080`, Commander.js:
1. Matches `gateway` to the registered command
2. Extracts `--port 8080` as `opts.port = "8080"`
3. Calls the `.action()` callback with the parsed options

### The Complete Entry Point Chain

Here's what happens from the moment you type `openclaw` to the program running:

```
You type: openclaw agent --message "Hello" --thinking high
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 1: openclaw.mjs (Bootstrap - plain JavaScript)│
│                                                     │
│  • Checks Node.js version (requires v22.12+)        │
│  • Enables module compile cache for faster startup   │
│  • Loads dist/entry.js (compiled TypeScript)          │
│  • If dist/ missing → error "run pnpm build first"   │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 2: src/entry.ts (Main Entry - TypeScript)     │
│                                                     │
│  • Sets process.title = "openclaw"                  │
│  • Suppresses experimental warnings                 │
│  • Handles --version and --help fast paths           │
│  • Parses --dev and --profile flags                  │
│  • Calls runCli() from run-main.ts                  │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 3: src/cli/run-main.ts (CLI Runner)           │
│                                                     │
│  • Normalizes Windows argv (path separators)        │
│  • Loads .env files                                  │
│  • Normalizes environment variables                  │
│  • Calls buildProgram() to create Commander program  │
│  • Registers all commands                            │
│  • Installs error handlers                           │
│  • Calls program.parseAsync(argv) → Commander runs   │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 4: src/cli/program/build-program.ts           │
│                                                     │
│  const program = new Command();  ← Commander.js     │
│  createProgramContext();         ← version, channels │
│  configureProgramHelp();         ← help system       │
│  registerPreActionHooks();       ← banner, plugins   │
│  registerProgramCommands();      ← all subcommands   │
│  return program;                                     │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 5: Commander parses argv and runs command     │
│                                                     │
│  "agent" matched → loads src/commands/agent.ts      │
│  --message "Hello" → opts.message = "Hello"         │
│  --thinking high → opts.thinking = "high"           │
│  action() callback fires → agent runs               │
└─────────────────────────────────────────────────────┘
```

### Why Two Entry Points?

- **`openclaw.mjs`** (Stage 1) is plain JavaScript so it can run without compilation. Its job is just to check prerequisites and load the compiled code.
- **`src/entry.ts`** (Stage 2) is TypeScript with the real logic. It's compiled to `dist/entry.js` by `pnpm build`.

This two-stage pattern is common in large TypeScript projects — you need a tiny JS bootstrap to load the compiled TS output.

### Lazy Command Loading (Performance Optimization)

**File:** `src/cli/program/register.subclis.ts`

OpenClaw has many commands, but loading ALL of them on startup would be slow. Instead:

```typescript
// Commands are registered as lightweight placeholders:
{
  name: "gateway",
  description: "Run the WebSocket Gateway",
  register: async (program) => {
    // Only imports the real code when "gateway" is actually invoked
    const mod = await import("../gateway-cli.js");
    mod.registerGatewayCli(program);
  },
}
```

If you run `openclaw agent ...`, the gateway code is never loaded. This makes startup fast.

### Pre-Action Hooks

**File:** `src/cli/program/preaction.ts`

Before ANY command runs, hooks fire:

1. **Set process title** (so `ps` shows meaningful names)
2. **Emit CLI banner** (version + tagline)
3. **Load plugin registry** (if command needs channels)
4. **Validate config** (blocks commands with broken config, except `doctor`)
5. **Set log levels**

---

## 5. daemon-cli Explained

### What is the Daemon?

The "daemon" is the **Gateway service running as a background process**. Instead of running `openclaw gateway` in a terminal and keeping it open, the daemon installs it as a **system service** that starts automatically and runs continuously.

Think of it like how Spotify runs in the background — you don't need a terminal window open.

### Platform-Specific Service Types

| Platform | Service Type | Config File |
|----------|-------------|-------------|
| **macOS** | LaunchAgent (launchd) | `~/Library/LaunchAgents/ai.openclaw.gateway.plist` |
| **Linux** | systemd user service | `~/.config/systemd/user/openclaw-gateway.service` |
| **Windows** | Scheduled Task (schtasks) | `OpenClaw Gateway` task |

### Daemon CLI Commands

**File:** `src/cli/daemon-cli/register.ts`

```bash
openclaw daemon install    # Install as system service
openclaw daemon start      # Start the service
openclaw daemon stop       # Stop the service
openclaw daemon restart    # Restart the service
openclaw daemon status     # Show service status + health
openclaw daemon uninstall  # Remove the system service
```

### Installation Pipeline

**File:** `src/cli/daemon-cli/install.ts`

When you run `openclaw daemon install`:

```
1. Validate port and runtime (node or bun)
2. Check if service already exists
3. Generate auth token (if not set)
4. Build service definition:
   - Program: /path/to/node openclaw gateway run --port 18789
   - Working directory: ~/.openclaw
   - Environment: OPENCLAW_PROFILE, etc.
5. Write platform-specific service file
6. Load/enable the service
```

### Health Monitoring

**File:** `src/cli/daemon-cli/restart-health.ts`

After restart, the daemon CLI monitors health:
1. Check if process is running (by PID)
2. Check if port is listening
3. Probe the gateway via WebSocket
4. Detect stale PIDs from crashed instances
5. Wait up to 60 seconds for healthy state

### Status Diagnostics

**File:** `src/cli/daemon-cli/status.gather.ts`

`openclaw daemon status` collects:
- Service load state (loaded/enabled/running)
- PID and exit status
- Port listening status
- Config audit (drift detection between service config and user config)
- RPC probe (can it actually respond?)
- TLS configuration
- Conflicting services
- Last error from logs

---

## 6. node-cli Explained

### What is a Node?

A "node" in OpenClaw is **not** Node.js. It's a **headless companion device** that connects to your Gateway and extends the agent's capabilities. Think of it as a "remote arm" for the agent.

**Use cases:**
- Run agents on a remote server while controlling from your laptop
- Connect a Raspberry Pi as a node for home automation
- Multi-machine setup where different nodes have different capabilities (camera, microphone, screen)

### How Nodes Work

```
┌──────────────┐         ┌──────────────┐
│  Your Laptop │         │ Remote Server│
│              │         │              │
│  Gateway     │◄───────►│  Node Host   │
│  (control    │  WebSocket  (runs agent │
│   plane)     │         │   commands)  │
└──────────────┘         └──────────────┘
```

The node connects TO the gateway (not the other way around). It registers itself and can then receive commands from the agent.

### Node CLI Commands

**File:** `src/cli/node-cli/register.ts`

```bash
openclaw node run         # Run node host in foreground
openclaw node install     # Install as system service
openclaw node status      # Show node service status
openclaw node stop        # Stop node service
openclaw node restart     # Restart node service
openclaw node uninstall   # Remove node service
```

### Node vs Daemon Differences

| Feature | Daemon (Gateway) | Node |
|---------|-------------------|------|
| Role | Central control plane | Remote executor |
| Service name | `ai.openclaw.gateway` | `ai.openclaw.node` |
| Config persistence | Ephemeral | Persisted to disk |
| TLS fingerprint | N/A | Supported (for remote gateways) |
| Node ID | N/A | Configurable, clears pairing token on change |
| Connection direction | Listens (server) | Connects (client) |

### Node Installation

**File:** `src/cli/node-cli/daemon.ts`

```bash
openclaw node install \
  --host gateway.example.com \
  --port 18789 \
  --tls \
  --tls-fingerprint ABC123 \
  --node-id "kitchen-pi" \
  --display-name "Kitchen Raspberry Pi"
```

This creates a system service that:
1. Connects to the specified gateway
2. Registers with the given node ID
3. Stays connected and waits for commands
4. Auto-restarts on crash

---

## 7. agent.ts Explained

### What is agent.ts?

**File:** `src/commands/agent.ts` (983 lines)

This is the **most important command file** in OpenClaw. When you send a message to the assistant — whether via CLI, WhatsApp, Telegram, Discord, or any channel — it eventually calls the logic in this file.

```bash
openclaw agent --message "What's the weather?" --thinking high
```

### The Two Execution Paths

agent.ts has two fundamentally different ways to run the AI:

```
                    ┌──────────────┐
                    │  agent.ts    │
                    │  (983 lines) │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
    ┌─────────▼─────────┐    ┌─────────▼──────────┐
    │  CLI Provider      │    │  Embedded PI Agent  │
    │  (runCliAgent)     │    │  (runEmbeddedPi     │
    │                    │    │   Agent)             │
    │  Spawns external   │    │  Runs AI in-process  │
    │  CLI process       │    │  with full tool      │
    │  (e.g. claude-cli) │    │  access              │
    └────────────────────┘    └──────────────────────┘
```

#### Path 1: CLI Provider

For providers like `claude-cli` (Anthropic's CLI tool):
- Spawns an external process
- Passes the message via command-line arguments
- Manages session IDs across invocations
- Handles session expiration with automatic retry

#### Path 2: Embedded PI Agent

For cloud API providers (Anthropic API, OpenAI, Google, etc.):
- Runs the agent in-process using the PI framework (`@mariozechner/pi-agent-core`)
- Full control over tools, auth, thinking levels
- Supports streaming, skills, memory, browser tools

### Complete Execution Flow

```
1. VALIDATE INPUT
   ├── Message provided?
   ├── Agent ID valid?
   └── Session specified?

2. RESOLVE CONFIGURATION
   ├── Load config from ~/.openclaw/config.json
   ├── Resolve secret references via gateway
   └── Load model catalog

3. RESOLVE SESSION
   ├── Find or create session by key
   ├── Load persisted settings (model, thinking, verbose)
   ├── Determine session scope (per-sender / per-thread / global)
   └── Ensure session file exists on disk

4. BUILD SKILLS SNAPSHOT
   ├── Scan workspace skills directory
   ├── Build metadata for each skill
   └── Cache in session store

5. RESOLVE MODEL + AUTH
   ├── Default provider/model from config
   ├── Session-level overrides
   ├── Validate against allowlist
   ├── Resolve auth profile
   └── Determine thinking level support

6. EXECUTE AGENT
   ├── CLI Provider → spawn external process
   └── Embedded PI → run in-process with tools

7. MODEL FALLBACK (if execution fails)
   ├── Classify failure reason
   ├── Try next model in fallback list
   └── Retry with "Continue where you left off..."

8. DELIVER RESULT (if --deliver flag)
   ├── Resolve delivery target (phone/channel)
   ├── Format outbound payload
   └── Send via channel plugin

9. PERSIST SESSION
   ├── Save thinking/verbose overrides
   ├── Save model/auth overrides
   ├── Update session file reference
   └── Write session store to disk
```

### Key Options

```
--message <text>          The user's prompt (required)
--agent-id <id>           Which agent to use
--to <phone>              Send to phone number (E.164 format)
--session-id <id>         Resume specific session
--session-key <key>       Session routing key
--thinking <level>        off|minimal|low|medium|high|xhigh|adaptive
--thinking-once <level>   One-shot thinking (doesn't persist)
--verbose <level>         on|full|off
--json                    Output as JSON
--timeout <seconds>       Execution timeout
--deliver                 Send result back to messaging channel
--channel <name>          Which channel to deliver to
--thread-id <id>          Reply in specific thread
--extra-system-prompt     Additional instructions for the agent
--images <paths>          Attach images (multimodal)
```

### Session Management

Sessions track conversation state across interactions:

```typescript
interface SessionEntry {
  sessionId: string;           // Unique ID
  updatedAt: number;           // Last updated timestamp

  // What model/provider to use
  modelOverride?: string;
  providerOverride?: string;

  // Authentication
  authProfileOverride?: string;

  // Execution preferences
  thinkingLevel?: ThinkLevel;
  verboseLevel?: VerboseLevel;

  // State
  skillsSnapshot?: WorkspaceSkillSnapshot;
  sessionFile?: string;        // Path to transcript file

  // Delivery context
  channel?: string;            // "whatsapp", "telegram", etc.
  chatType?: string;           // "dm", "group"
}
```

**Session scopes:**
- `per-sender` (default) — One session per phone number / user
- `per-thread` — One session per chat thread
- `global` — Single shared session for all conversations

### Model Fallback System

When a model fails, agent.ts automatically tries alternatives:

```
Primary model: claude-3.5-sonnet
    ↓ fails (rate limit)
Fallback 1: claude-3-haiku
    ↓ fails (auth error)
Fallback 2: gpt-4
    ↓ succeeds
Agent runs with gpt-4
```

On retry, the prompt is replaced with "Continue where you left off..." to avoid sending the user's message twice.

### Delivery System

After the agent produces a response, if `--deliver` is set:

```
Agent produces response
    ↓
Resolve delivery target
    ├── --reply-to override → use that
    └── Original --to from session → use that
    ↓
Resolve delivery channel
    ├── --reply-channel override → use that
    └── Original channel from session → use that
    ↓
Format outbound payload (text + media)
    ↓
Call channel plugin to send
    ↓
Message appears in WhatsApp/Telegram/Discord/etc.
```
