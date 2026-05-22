# Atomic Hermes — VPS Hosting & Custom UI Build Plan

Comprehensive reference for hosting `atomic-hermes` on your own VPS and building a
modern web/mobile UI that talks to it from anywhere. Inventories every backend
surface, every feature, every event type, and every UI component you'll need.

Built from a full audit of the repo (May 2026) — see the "Source map" section
at the end for pointers into the code if you want to verify any of this.

---

## 1. TL;DR — what you're working with

Atomic Hermes is a desktop wrapper around the **Hermes Agent** core
(Nous Research, MIT). The desktop app you see on macOS is just one client; the
agent itself is a Python process that already exposes **three distinct HTTP
surfaces** you can hit from a custom UI:

| Surface | Where | Purpose | Auth | Best for |
|---|---|---|---|---|
| **Gateway API** (`gateway/platforms/api_server.py`, port `8642`) | aiohttp | OpenAI-compatible chat + Responses API + jobs + Runs SSE | `Authorization: Bearer $API_SERVER_KEY` | The chat itself + agent runs |
| **Admin web server** (`hermes_cli/web_server.py`) | FastAPI | Sessions, config, env, OAuth, cron, skills, themes, analytics, logs | Same key, optional | Everything else (settings, history, scheduler, etc.) |
| **Desktop bridge** (`desktop/src/python-server/server.py`) | FastAPI WS | WebSocket chat stream w/ rich event types | None (localhost only by default) | Reference for the streaming protocol |

For a VPS the play is: **run the Gateway API + Admin server on the VPS, ditch
the Electron shell, build your own UI** that hits both. You can ignore the
desktop bridge entirely (or reuse its WebSocket event shape if you prefer that
over SSE).

The agent ships **40+ built-in tools**, **6 terminal backends** (local, Docker,
SSH, Daytona, Singularity, Modal), **persistent SQLite session storage with
FTS5 search**, an **agent-curated memory system**, a **self-improving skills
library**, **cron scheduling**, **MCP-native tool ingestion**, **subagent
delegation**, **prompt caching**, **20+ model providers**, and a
**16+ messenger gateway** (Telegram, Discord, Slack, WhatsApp, Signal,
iMessage, Email, Matrix, Teams, etc.). Computer-use / OCR is a separate
npm-published add-on (`@atomicbotai/computer-use-mcp`) and is **macOS/Windows
only** — you cannot run it on a Linux VPS, so plan to drop or remote-attach it.

---

## 2. Architecture for VPS hosting

```
                       ┌─────────────────────────────────────────────┐
                       │             Your VPS (Linux)                │
                       │                                             │
  Your phone/laptop    │   nginx / Caddy (TLS, auth, rate-limit)     │
       │               │     │                                       │
       │  HTTPS/WSS    │     ├──→ :8642  Gateway API (aiohttp)        │
       └───────────────┼──→  │           ├ /v1/chat/completions       │
       (your UI app)   │     │           ├ /v1/responses              │
                       │     │           ├ /v1/runs (SSE)             │
                       │     │           ├ /v1/models                 │
                       │     │           ├ /api/jobs (cron)           │
                       │     │           ├ /health/detailed           │
                       │     │           └ /warmup                    │
                       │     │                                        │
                       │     └──→ :8645  Admin Web Server (FastAPI)   │
                       │                  ├ /api/sessions ...         │
                       │                  ├ /api/cron/jobs ...        │
                       │                  ├ /api/skills ...           │
                       │                  ├ /api/config ...           │
                       │                  ├ /api/env ...              │
                       │                  ├ /api/providers/oauth ...  │
                       │                  ├ /api/analytics/usage ...  │
                       │                  ├ /api/logs                 │
                       │                  └ /api/dashboard/themes ... │
                       │                                              │
                       │   $HERMES_HOME/  ($HOME/.hermes by default)  │
                       │     ├ config.yaml                            │
                       │     ├ .env             (provider keys)       │
                       │     ├ sessions.db      (SQLite + FTS5)       │
                       │     ├ skills/          (markdown skills)     │
                       │     ├ memory/          (agent memory)        │
                       │     ├ logs/            (agent.log, etc.)     │
                       │     ├ auth-profiles.json (OAuth tokens)      │
                       │     └ <profile>/       (multi-tenant)        │
                       │                                              │
                       │   Hermes Agent core (Python 3.11 venv)       │
                       │     spawns:                                  │
                       │       • terminal subprocesses                │
                       │       • subagents                            │
                       │       • cron scheduler                       │
                       │       • messaging gateway adapters           │
                       │       • MCP server child processes           │
                       │                                              │
                       └──────────────────────────────────────────────┘
```

**Deployment shape:**

1. `git clone` the repo to `/opt/atomic-hermes` (or wherever).
2. Install Python 3.11 + `uv`, then `uv sync --all-extras` to create `.venv/`.
3. Set required env vars in `$HERMES_HOME/.env` (provider API keys).
4. Set `API_SERVER_ENABLED=true`, `API_SERVER_HOST=127.0.0.1`,
   `API_SERVER_PORT=8642`, and a strong `API_SERVER_KEY=<random-secret>`.
   The Gateway **refuses to bind to a non-loopback address without a key**
   (good — keeps you safe).
5. Run the gateway as a systemd unit (entrypoint: `python -m gateway.run`).
6. Run the admin web server as a second systemd unit
   (`python -m hermes_cli.web_server` or equivalent — see source map).
7. Front both with nginx/Caddy bound to 0.0.0.0:443 with a real TLS cert.
   Add `Authorization: Bearer …` injection in nginx if you don't want clients
   to hold the master key (i.e. nginx auths the user, then forwards with the
   shared bearer).
8. Your UI app talks **only to nginx/Caddy over HTTPS+WSS**, never directly to
   the Python ports.

**Profiles:** Hermes supports multi-tenant isolation via the `HERMES_PROFILE`
env var (or `--profile`). Each profile gets its own `~/.hermes/<name>/` with
isolated keys, sessions, skills, memory, cron jobs, and gateway config. If you
want a multi-user product later, run one process per profile, or use the
gateway's profile registry (`gateway/platforms/api_profile_registry.py`) which
already routes requests to per-profile worker processes via the
`X-Hermes-Profile` header.

---

## 3. Backend API inventory

### 3.1 Gateway API (`:8642`, aiohttp) — **the chat surface**

OpenAI-compatible — any OpenAI SDK / library "just works" if you point it at
`https://your-vps/v1` and use your bearer key.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/chat/completions` | OpenAI chat (stateless; opt-in continuity via `X-Hermes-Session-Id` header) — supports `stream: true` SSE |
| `POST` | `/v1/responses` | OpenAI Responses API (stateful via `previous_response_id`) |
| `GET`  | `/v1/responses/{id}` | Retrieve stored response |
| `DELETE` | `/v1/responses/{id}` | Delete stored response |
| `GET`  | `/v1/models` | List available models (returns `hermes-agent`) |
| `POST` | `/v1/runs` | Start a run, returns `run_id` (202 Accepted) — for long-running agentic work |
| `GET`  | `/v1/runs/{id}/events` | **SSE stream of structured lifecycle events** for that run |
| `GET`  | `/api/jobs` | List cron jobs |
| `POST` | `/api/jobs` | Create cron job |
| `GET`/`PATCH`/`DELETE` | `/api/jobs/{id}` | Read/update/delete |
| `POST` | `/api/jobs/{id}/pause` | Pause |
| `POST` | `/api/jobs/{id}/resume` | Resume |
| `POST` | `/api/jobs/{id}/run` | Trigger now |
| `GET`  | `/health` | Liveness |
| `GET`  | `/health/detailed` | Rich status (model, provider, queue depth, etc.) |
| `POST` | `/warmup`, `/api/warmup` | Local model warmup (desktop convenience — useful if you self-host an Ollama/llama.cpp) |

There is a documented **extension layer** (`gateway/platforms/api_extensions.py`)
that adds 7+ more endpoints when present. From `docs/api-extensions.md`:

| Method | Path | Purpose |
|---|---|---|
| `GET`  | `/api/capabilities` | One-shot probe of what the server supports |
| `GET`/`PATCH` | `/api/config` | Read/write config.yaml + .env values |
| `GET`  | `/api/providers` | All providers + auth status + model lists |
| `POST` | `/api/model-switch` | Hot-swap model/provider at runtime (no restart) |
| `GET`  | `/api/skills` | Installed skills |
| `GET`  | `/api/memory` | Memory files |
| `GET`  | `/api/mcp/servers` | Configured MCP servers |

### 3.2 Admin Web Server (`hermes_cli/web_server.py`, FastAPI) — **the dashboard surface**

Where you wire the bulk of "settings/history/admin/observability" screens. Full
list (44 endpoints), grouped:

**Status & health**
- `GET /api/status`
- `GET /api/model/info`
- `POST /api/gateway/restart`
- `POST /api/hermes/update`
- `GET /api/actions/{name}/status`

**Sessions (conversation history)**
- `GET /api/sessions` — paginated list
- `GET /api/sessions/search` — FTS5 full-text search across all conversations
- `GET /api/sessions/{id}` — metadata
- `GET /api/sessions/{id}/messages` — full transcript
- `DELETE /api/sessions/{id}`

**Config & env**
- `GET /api/config`, `PUT /api/config`
- `GET /api/config/raw`, `PUT /api/config/raw` — direct YAML edit
- `GET /api/config/defaults`, `GET /api/config/schema` — for building a typed settings form
- `GET /api/env`, `PUT /api/env`, `DELETE /api/env`
- `POST /api/env/reveal` — unmask a stored key (with re-auth)

**Provider OAuth flows** (for Anthropic Console, GitHub Copilot, Google, etc.)
- `GET /api/providers/oauth` — list providers + auth state
- `DELETE /api/providers/oauth/{provider_id}` — sign out
- `POST /api/providers/oauth/{provider_id}/start` — start device/PKCE flow
- `POST /api/providers/oauth/{provider_id}/submit` — submit user code
- `GET /api/providers/oauth/{provider_id}/poll/{session_id}` — poll until done
- `DELETE /api/providers/oauth/sessions/{session_id}` — cancel

**Logs**
- `GET /api/logs` — tail with filters (`level`, `session`, `follow`, `limit`)

**Cron jobs**
- `GET /api/cron/jobs`, `POST /api/cron/jobs`
- `GET`/`PUT`/`DELETE` `/api/cron/jobs/{id}`
- `POST /api/cron/jobs/{id}/pause`, `/resume`, `/trigger`

**Skills**
- `GET /api/skills` — installed + available from skills hub
- `PUT /api/skills/toggle` — enable/disable

**Tools**
- `GET /api/tools/toolsets` — toolset catalog + per-tool enable state

**Analytics**
- `GET /api/analytics/usage` — token counts, costs, model breakdown, time series

**Dashboard customisation**
- `GET /api/dashboard/themes`, `PUT /api/dashboard/theme`
- `GET /api/dashboard/plugins`, `GET /api/dashboard/plugins/rescan`
- `GET /dashboard-plugins/{plugin_name}/{file_path}` — static-serve plugin UIs

### 3.3 Desktop bridge WebSocket (reference only)

`ws://host/chat` accepts `{ message, session_id, model }` JSON frames and pushes
event JSON frames. Use this **only if you want WS-native streaming** instead of
SSE — both work, and the event shapes are very similar.

---

## 4. Streaming event types (so you know what to render)

From `desktop/src/python-server/bridge.py` (WS) and
`desktop/renderer/src/services/sse-chat.ts` (SSE), the agent emits these
event types during a single user turn. Your UI must handle each:

| Event | Payload | When | What to render |
|---|---|---|---|
| `session_id` | `{ session_id }` | Once at start of new session | Update URL / store thread id |
| `stream_delta` (SSE: regular `data:` chunks) | `{ text }` | Continuously while LLM streams | Append to current assistant bubble |
| `reasoning_delta` (thinking) | `{ text }` | While model reasons (Opus/Sonnet/Gemini Thinking) | Render in a collapsible "Thinking" panel |
| `tool_start` | `{ id, name, args, preview }` | When agent invokes a tool | Add an inline tool-call card with status "running" |
| `tool_progress` | `{ emoji, label, tool }` | Live progress strings | Update the tool card's status line |
| `tool_complete` | `{ id, name, result }` | When tool returns | Mark card "done", make result expandable |
| `step` | `{ api_call_count }` | After each LLM turn within the loop | Iteration counter |
| `status` | `{ text }` | Generic status (`"Thinking..."`, `"Compressing context..."`) | Subtle status bar |
| `exec_approval_requested` | `{ command, description, session_id }` | When agent wants to run a dangerous shell command | **Block the UI with an approval modal** |
| `final_response` | `{ text, session_id }` | End of turn | Finalize the assistant bubble |
| `error` | `{ text }` | Anything blew up | Toast + don't lose user input |

Approval flow: the gateway also exposes `POST /api/approval/{request_id}` to
respond yes/no/always-allow (see `desktop/renderer/src/services/approval-api.ts`).

---

## 5. Tool catalog — what the agent can actually do

These are the `_HERMES_CORE_TOOLS` from `toolsets.py`. Your UI doesn't have to
expose each as a button — the agent calls them — but you'll render them in
tool-call cards and you can offer **toolset-level on/off toggles** in settings
(via `GET /api/tools/toolsets`).

**Web & research:** `web_search`, `web_extract`

**Filesystem (with safety guards):** `read_file`, `write_file`, `patch`
(fuzzy-matched diffs), `search_files` (content + path)

**Shell / processes:** `terminal` (run command), `process` (manage background
processes — start, stop, tail, kill). Supports 6 backends: local, Docker, SSH,
Daytona, Singularity, Modal.

**Vision & media:** `vision_analyze` (image understanding), `image_generate`
(via 5+ image providers), `text_to_speech` (Edge TTS free, ElevenLabs, OpenAI,
xAI)

**Skills:** `skills_list`, `skill_view`, `skill_manage` — agent reads/edits
its own skills library

**Browser automation (camoufox + CDP):** `browser_navigate`, `browser_snapshot`,
`browser_click`, `browser_type`, `browser_scroll`, `browser_back`,
`browser_press`, `browser_get_images`, `browser_vision`, `browser_console`,
`browser_cdp`

**Productivity:** `todo` (multi-step planner), `memory` (persistent notes +
user profile via Honcho), `session_search` (FTS5 across all past chats)

**Agent ops:** `clarify` (ask user multi-choice), `execute_code` (programmatic
tool calls in one inference), `delegate_task` (spawn subagents),
`cronjob` (the agent can schedule itself), `send_message` (cross-platform)

**Smart home:** `ha_list_entities`, `ha_get_state`, `ha_list_services`,
`ha_call_service`

**Feishu / Lark integration:** `feishu_doc_read`,
`feishu_drive_list_comments`, `feishu_drive_list_comment_replies`,
`feishu_drive_reply_comment`, `feishu_drive_add_comment`

**RL training (Tinker-Atropos):** 10× `rl_*` tools

**MCP plug-in tools:** anything from any MCP server the user wires up appears
here at runtime — `GET /api/capabilities` and the model's tool list at the
start of a run tell you what's available right now.

**Toolset packages** (presets you toggle as a unit): `web`, `search`, `vision`,
`image_gen`, `terminal`, `moa`, `skills`, `browser`, `cronjob`, `messaging`,
`rl`, `file`, `tts`, `todo`, `memory`, `session_search`, `clarify`,
`code_execution`, `delegation`, `homeassistant`, `feishu_doc`, `feishu_drive`,
`debugging`, `safe`, `hermes-acp`. Each has a description and tool list — your
"Toolsets" settings page can render these as cards.

---

## 6. Subsystems your UI needs to cover

### 6.1 Sessions

- SQLite at `$HERMES_HOME/sessions.db` (or per-profile).
- Schema: sessions + messages + FTS5 index.
- Each session has: id, title (auto-generated by `agent/title_generator.py`),
  created_at, updated_at, platform (`api`, `cli`, `telegram`, …),
  message_count, token totals, model used.
- Messages are OpenAI-format role/content + tool calls + reasoning content.

### 6.2 Memory

- `$HERMES_HOME/memory/` — markdown files curated by the agent.
- Personal notes vs user profile (Honcho dialectic).
- Agent decides what's worth remembering; you let the user audit/edit.

### 6.3 Skills

- `$HERMES_HOME/skills/` — markdown skills with optional triggers (e.g.
  `/web-search`).
- "Skills Hub" pulls from `agentskills.io`. The list comes back from
  `GET /api/skills`.
- Skills can be installed, toggled on/off, edited, removed, and the agent itself
  **writes new skills** after finishing complex tasks (self-improving loop).

### 6.4 Cron

- Job fields: `id`, `name`, `prompt`, `schedule` (cron expr or `every Nm`),
  `enabled`, `enabled_toolsets`, `last_run`, `next_run`, `delivery` (where to
  send results — chat, Slack, email, …).
- Jobs run with full agent context (memory + skills available).

### 6.5 MCP servers

- Stored under `config.yaml > mcp_servers`.
- Each: `name`, `transport` (`stdio`|`http`), `command`, `args`, `env`,
  `timeout`, `oauth` config (some MCPs need OAuth — `tools/mcp_oauth.py`).
- Once added, the agent auto-discovers their tools at startup.

### 6.6 Profiles

- Multiple isolated agents (personal, work, research). Each is a separate
  `$HERMES_HOME/<profile>/`.
- Switch via `HERMES_PROFILE=work` or the `X-Hermes-Profile` header.
- For a multi-user product: one profile = one user; lifecycle is managed by
  `gateway/platforms/api_profile_runtime.py` (it spins per-profile worker
  processes on demand and reaps idle ones).

### 6.7 Messaging Gateway (out-bound integrations)

Adapters in `gateway/platforms/`. Toggle each on from settings and the same
agent picks up messages from:

`telegram` · `discord` · `slack` · `whatsapp` · `signal` · `imessage` (via
BlueBubbles) · `sms` (Twilio) · `email` · `matrix` · `mattermost` ·
`microsoft-teams` · `feishu` · `dingtalk` · `wecom` · `weixin` · `qqbot` ·
`homeassistant` · `webhook` (generic)

Each has its own config block (tokens, webhooks, channel allowlist, …). Your UI
needs a per-platform config screen + an "enabled" toggle + a status indicator
(running / errored / disconnected).

### 6.8 Providers (in-bound model APIs)

20+: OpenRouter, Anthropic, OpenAI, Google (Gemini), DeepSeek, Kimi/Moonshot,
MiniMax, Nous Portal, xAI, z.ai, Venice, NVIDIA NIM, Alibaba Cloud, Xiaomi
MiMo, Ollama, HuggingFace, Hugging Face Inference, AWS Bedrock,
Google Code Assist, GitHub Copilot, …

Two auth styles:
- **API key** in `.env` (most providers) — `OPENROUTER_API_KEY`,
  `ANTHROPIC_API_KEY`, …
- **OAuth** (`hermes_cli/auth.py` + `agent/google_oauth.py` +
  `hermes_cli/copilot_auth.py` + `hermes_cli/dingtalk_auth.py`) — device-code or
  PKCE flow ending in tokens saved to `auth-profiles.json`.

`hermes_cli/model_switch.py` resolves aliases, picks credentials, normalises
model names against `models.dev` catalog.

---

## 7. UI component inventory — the actual build list

Top-level information architecture (left rail):

```
  Chat (default landing)
  Sessions / History
  Files / Workspace
  Terminal
  Skills
  Memory
  Cron / Jobs
  Tools & Toolsets
  MCP Servers
  Messaging Gateway
  Providers / Models
  Analytics
  Logs
  Settings
  Profiles  (top-right switcher, like Slack workspaces)
```

For each area below: components you need + the API/event surface that feeds
them.

### 7.1 Chat (the headline screen)

**Components:**
- **ConversationView** — virtualised list of message bubbles.
- **MessageBubble** — variants: `user`, `assistant`, `system`, `tool`.
- **AssistantBubble** — supports streaming append; renders markdown +
  syntax-highlighted code (use Shiki or Prism) + math (KaTeX) + tables + image
  references + audio players (for TTS results).
- **ThinkingPanel** — collapsible, monospace, dimmed; streams `reasoning_delta`.
- **ToolCallCard** — collapsible card per tool call: header (icon + tool name +
  status pill running/done/failed), args preview, expandable result. Special
  renderers for: `read_file`/`write_file`/`patch` (diff view), `terminal`
  (terminal-themed output), `web_search` (link list), `vision_analyze` (image
  + bullets), `image_generate` (gallery), `browser_*` (screenshot strip),
  `delegate_task` (nested mini-conversation).
- **ApprovalModal** — pops on `exec_approval_requested`. Buttons: Allow,
  Allow Always (for this command pattern), Deny.
- **Composer** — multiline textarea, slash-command autocomplete (fetch list
  from `/api/capabilities` or hardcode from `hermes_cli/commands.py`), file
  upload (drag-drop, paste), voice memo (Web Speech API + upload to a
  transcription endpoint), model picker chip, "thinking budget" picker for
  reasoning models, attach image button (multi-modal), attach file button.
- **TokenMeter** — live tokens-in / tokens-out / cost in the corner during
  streaming (use the SSE `usage` events that mirror OpenAI).
- **PromptCacheIndicator** — small chip showing cache hits this turn.
- **InterruptButton** — Esc / button → calls the gateway interrupt endpoint
  to abort the current run mid-tool.
- **NewSessionButton** + **SessionTitleHeader** (auto-renamed by agent;
  editable).
- **CopyConversationButton** / **ExportMarkdown** / **ShareLink** (if you add
  a sharing endpoint).
- **TodoSidecar** — live view of the agent's `todo` tool state (it's basically
  the todo list the agent is keeping for itself).
- **SubagentTabs** — when `delegate_task` is in flight, show a tab per
  subagent with its own (nested) ConversationView.

**Data flow:**
- Open WebSocket or SSE to `POST /v1/chat/completions` with `stream: true`,
  pass `X-Hermes-Session-Id` if continuing.
- For long autonomous runs, prefer `POST /v1/runs` → `GET /v1/runs/{id}/events`
  (SSE) so you can disconnect and re-attach.

### 7.2 Sessions / History

- **SessionList** — virtualised, infinite scroll. Per row: title, last message
  preview, model, timestamp, platform icon, token count, cost. `GET /api/sessions`.
- **SearchBar** — debounced; hits `GET /api/sessions/search?q=...&limit=...`.
  Server returns hits w/ snippet highlighting (it does the FTS5 +
  LLM-summarised recall).
- **SessionDetailModal / Drawer** — opens transcript via
  `GET /api/sessions/{id}/messages`; "Continue this session" button (resumes
  with that `session_id`); Delete; Export.
- **Filters** — platform (CLI/api/Telegram/…), date range, model, has-errors.

### 7.3 Files / Workspace

The desktop app has a workspace browser with time-travel snapshots. For VPS,
the workspace is whatever directory you point Hermes at (`config.yaml >
workspace_root`). You need either:
1. A shell-back endpoint on the admin server that does directory listing /
   file read / file write (you'd add this — it's not in stock Hermes), OR
2. Drive the agent itself via tool calls (less direct).

**If you build (1) — recommended:**

- **FileTree** — lazy-load directories, multi-select.
- **MonacoEditor** (or CodeMirror 6) — syntax highlighting, language detection,
  read-only-by-default, save on cmd-S.
- **DiffView** — side-by-side, line-by-line; powered by `diff` library.
- **SnapshotTimeline** — right panel showing every snapshot of the active file
  with relative timestamps ("just now", "12m ago"). The agent's edits go
  through `tools/file_state.py` which already maintains `.history/`.
- **RestoreSnapshotButton** — restores from a snapshot (which itself snapshots
  current first).
- **FavoritesSection** + **MemoriesSection** + **SkillsSection** — like the
  desktop app: pin files, view memories adjacent to the file, view related
  skills.

### 7.4 Terminal

- **TerminalTabs** + **TerminalPane** powered by `xterm.js` + an admin
  endpoint that opens a PTY and streams I/O over WebSocket. Optionally back
  this with Docker/SSH for isolation (see Hermes's terminal backends).
- **BackgroundProcessList** — the agent's `process` tool runs jobs in the
  background; expose them here with start/stop/tail buttons. Backed by
  `tools/process_registry.py`.
- **CommandHistory** + **CommandPalette** with approval-required indicator.

### 7.5 Computer Use (Linux VPS: skip or remote-attach)

Native OCR is macOS / Windows only. On a Linux VPS you have two options:
1. Skip the feature.
2. Remote computer-use: the user's local machine runs the `computer-use-mcp`
   server; your UI tunnels the agent's clicks down to it over an authenticated
   WS. Out of scope of this doc, but the npm package is open source:
   `@atomicbotai/computer-use-mcp`.

If you do build the **viewer** UI:
- **ScreenViewport** — last screenshot, scaled, with green OCR bounding boxes
  overlaid (Apple Vision / WinRT OCR data attached to every screenshot).
- **AgentActiveOverlay** — pulse border whenever the agent is "driving".
- **SessionLockIndicator** — to prevent two agents fighting.

### 7.6 Skills

- **SkillsList** — installed skills (toggle, edit, remove).
- **SkillsHub** — searchable catalog from agentskills.io. Install button →
  `POST /api/skills/install` (you may need to add this; in stock Hermes the
  agent installs them itself via the `skill_manage` tool).
- **SkillEditor** — markdown editor with front-matter (name, description,
  trigger).
- **TriggerBadges** — show the slash command each skill installs.
- **AgentWroteThis** flag for skills the agent created itself.

### 7.7 Memory

- **MemoryFileList** — `GET /api/memory` (admin extension).
- **MemoryEditor** — markdown editor.
- **MemoryTimeline** — when the agent wrote/updated each piece.
- **UserProfileView** — Honcho dialectic profile (Q&A pairs the agent has
  inferred). Read-only with "forget" button per item.

### 7.8 Cron / Jobs

- **JobsTable** — name, schedule (humanised — use `cronstrue`), next run, last
  run, status, last result snippet. `GET /api/cron/jobs`.
- **JobEditorDrawer** — name; prompt (full chat composer); cron expr (with a
  cron-builder widget); toolsets to enable (multi-select); delivery target
  (chat / messenger / email / webhook); enabled toggle.
- **JobRunHistory** — per-job: timestamps, status, output, tokens used.
- **Buttons** — Trigger now, Pause, Resume, Delete.

### 7.9 Tools & Toolsets

- **ToolsetGrid** — card per toolset from `GET /api/tools/toolsets` with
  description, tool list, enable toggle.
- **ToolList** — flat 40+ tool list with per-tool docs (pulled from each
  tool's docstring — they're already in the registry).
- **ToolConfig** — per-tool settings (e.g. for `web_search`: which provider —
  Tavily, Exa, SearXNG; for `image_generate`: which provider).

### 7.10 MCP Servers

- **MCPServerList** — name, transport, status (running/errored/disconnected),
  tool count. `GET /api/mcp/servers`.
- **AddServerDialog** — name, transport (stdio/http), command + args + env
  (for stdio) or URL (for http), OAuth-enabled flag.
- **MCPLogs** — tail per-server stdout/stderr.
- **ToolPreview** — once connected, list the tools the server exposed.

### 7.11 Providers / Models

- **ProvidersGrid** — card per provider: logo, status (configured / not), key
  status, "Sign in" button (kicks OAuth flow if applicable),
  "Add API key" inline form. `GET /api/providers/oauth` and the extension
  layer's `GET /api/providers`.
- **OAuthModal** — runs device-code flow: show user code + verification URL +
  copy button + countdown; polls `/api/providers/oauth/{id}/poll/{session}`.
- **ModelPicker** — searchable list grouped by provider; shows context length,
  pricing per 1M tokens (from `models.dev`), supports-vision flag,
  supports-reasoning flag. `POST /api/model-switch` to swap.
- **ActiveModelChip** — sits in the top bar of every screen; click to swap.

### 7.12 Analytics

- **UsageOverview** — total tokens this period, total cost, top model, top
  toolset, daily series chart. `GET /api/analytics/usage`.
- **PerModelBreakdown** — table with model, input tokens, output tokens,
  cached tokens, cost.
- **PerSessionDrilldown** — clickable; opens the session.
- **DateRangePicker** — last 24h / 7d / 30d / custom.

### 7.13 Logs

- **LogViewer** — virtualised, follow-mode (live tail), filter by level / file
  (`agent.log`, `errors.log`, `gateway.log`) / session id, regex search,
  copy-line. `GET /api/logs?level=&session=&follow=true&limit=`.
- **ErrorDetailModal** — pretty stack-trace renderer.

### 7.14 Settings

Tabs (mirror the existing desktop `SettingsPage.tsx` and the existing web
`ConfigPage.tsx`):
- **General** — workspace root, default profile, language, theme, send-on-enter.
- **AI / Models** — default model, fallback model, max iterations, reasoning
  budget, prompt-caching toggle, thinking-content visibility.
- **API Keys / Env** — table view of every env var with category & description,
  edit/reveal/delete. `GET /api/env`. Reveals require re-auth
  (`POST /api/env/reveal`).
- **Toolsets** — enable/disable toolsets.
- **MCP Servers** — see 7.10.
- **Skills** — see 7.6.
- **Messaging Gateway** — see 7.15.
- **Profiles** — multi-tenant.
- **Advanced** — raw YAML editor (`GET /api/config/raw`, `PUT /api/config/raw`).
- **About** — version, links, "Update Hermes" (`POST /api/hermes/update`).

### 7.15 Messaging Gateway

For each platform (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, SMS,
Email, Matrix, Mattermost, Microsoft Teams, Feishu, DingTalk, WeChat Work,
Weixin, QQ Bot, BlueBubbles, Home Assistant, Webhook):

- **PlatformCard** — logo, enable toggle, status dot.
- **PlatformConfigDrawer** — platform-specific fields (token, webhook URL,
  channel allowlist, user allowlist, command prefix, persona override).
- **PlatformStatusPanel** — uptime, last error, message volume.
- **GatewayRestartButton** — `POST /api/gateway/restart` (applies new config).

### 7.16 Profiles (top-right switcher)

- **ProfileSwitcher** — list current profiles, switch, create new, delete
  (with confirmation).
- **PerProfileColorAccent** — workplace-style coloured corner so the user
  always knows which profile they're talking to.

### 7.17 Cross-cutting

- **CommandPalette** (cmd-K) — search across sessions, skills, tools, settings,
  files; trigger actions.
- **NotificationCenter** — toast feed + persistent inbox of cron results,
  long-run completions, gateway errors.
- **GlobalApprovalQueue** — if multiple sessions request approvals at once,
  surface them all here.
- **OfflineBanner** — when the WebSocket/SSE drops.
- **DarkLightSystemTheme** + custom themes from `/api/dashboard/themes`.
- **PWA / Mobile shell** — install as standalone, support gestures, support
  iOS share sheet (so the user can paste from anywhere into the agent).

---

## 8. Authentication & multi-user concerns

Stock Hermes is **single-user**: `API_SERVER_KEY` is one shared secret. For a
"my app, anywhere" deployment, layer auth above:

**Minimum viable (single user, just you):**
1. Strong `API_SERVER_KEY`, only known to your client app.
2. nginx with TLS + IP allowlist (optional).

**If you want real user accounts:**
1. Your app gets its own Postgres/SQLite with users + per-user bearer tokens.
2. An **auth proxy** layer (could be a tiny FastAPI in front) that:
   - Validates the user's token,
   - Looks up the user's Hermes profile,
   - Injects `Authorization: Bearer $HERMES_KEY` and `X-Hermes-Profile: <user-id>` upstream,
   - Rate-limits per user.
3. The gateway's profile registry will spin up one worker per active profile
   on demand. Idle workers get reaped (see
   `gateway/platforms/api_profile_runtime.py`).
4. Run the file/terminal endpoints **per-user** under a confined chroot or
   container (each user's `workspace_root` must not escape their home).

**Threats to design for early:**
- The agent can run arbitrary shell. **Sandbox the terminal backend.** Use the
  Docker or Modal terminal backend in `tools/environments/` so commands run
  inside an ephemeral container, not on the VPS host.
- The agent can hit the network. Egress controls (nginx outbound, or a
  firewall rule) limit data exfiltration risk.
- Approval modal flows are your last line of defence — surface them
  *aggressively* and don't auto-dismiss.

---

## 9. Suggested tech stack for the UI

Pick what you like, but here's a balanced choice if you want recommendations:

| Layer | Recommendation | Why |
|---|---|---|
| Framework | **Next.js 15 + React 19** (App Router, RSC for static pages, client comps for chat) | One codebase deploys to web + Capacitor for mobile shell |
| State | **Zustand** for client state, **TanStack Query** for server cache + SSE/WS | Already what the desktop app uses (`renderer/src/store`) |
| Styling | **Tailwind v4 + shadcn/ui + Radix** | Fast iteration, ships well-themed primitives |
| Markdown | **react-markdown + remark-gfm + remark-math + rehype-katex + Shiki** | Matches what the desktop chat does |
| Editor | **Monaco** (desktop) or **CodeMirror 6** (lighter for mobile) | Both used in the existing desktop |
| Terminal | **xterm.js** + `xterm-addon-fit` + `xterm-addon-web-links` | Industry standard |
| Diff | **diff2html** or roll your own with `diff` + Monaco's diff editor | |
| Charts | **Recharts** or **visx** | Analytics dashboards |
| Forms | **React Hook Form + Zod** + render from `/api/config/schema` | Schema-driven settings page |
| Streaming | **EventSource** for SSE (with reconnect), **native WebSocket** for the chat | EventSource is simpler; WS is faster |
| Mobile | **Capacitor** wraps the same Next build with native shell + push notifications + share-target | One repo, three platforms |
| Auth | **Auth.js (NextAuth)** with credentials/OAuth + a thin token-issue endpoint | Battle-tested |
| Hosting | UI on Vercel/Cloudflare Pages, agent on your VPS | Cheap and decoupled |

If you'd rather **reuse what already exists**: the repo's `web/` dashboard is a
working Vite + React + Tailwind admin app (see `web/src/pages/` — Status,
Sessions, Config, Env, Cron, Skills, Analytics, Logs). It's not a chat UI but
it covers ~70% of the settings surface and you can fork it.

---

## 10. Phased build plan (suggested)

**Phase 0 — VPS up, agent reachable** (1–2 days)
- Provision Linux VPS, install Python 3.11 + uv, clone repo, `uv sync`.
- Drop a strong `API_SERVER_KEY` in `~/.hermes/.env`.
- Add 1–2 provider keys (OpenRouter is the easiest — 200+ models on one key).
- systemd units for `gateway/run.py` and `hermes_cli/web_server.py`.
- Caddyfile with auto-HTTPS in front of both.
- Smoke test: `curl https://your-vps/v1/models` with the bearer.

**Phase 1 — Chat MVP** (1 week)
- Next.js scaffold, login, single chat screen, SSE streaming.
- Render `stream_delta` + `reasoning_delta` + `tool_start`/`tool_complete` as
  collapsible cards.
- Wire the Approval Modal — non-negotiable for safety.
- Model picker chip in the header.

**Phase 2 — Sessions + Settings** (1 week)
- Sessions list + search + drilldown + delete.
- Providers screen with OAuth flows + add-key form.
- Toolsets toggles.

**Phase 3 — Files + Terminal** (1–2 weeks)
- File tree + Monaco editor + diff/snapshot panel.
- xterm.js terminal over WS (start with one tab, add tabs later).
- This is where you also add a tiny FastAPI in front for the file/terminal
  endpoints that aren't in stock Hermes.

**Phase 4 — Power features** (2 weeks)
- Skills (browse + install + edit).
- Memory viewer.
- Cron / Jobs (the killer feature — set up "every Monday morning, summarise
  what shipped last week to Slack").
- MCP server management.
- Analytics + Logs.

**Phase 5 — Multi-user & gateway** (2+ weeks)
- Profile-per-user with the gateway's profile runtime.
- Auth proxy for token issuance.
- Messaging gateway config screens (so the user can hook up Telegram/Slack
  from the UI).
- Mobile shell via Capacitor.

**Phase 6 — Polish**
- Approvals queue, command palette, theme system, PWA install, push
  notifications, sharing.

---

## 11. Source map (where to read the code yourself)

| What | File |
|---|---|
| Agent core (`AIAgent` class, conversation loop) | `run_agent.py` (~12k LOC) |
| Tool registry + dispatch | `model_tools.py`, `tools/registry.py` |
| Toolset definitions | `toolsets.py` |
| Sessions DB (SQLite + FTS5) | `hermes_state.py` |
| CLI orchestrator | `cli.py` (~11k LOC) |
| Slash command registry | `hermes_cli/commands.py` |
| Setup wizard | `hermes_cli/setup.py` |
| Model switching logic | `hermes_cli/model_switch.py` |
| Config + env file handling | `hermes_cli/config.py` |
| OAuth (provider sign-in) | `hermes_cli/auth.py`, `agent/google_oauth.py`, `hermes_cli/copilot_auth.py` |
| **Gateway HTTP API (the main one)** | `gateway/platforms/api_server.py` |
| Gateway extension routes | `gateway/platforms/api_extensions.py` (+ `docs/api-extensions.md`) |
| Profile registry / per-user worker | `gateway/platforms/api_profile_registry.py`, `api_profile_runtime.py`, `api_profile_worker.py` |
| **Admin web server (44 endpoints)** | `hermes_cli/web_server.py` |
| Desktop chat WebSocket reference | `desktop/src/python-server/server.py`, `bridge.py` |
| Desktop gateway boot | `desktop/src/python-server/desktop-gateway.py` |
| Existing dashboard (forkable) | `web/src/` (pages: Status, Sessions, Config, Env, Cron, Skills, Logs, Analytics) |
| Existing desktop chat UI (reference) | `desktop/renderer/src/ui/chat/`, `services/sse-chat.ts` |
| Cron scheduler | `cron/` |
| Messaging gateway adapters (18+) | `gateway/platforms/` |
| Built-in skills (25 categories) | `skills/` |
| Optional skills | `optional-skills/` |
| Memory provider plugins (Honcho, Mem0, Supermemory) | `plugins/memory/` |
| Image gen providers | `plugins/image_gen/`, `agent/image_gen_registry.py` |
| Terminal backends | `tools/environments/` |
| MCP integration | `tools/mcp_tool.py`, `tools/mcp_oauth.py`, `mcp_serve.py` |
| Architecture deep-dive | `AGENTS.md` (751 lines — read this) |
| Release notes (feature changelog) | `RELEASE_v0.*.md` |

---

## 12. Open questions you'll want to answer before coding

1. **Single user or multi-tenant?** Changes everything downstream (auth proxy,
   profile-per-user, isolation level for terminal/files).
2. **Mobile-first or desktop-first?** Affects whether Capacitor wrap or Tauri
   wrap is on the roadmap from day one.
3. **Will users self-host the messaging gateway, or do you host it?**
   Self-host = simpler infra, harder UX. You-host = need webhook routing per
   user.
4. **Do you need the file workspace + terminal at all?** If your users are
   non-developers, you can drop these and have a chat-only product that's
   way smaller.
5. **How much agent autonomy?** Set the default `max_iterations` and
   approval-required policy conservatively at first; loosen per user.
6. **Billing?** If you're charging, Hermes already tracks usage per session —
   `GET /api/analytics/usage` is your meter.

---

**Bottom line:** the agent backend is already complete and battle-tested. Your
build is essentially "modern web UI on top of two already-existing Python HTTP
servers", plus a thin file/terminal endpoint you'll add, plus an auth proxy if
you want multi-user. Start with the SSE chat against `/v1/chat/completions`
behind a bearer token — you'll have a working remote Hermes in a weekend, and
then it's just feature surface from there.
