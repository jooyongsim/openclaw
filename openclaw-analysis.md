# OpenClaw Codebase Analysis

## Overview

**OpenClaw** is a self-hosted, multi-channel AI assistant gateway. You run it on your own devices and it connects to the messaging platforms you already use — WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, Microsoft Teams, and 15+ more. The product is the assistant; the gateway is the control plane.

**Version:** 2026.3.3 | **Language:** TypeScript (ESM) | **Runtime:** Node 22+ | **Package Manager:** pnpm (monorepo)

---

## Architecture Overview

```
                    ┌──────────────────────────────┐
                    │         CLI (openclaw)        │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │     Gateway (HTTP + WebSocket)│ ◄── Web UI (Lit.js)
                    │  sessions, presence, cron,    │
                    │  config, webhooks, canvas     │
                    └──┬──────────┬────────────┬───┘
                       │          │            │
              ┌────────▼──┐  ┌───▼────┐  ┌────▼──────┐
              │  Routing   │  │ Agents │  │  Plugins  │
              │ (account   │  │ (Pi    │  │ (SDK +    │
              │  mapping)  │  │ runtime│  │  hooks)   │
              └────┬───────┘  └───┬────┘  └───────────┘
                   │              │
    ┌──────────────▼──────────────▼──────────────────┐
    │           Channels (unified abstraction)        │
    ├─────┬──────┬────────┬───────┬────────┬─────────┤
    │Slack│Discord│Telegram│Signal │WhatsApp│ 20 more │
    └─────┴──────┴────────┴───────┴────────┴─────────┘
```

---

## Key Directories

| Directory | Purpose | Scale |
|-----------|---------|-------|
| `src/` | Core source code — CLI, gateway, agents, channels, routing, plugins | ~2,000+ TS files |
| `extensions/` | Channel plugins (Matrix, MS Teams, IRC, Zalo, Twitch, etc.) | **40 extensions** |
| `skills/` | Built-in AI tools (GitHub, Notion, Spotify, coding-agent, etc.) | **52 skills** |
| `apps/` | Native companion apps | iOS (Swift), Android (Kotlin), macOS (Swift) |
| `ui/` | Web control dashboard | Lit.js + Vite |
| `packages/` | Compatibility shims (`clawdbot`, `moltbot`) | 2 packages |
| `docs/` | Mintlify-hosted documentation | |

---

## Core Modules (inside `src/`)

### 1. Gateway (`src/gateway/`)

The central server (219 files). Manages sessions, client connections, protocol schemas, and RPC methods. Boots via `boot.ts`, exposes HTTP + WebSocket endpoints.

### 2. Agents (`src/agents/`)

The AI execution engine (484 files). Built on the `@mariozechner/pi-*` agent framework. Supports embedded agent runners, sandboxed execution, auth profiles, and bash/skill tools.

### 3. Channels (`src/channels/`)

Unified abstraction over 25+ messaging platforms. Each channel normalizes inbound messages into a common format. Core channels (Telegram, Discord, Slack, Signal, iMessage, WhatsApp) live in `src/`; additional channels are in `extensions/`.

### 4. Routing (`src/routing/`)

Maps inbound messages to the correct agent/session. Handles account lookup, multi-agent routing, and peer isolation.

### 5. CLI (`src/cli/` + `src/commands/`)

Commander.js-based CLI with subcommands: `gateway`, `agent`, `channels`, `models`, `onboard`, `doctor`, `send`, etc.

### 6. Plugin System (`src/plugin-sdk/` + `src/plugins/`)

Extensible plugin architecture. The SDK provides channel-specific APIs; discovery, loading, and hook execution handle the lifecycle.

### 7. Config & Secrets (`src/config/` + `src/secrets/`)

Configuration loading, session storage/persistence, and credential management.

### 8. Specialized Features

- `src/browser/` — Browser automation
- `src/canvas-host/` — Agent-driven visual canvas (A2UI)
- `src/cron/` — Scheduled tasks
- `src/tts/` — Text-to-speech (ElevenLabs + system TTS)
- `src/media/` + `src/media-understanding/` — Image/video/audio pipeline
- `src/memory/` — Vector memory and semantic search
- `src/acp/` — Agent Control Protocol (control-plane + runtime)

---

## Entry Point Flow

1. **`openclaw.mjs`** — Checks Node version (>=22.12), enables module compile cache, dynamically imports `dist/entry.js`
2. **`src/entry.ts`** — CLI wrapper with respawn logic, profile parsing, env setup
3. **`src/cli/program/build-program.ts`** — Builds the Commander.js program and registers all commands
4. **`openclaw gateway`** — Boots the HTTP/WS server, initializes channels, starts agents
5. **`openclaw agent`** — Runs the Pi agent runtime with tool streaming

---

## Key Dependencies

| Category | Dependencies |
|----------|-------------|
| Agent Framework | `@mariozechner/pi-*` (agent, TUI, coding-agent) |
| Channels | `discord.js`, `@slack/bolt`, `grammy` (Telegram), `matrix-js-sdk` |
| LLM Providers | OpenAI, Anthropic, Google, and more via provider abstraction |
| Web Server | Express 5.2 |
| Web UI | Lit.js 3.3 |
| Media | Sharp (image processing), pdfjs-dist |
| Storage | SQLite-vec (vector search) |
| Build | tsdown, Vite, Vitest (8 test configs, 70% coverage threshold) |

---

## Extensions (40 total)

### Messaging Platforms

Slack, Discord, Telegram, Signal, WhatsApp, LINE, MS Teams, Matrix, Mattermost, Google Chat, Feishu, Nextcloud Talk, Synology Chat, Tlon, Nostr, IRC, Zalo, Zalo Personal, Twitch, BlueBubbles (iMessage), native iMessage

### Specialized Integrations

- `voice-call` — Voice calling
- `talk-voice` — Talk voice features
- `device-pair` — Device pairing
- `copilot-proxy` — Microsoft Copilot proxy
- `memory-core` / `memory-lancedb` — Memory and vector storage
- `llm-task` — LLM task execution
- `diagnostics-otel` — OpenTelemetry diagnostics

---

## Skills (52 total)

### Productivity

1Password, Apple Notes, Apple Reminders, Bear Notes, Notion, Obsidian, Trello, Things (Mac), Himalaya (email)

### Code & Development

GitHub, GitHub Issues, Coding Agent

### Media & Creative

Canvas, Video Frames, GIF Search, Camera Snapshots, OpenAI Image Gen, OpenAI Whisper

### Device & Home Control

Spotify Player, Sonos CLI, OpenHue, Peekaboo, Eightctl, Ordercli

### System & Analytics

Healthcheck, Model Usage, Summarize, Session Logs, Blog Watcher, Skill Creator

---

## Companion Apps

| Platform | Language | Key Features |
|----------|----------|-------------|
| **iOS** | Swift | Calendar, Camera, Chat, Contacts, Gateway, Location, Media, Voice, Reminders, Settings |
| **Android** | Kotlin | Gradle-based, companion app |
| **macOS** | Swift | Menu bar app, OpenClawMacCLI, OpenClawProtocol |

---

## Build & Development

```bash
# Install dependencies
pnpm install

# Build UI + TypeScript
pnpm ui:build
pnpm build

# Run gateway (dev mode with auto-reload)
pnpm gateway:watch

# Run tests
pnpm test              # Vitest parallel suite
pnpm test:coverage     # With V8 coverage (70% threshold)

# Lint & format
pnpm check             # Oxlint + Oxfmt
pnpm format:fix        # Auto-fix formatting

# Type checking
pnpm tsgo
```

---

---

# Deep Dive: `src/memory/`

## Overview

The memory module (~91 TypeScript files, ~18,070 lines) provides **semantic memory and search** for the AI agent. It indexes markdown files and session transcripts, embeds them into vectors, and supports hybrid search (vector similarity + full-text keyword search).

## Architecture Diagram

```
┌─────────────────────────────────────────────┐
│         MemorySearchManager API             │
│  search() readFile() sync() status()        │
└────────────────┬────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
   [Builtin]         [QMD Daemon]
   MemoryIndexManager  QmdMemoryManager
        │                 │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │    SQLite DB     │
        ├─────────────────┤
        │ files           │  Track memory files
        │ chunks          │  Chunked text + embeddings
        │ embedding_cache │  LRU embedding cache
        │ chunks_fts      │  Full-text search (FTS5)
        │ chunks_vec      │  Vector index (sqlite-vec)
        │ meta            │  Index metadata
        └─────────────────┘

   Dual Search Path:
   ┌──────────────────────┐
   │  Hybrid Search       │
   ├──────────────────────┤
   │ Vector Search (0.7)  │  Embeddings + cosine similarity
   │ + Keyword Search(0.3)│  FTS5 + BM25 ranking
   │ → Temporal Decay     │  For dated memories
   │ → MMR Reranking      │  For diversity
   └──────────────────────┘

   Embedding Providers:
   ┌──────────────────────┐
   │ OpenAI (batch)       │  text-embedding-3-small
   │ Gemini (batch)       │  gemini-embedding-001
   │ Voyage (batch)       │  voyage-3
   │ Mistral (batch)      │  mistral-embed
   │ Ollama (local)       │  nomic-embed-text
   │ Local GGUF (llama)   │  embeddinggemma-300m-qat
   └──────────────────────┘
```

## Data Model

A `MemorySearchResult` contains:

```typescript
type MemorySearchResult = {
  path: string;          // e.g. "memory/2026-01-12.md"
  startLine: number;     // 1-indexed
  endLine: number;       // 1-indexed
  score: number;         // Relevance [0, 1]
  snippet: string;       // Max 700 chars
  source: "memory" | "sessions";
  citation?: string;
};
```

**Memory file organization:**
- `MEMORY.md` / `memory.md` — Root memory file (evergreen, no temporal decay)
- `memory/*.md` — Topic files or dated files (`YYYY-MM-DD.md`)
- Session transcripts — `.jsonl` files from agent sessions

## Database Schema (SQLite)

Six storage structures:

1. **`meta`** — Key-value metadata (model, provider, chunk config)
2. **`files`** — Tracked files with path, SHA256 hash, mtime, size, source
3. **`chunks`** — Segmented text with embeddings (id, path, start/end lines, hash, model, text, embedding vector)
4. **`embedding_cache`** — LRU cache keyed by (provider, model, key hash, content hash) — max 10,000 entries
5. **`chunks_fts`** — FTS5 virtual table for BM25 keyword search
6. **`chunks_vec`** — sqlite-vec virtual table for cosine distance vector search (optional)

## Backends

| Backend | Implementation | Description |
|---------|---------------|-------------|
| **Builtin** | `MemoryIndexManager` (~900 lines) | Local SQLite + optional sqlite-vec. Hybrid search, batch embeddings, temporal decay, MMR. |
| **QMD** | `QmdMemoryManager` (~700 lines) | External Query Memory Daemon process. Falls back to builtin if QMD fails. |

Selection via `getMemorySearchManager()` factory in `search-manager.ts`.

## Embedding Providers

| Provider | Default Model | Batch Support | Notes |
|----------|--------------|---------------|-------|
| OpenAI | `text-embedding-3-small` | Yes (async file API) | Max 8192 tokens |
| Google Gemini | `gemini-embedding-001` | Yes | Configurable base URL |
| Voyage AI | — | Yes | Remote API |
| Mistral | — | Yes | Custom headers + SSRF policy |
| Ollama | configurable | No (single queries) | Local/self-hosted |
| Local GGUF | `embeddinggemma-300m-qat` | No | Via node-llama-cpp, lazy-loaded |

**Auto-selection:** Tries remote providers (OpenAI → Gemini → Voyage → Mistral), then falls back to local. FTS-only mode if no provider available.

## Search Pipeline

### 1. Vector Search (`manager-search.ts`)
- If sqlite-vec available: native `vec_distance_cosine()` for fast search
- Fallback: load all chunks, compute cosine similarity in JavaScript
- Returns top-k results sorted by similarity score

### 2. Keyword Search (`manager-search.ts`)
- Tokenizes query (Unicode-aware: `\p{L}\p{N}_`)
- Builds FTS5 query: `"token1" AND "token2" AND ...`
- BM25 ranking, score = `1 / (1 + rank)`

### 3. Hybrid Merge (`hybrid.ts`)
- Combined score = `0.7 * vectorScore + 0.3 * textScore`
- **Temporal decay** (for dated files): `score * exp(-lambda * ageInDays)` with configurable half-life (default 30 days)
- **MMR re-ranking** (optional): `MMRScore = lambda * relevance - (1-lambda) * maxSimilarityToSelected` using Jaccard similarity on tokenized snippets

### 4. Query Expansion (`query-expansion.ts`)
- For FTS-only mode (no embedding provider)
- Strips 100+ English stop words
- CJK text detection support
- Searches each keyword separately, merges results

## Chunking Strategy

```typescript
function chunkMarkdown(content: string, chunking: {
  tokens: number;    // Default 256
  overlap: number;   // Default 32
}): MemoryChunk[]
```

- Splits content into lines, accumulates until `chars > tokens * 4`
- Overlap carries end of previous chunk into next
- Maintains 1-indexed line numbers for citation

## Sync & Indexing

**Triggers:**
1. **File system watcher** (chokidar) — debounced 100ms, ignores `.git`, `node_modules`, etc.
2. **Session transcript listener** — debounced 5 seconds on new messages
3. **Interval sync** — every 5 minutes (configurable)
4. **Manual sync** — `sync({ reason, force, progress })` API
5. **Session warm-up** — optional pre-sync before first message

**Batch embedding:** Groups chunks until ~8000 tokens, submits 4 concurrent batch requests. Provider-specific timeout (60s remote, 5min local).

**Resilience:** Provider fallback chain, FTS fallback without embeddings, read-only filesystem recovery with exponential backoff, batch failure tracking (disables after 2 consecutive failures).

---

---

# Deep Dive: `src/cli/`

## Overview

The CLI module (~450+ files) implements the `openclaw` command-line interface using **Commander.js** with lazy-loaded subcommands, dependency injection, profile isolation, and rich terminal output.

## Architecture

```
openclaw.mjs
  → src/entry.ts (respawn, profile, env)
    → src/cli/run-main.ts (argv normalization, .env, error handlers)
      → src/cli/program/build-program.ts
          ├── createProgramContext() — version, channel options
          ├── configureProgramHelp() — help system, theming
          ├── registerPreActionHooks() — banner, plugins, config guard
          └── registerProgramCommands() — all subcommands
```

## Key Design Patterns

### Lazy Subcommand Loading

Commands are registered as placeholders. On invocation, the real implementation is dynamically imported, and the program is reparsed. This minimizes startup time.

```typescript
// register.subclis.ts
const entries: SubCliEntry[] = [
  {
    name: "acp",
    description: "Agent Control Protocol tools",
    register: async (program) => {
      const mod = await import("../acp-cli.js");
      mod.registerAcpCli(program);
    },
  },
  // ... gateway, daemon, logs, system, models, etc.
];
```

Lazy loading is disabled when `--help` is passed (all commands need to be visible).

### Dependency Injection (`deps.ts`)

```typescript
type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  sendMessageSlack: typeof sendMessageSlack;
  sendMessageSignal: typeof sendMessageSignal;
  sendMessageIMessage: typeof sendMessageIMessage;
};
```

Each sender uses lazy runtime loading — imports only when actually called.

### Program Context (`context.ts`)

Stored on the Commander `Command` instance via a Symbol for type-safe access:

```typescript
type ProgramContext = {
  programVersion: string;
  channelOptions: string[];       // Dynamic list of available channels
  messageChannelOptions: string;  // "whatsapp|telegram|discord|..."
  agentChannelOptions: string;    // "last|whatsapp|telegram|..."
};
```

### Pre-Action Hooks (`preaction.ts`)

Execute before every command:
1. Set process title
2. Emit CLI banner with version + tagline
3. Load plugin registry (if command requires channels)
4. Validate config (blocks invalid state, allows diagnostic commands like `doctor`)
5. Set log levels

### Fast-Path Routing (`routes.ts`)

Common commands (`health`, `status`, `sessions`, `agents list`, `memory status`) bypass Commander entirely for faster execution:

```typescript
type RouteSpec = {
  match: (path: string[]) => boolean;
  loadPlugins?: boolean;
  run: (argv: string[]) => Promise<boolean>;
};
```

### Profile Isolation (`profile.ts`)

`--dev` and `--profile <name>` isolate state under `~/.openclaw-<profile>`:

```typescript
function applyCliProfileEnv(params: { profile: string }): void {
  // Sets OPENCLAW_PROFILE, OPENCLAW_STATE_DIR, OPENCLAW_CONFIG_PATH
}
```

## Subcommand Tree

| CLI Group | File | Subcommands |
|-----------|------|-------------|
| `gateway` | `gateway-cli/register.ts` | `run`, `call`, `discover` |
| `daemon` | `daemon-cli/register.ts` | `install`, `start`, `stop`, `restart`, `status`, `uninstall` |
| `node` | `node-cli/register.ts` | `run`, `install`, `status`, `stop`, `restart`, `uninstall` |
| `browser` | `browser-cli.ts` | manage, extensions, inspect, actions (input/observe), debug, state |
| `nodes` | `nodes-cli/register.ts` | invoke, camera, screen, pairing, rpc |
| `cron` | `cron-cli/` | Cron job scheduling |
| `update` | `update-cli/` | Update mechanism |
| `acp` | `acp-cli.ts` | Agent Control Protocol tools |
| `logs` | — | Log viewing |
| `system` | — | System info |
| `models` | — | Model management |
| `approvals` | — | Approval management |

## Progress Reporting (`progress.ts`)

Multi-mode progress system:

| Mode | When Used |
|------|-----------|
| OSC progress | Modern terminals (escape sequences) |
| Spinner | Interactive TTY (via `@clack/prompts`) |
| Line-based | Non-TTY environments |
| Log output | Piped output |
| Noop | Disabled |

```typescript
type ProgressReporter = {
  setLabel: (label: string) => void;
  setPercent: (percent: number) => void;
  tick: (delta?: number) => void;
  done: () => void;
};
```

## Channel Options Resolution (`channel-options.ts`)

Builds a dynamic list of available channels by merging:
1. Precomputed metadata from `cli-startup-metadata.json`
2. Channel plugin catalog entries
3. Loaded plugin IDs
4. Deduplicated with built-in channel order

## Gateway RPC (`gateway-rpc.ts`)

Wraps gateway calls with progress reporting and connection management:

```typescript
async function callGatewayFromCli(
  method: string,
  opts: GatewayRpcOpts,
  params?: unknown,
): Promise<unknown>
```

---

---

# Deep Dive: `src/commands/`

## Overview

The commands module (~340 files, ~52,282 lines) implements all CLI command logic: `agent`, `agents`, `channels`, `models`, `onboard`, `status`, and supporting utilities.

## Command Summary

| Command | File | Purpose |
|---------|------|---------|
| `agent` | `agent.ts` | Execute agent with a user prompt |
| `agents add` | `agents.commands.add.ts` | Create new agent |
| `agents list` | `agents.commands.list.ts` | Show all agents |
| `agents delete` | `agents.commands.delete.ts` | Remove an agent |
| `agents bind` | `agents.commands.bind.ts` | Configure message routing rules |
| `agents identity` | `agents.commands.identity.ts` | Set agent persona/avatar |
| `channels add` | `channels/add.ts` | Configure a messaging channel |
| `channels list` | `channels/list.ts` | Show configured channels |
| `channels remove` | `channels/remove.ts` | Disable or delete a channel |
| `channels status` | `channels/status.ts` | Check channel connections |
| `models list` | `models/list.ts` | Show available models |
| `models auth` | `models/auth.ts` | Set up model authentication |
| `models scan` | `models/scan.ts` | Auto-detect local models |
| `onboard` | `onboard.ts` | Full interactive setup wizard |
| `status` | `status.ts` | System status |
| `status --all` | `status-all.ts` | Comprehensive status report |

## `agent` Command (Main Entry Point)

**File:** `agent.ts` (~32KB) — the core command that invokes the AI agent.

**Key options:**

```
--message <text>          User prompt (required)
--agent-id <id>           Agent override
--to <phone>              Phone number (E.164)
--session-id <id>         Session ID
--thinking <level>        on|off|extended|xhigh
--verbose <level>         on|full|off
--json                    JSON output
--timeout <seconds>       Timeout
--deliver                 Send result to channel
--channel <name>          Delivery channel
--thread-id <id>          Thread/topic ID
--extra-system-prompt     Additional system instructions
--images <paths>          Multimodal image attachments
```

**Execution flow:**

1. Validate message and session
2. Load config, resolve agent ID
3. Resolve or create session (per-sender / per-thread / global scope)
4. Build agent workspace, ensure bootstrap files
5. Resolve model/provider with fallback support
6. Execute via one of two paths:
   - **CLI Provider Path** (`runCliAgent`) — for `claude-cli` and CLI-based providers
   - **Embedded PI Agent Path** (`runEmbeddedPiAgent`) — for cloud providers (Anthropic, OpenAI, etc.)
7. Handle session expiration with automatic retry
8. Optionally deliver result to messaging channel
9. Persist session state

## `agents` Management Commands

### `agents add`
Interactive wizard: creates workspace, prompts for model, sets up auth, optionally binds channels.

### `agents bind`
Configures routing rules with binding specs:
```
telegram                    → All Telegram messages
discord #general            → Discord #general channel
slack @user-id              → Slack user DMs
whatsapp +1234567890        → WhatsApp contact
channel[:account] [peer=kind:id] [guild=id] [roles=role1,role2]
```

### `agents identity`
Sets persona: display name, emoji, theme, avatar URL, or loads from `IDENTITY.md` file.

## `channels` Commands

### `channels add` — Interactive Channel Setup
1. Lists available channel plugins
2. Prompts for channel selection
3. For each: account setup, optional npm plugin install, credential configuration
4. Sets channel-specific options (DM policy, sync limit, allowlists)

### `channels status`
Probes channel connections and reports: connection state, last activity, configuration issues.

## `models` Commands

### `models auth` — Authentication Setup

**50+ auth providers supported:**

| Category | Providers |
|----------|-----------|
| Anthropic | `token`, `setup-token`, `oauth` |
| OpenAI | `openai-codex`, `openai-api-key` |
| Open Source | Ollama, vLLM, LM Studio (local) |
| Alternative | OpenRouter, Mistral, HuggingFace, Google Gemini, Moonshot, Minimax |
| Custom | Custom base URL + API key |

Auth flow: provider selection → OAuth/token/API-key → credential validation → storage in auth profile.

### `models scan`
Auto-detects locally running models (Ollama, vLLM, etc.).

## `onboard` — Full Setup Wizard

**Interactive onboarding flow:**

```
1. Welcome / Mode Selection (local vs remote)
2. Flow Selection (quickstart vs advanced)
3. Workspace Setup (create/select directory)
4. Auth Selection (choose AI provider + auth method)
5. Channel Setup (configure messaging channels)
6. Skills Setup (install/enable workspace skills)
7. Health Check (verify configuration)
8. Control UI (optional dashboard setup)
```

**Plugin installation:** Supports `npm` (download from registry), `local` (development path), or `skip`.

## `status --all` — Comprehensive Report

Checks:
1. Configuration validity
2. Tailscale status
3. Update availability
4. Node service status
5. Gateway status and connection
6. Model/provider availability
7. Channel connection status
8. Daemon service status
9. Workspace skills status
10. Session history
11. Control UI availability

## Session Management (`agent/session.ts`)

**Session resolution:**
1. Resolve explicit key from `--session-key`, agent config, or `--to` phone number
2. Load session store from disk
3. Evaluate freshness (reset policies)
4. Return session entry with ID, key, metadata, model overrides, thinking settings

**Scopes:** `per-sender` (default) | `per-thread` | `global`

## Message Delivery (`agent/delivery.ts`)

1. Resolve delivery target (override `--reply-to` or original `--to`)
2. Resolve delivery channel (override `--reply-channel` or original channel)
3. Normalize outbound payloads (text + media)
4. Call channel plugins to send
5. Log delivery results

---

---

# Deep Dive: `src/browser/`

## Overview

The browser module (~155 TypeScript files) provides **full browser automation** for the AI agent, built on **Playwright** (not Puppeteer). It exposes 50+ granular tools via an HTTP REST API and direct TypeScript bindings.

## Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│                AI Agent                          │
└──────────────────┬──────────────────────────────┘
                   │ HTTP POST /act, /snapshot, etc.
┌──────────────────▼──────────────────────────────┐
│          Express Server (127.0.0.1:port)         │
│  Auth: Bearer token / password                   │
├──────────────────────────────────────────────────┤
│  Routes:                                         │
│  /act     → click, type, press, hover, drag,     │
│             select, fill, resize, wait, evaluate │
│  /snapshot → aria, ai, role snapshots            │
│  /navigate → URL navigation (SSRF-guarded)       │
│  /download → file download tracking              │
│  /storage  → cookies, localStorage, sessionStorage│
│  /debug    → console, errors, network, traces    │
│  /tabs     → list, open, close, focus            │
│  /basic    → status, start, stop, reset          │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│     Playwright Tools Layer (pw-tools-core.ts)    │
│  interactions, snapshot, state, responses,        │
│  downloads, storage, activity, trace             │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│     Playwright Session (pw-session.ts)           │
│  Page management, role refs caching,             │
│  event observers (console, errors, network)      │
└──────────────────┬──────────────────────────────┘
                   │ CDP (Chrome DevTools Protocol)
┌──────────────────▼──────────────────────────────┐
│         Chrome / Chromium Process                 │
│  Profile management, subprocess lifecycle        │
│  Platform-specific binary detection              │
└──────────────────────────────────────────────────┘
```

## Browser Engine

**Playwright-core** via `import { chromium } from "playwright-core"`:
- Connects to Chrome/Chromium via CDP over WebSocket
- Supports both local Chrome processes and remote CDP connections
- Persistent browser connection with pooling and reconnection logic

## Core Files

| File | Lines | Purpose |
|------|-------|---------|
| `pw-session.ts` | 824 | Primary: Playwright page session management, event observers |
| `chrome.ts` | 400+ | Chrome process spawning, profile management, subprocess lifecycle |
| `pw-tools-core.interactions.ts` | — | Click, hover, type, drag, scroll, fill, select |
| `pw-tools-core.snapshot.ts` | — | Aria, AI, and Role-based page snapshots |
| `pw-tools-core.state.ts` | — | Offline mode, headers, credentials, geolocation, emulation |
| `pw-tools-core.responses.ts` | — | Network response interception and body extraction |
| `pw-tools-core.downloads.ts` | — | Download handling with temp path management |
| `pw-tools-core.storage.ts` | — | Cookies, localStorage, sessionStorage |
| `pw-tools-core.activity.ts` | — | Console messages, page errors, network requests |
| `pw-tools-core.trace.ts` | — | Playwright trace recording |
| `screenshot.ts` | — | Screenshot normalization, resizing, JPEG compression |
| `pw-role-snapshot.ts` | — | Role-based element refs (e1, e2, e3...) |
| `config.ts` | — | Browser config resolution |
| `server.ts` | — | Express HTTP server |
| `navigation-guard.ts` | — | SSRF/navigation policy validation |

## Browser Tools (50+ exposed to AI agent)

### Page Navigation & Lifecycle
- `navigateViaPlaywright()` — Navigate to URLs with SSRF checks
- `createPageViaPlaywright()` — Open new tabs
- `closePageViaPlaywright()` — Close tabs
- `focusPageByTargetIdViaPlaywright()` — Bring tab to front

### Element Interaction
- `clickViaPlaywright()` — Click with button/modifiers/timeout
- `typeViaPlaywright()` — Type text with submit/slowly options
- `pressKeyViaPlaywright()` — Press keyboard keys
- `hoverViaPlaywright()` — Hover over elements
- `fillFormViaPlaywright()` — Multi-field form filling
- `selectOptionViaPlaywright()` — Select dropdown options
- `dragViaPlaywright()` — Drag and drop
- `scrollIntoViewViaPlaywright()` — Scroll element into view

### Page Observation & Snapshots

**Three snapshot modes:**

| Mode | Method | Description |
|------|--------|-------------|
| **AI** | `snapshotAiViaPlaywright()` | Optimized accessibility tree for LLMs (max 80K chars). Uses Playwright's `_snapshotForAI()`. |
| **ARIA** | `snapshotAriaViaPlaywright()` | Full DOM accessibility tree via CDP `Accessibility.getFullAXTree`. |
| **Role** | `snapshotRoleViaPlaywright()` | Builds role-based refs (e1, e2...) with role, name, nth for disambiguation. 50-entry LRU cache. |

**Screenshots:**
- `takeScreenshotViaPlaywright()` — Full/element screenshots
- `screenshotWithLabelsViaPlaywright()` — Screenshots with labeled overlays
- Auto-compression: resizes >2000px, converts to JPEG if >5MB, quality steps 100→60

**Observation:**
- `getConsoleMessagesViaPlaywright()` — Last 500 console messages
- `getPageErrorsViaPlaywright()` — Last 200 JS errors with stack traces
- `getNetworkRequestsViaPlaywright()` — Last 500 requests with status

### Network Control
- `setExtraHTTPHeadersViaPlaywright()` — Custom request headers
- `setHttpCredentialsViaPlaywright()` — HTTP Basic auth
- `setOfflineViaPlaywright()` — Offline mode simulation
- `responseBodyViaPlaywright()` — Get response body by URL pattern (max 200KB, 20s timeout)

### Device Emulation
- `setDeviceViaPlaywright()` — Emulate phone/tablet
- `resizeViewportViaPlaywright()` — Change viewport size
- `setGeolocationViaPlaywright()` — Mock GPS location
- `emulateMediaViaPlaywright()` — Dark/light mode, color scheme
- `setLocaleViaPlaywright()` — Language/locale
- `setTimezoneViaPlaywright()` — Timezone

### Storage Access
- `cookiesGet/Set/ClearViaPlaywright()` — Cookie management
- `storageGet/Set/ClearViaPlaywright()` — localStorage/sessionStorage

### File Operations
- `downloadViaPlaywright()` — Trigger file download
- `waitForDownloadViaPlaywright()` — Wait for completion
- `armFileUploadViaPlaywright()` / `setInputFilesViaPlaywright()` — File inputs

### Other
- `armDialogViaPlaywright()` — Accept/reject alert dialogs
- `evaluateViaPlaywright()` — Execute JavaScript in page context
- `waitForViaPlaywright()` — Wait for text, URL, loadState, custom functions
- `traceStart/StopViaPlaywright()` — Playwright trace recording
- `pdfViaPlaywright()` — Generate PDF of page
- `highlightViaPlaywright()` — Visual highlight element

## Data Flow (Example: Click Action)

```
AI Agent
  → HTTP POST /act { ref: "e5", kind: "click" }
    → routes/agent.act.ts (deserialize, validate)
      → pw-tools-core.interactions.clickViaPlaywright(opts)
        → pw-session.ts getPageForTargetId() → Playwright Page
          → page.locator().click() → CDP command to Chrome
            → HTTP 200 { ok: true, targetId, url }
              → AI receives result
```

## Chrome Process Management

- **Detection:** Platform-specific binary finding (Windows Registry, Mac standard paths, Linux `which`)
- **Profiles:** Named browser profiles with persistent user data dirs
- **CDP Connection:** WebSocket `ws://127.0.0.1:cdpPort`, with HTTP fallback for tab enumeration
- **Connection pooling:** Cached, reconnects on disconnect (3 retries, 250ms backoff)
- **Drivers:** `"openclaw"` (direct subprocess) or `"extension"` (Chrome extension relay for remote control)

## Security

- **Auth:** Bearer token or password header on the control server
- **SSRF:** Navigation guard validates URLs before and after navigation (blocks private IPs unless configured)
- **CDP proxy bypass:** Loopback CDP connections skip HTTP proxy
- **CSRF:** Token management for browser control endpoints
- **Evaluation abort:** Stuck JS evaluations killed via `Runtime.terminateExecution` before disconnect

---

---

## Summary

OpenClaw is a **large, production-grade monorepo** (~566K lines of TypeScript in `src/` alone) that acts as a personal AI gateway. Its key innovation is the **unified multi-channel inbox** — one AI assistant that speaks across 25+ messaging platforms with a plugin architecture for extending to new ones. It supports voice (wake words + talk mode), a visual canvas, companion native apps, and 52 built-in skills for tasks like GitHub, Notion, Spotify, coding, and more.

The four modules analyzed in detail reveal the depth of the system:
- **Memory** — Sophisticated hybrid search with vector embeddings, temporal decay, and MMR diversity re-ranking
- **CLI** — Lazy-loaded Commander.js architecture with profile isolation, plugin discovery, and multi-mode progress
- **Commands** — 50+ auth providers, multi-agent routing, interactive onboarding wizard, session management
- **Browser** — Production Playwright-based automation with 50+ tools, three snapshot modes, SSRF protection, and Chrome lifecycle management
