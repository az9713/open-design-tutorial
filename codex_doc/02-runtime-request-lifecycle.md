# Runtime Request Lifecycle

This document follows the core path from "user sends a prompt" to "a file appears in the preview." The best source files are:

- `apps/web/src/components/ProjectView.tsx`
- `apps/web/src/providers/daemon.ts`
- `apps/daemon/src/server.ts`
- `apps/daemon/src/runs.ts`
- `apps/daemon/src/agents.ts`
- `apps/daemon/src/projects.ts`
- `apps/daemon/src/db.ts`
- `packages/contracts/src/api/chat.ts`
- `packages/contracts/src/sse/chat.ts`

## Lifecycle In One Page

1. Web bootstraps app state from `/api/health`, `/api/agents`, `/api/skills`, `/api/design-systems`, `/api/projects`, `/api/templates`, `/api/prompt-templates`, and `/api/version`.
2. User creates or opens a project. Project metadata lives in SQLite, files live on disk under `.od/projects/<projectId>/`.
3. User sends a chat prompt with optional project-file attachments.
4. `ProjectView.handleSend` persists a user message and an assistant placeholder.
5. If local CLI mode is active, web calls `streamViaDaemon`.
6. `streamViaDaemon` posts a `ChatRequest` to `/api/runs`.
7. Daemon creates an in-memory run via `createChatRunService`, responds `202 { runId }`, then starts the agent in the background.
8. Web opens `/api/runs/:id/events` and consumes SSE frames.
9. Daemon composes instructions from skill, design system, project metadata, and user message.
10. Daemon detects/resolves the selected agent binary, chooses prompt delivery mode, and spawns the CLI with cwd set to the project folder.
11. Daemon parses stdout into typed agent events when possible; otherwise it forwards stdout chunks.
12. Web translates run events into chat text, tool cards, status rows, usage, and file-open actions.
13. Agent writes files into `.od/projects/<projectId>/`; the daemon file API lists and serves them.
14. File workspace refreshes and `FileViewer` chooses a renderer based on file kind and artifact manifest.
15. The assistant message persists run metadata, content, events, produced files, and status.

## Bootstrapping

`apps/web/src/App.tsx` starts by loading local UI config and then asking the daemon for:

- `GET /api/health`
- `GET /api/agents`
- `GET /api/skills`
- `GET /api/design-systems`
- `GET /api/projects`
- `GET /api/templates`
- `GET /api/prompt-templates`
- `GET /api/version`

If the daemon is live, the app selects a reasonable default agent, default skill, and default design system. This is why first-run feels automatic: the web app does not hard-code installed tools; the daemon reports what exists.

## Project Creation

`apps/web/src/components/NewProjectPanel.tsx` collects the user's project intent:

- Project kind: prototype, deck, template, image, video, audio, or other.
- Skill id, usually derived from the active tab and skill default metadata.
- Design system id for web-like surfaces.
- Template id for template projects.
- Media model, aspect, length, duration, voice, and prompt template snapshots for media projects.
- Initial prompt and structured metadata.

`apps/web/src/state/projects.ts:createProject` generates a UUID client-side and posts to `POST /api/projects`.

The daemon route in `apps/daemon/src/server.ts` inserts:

- A project row in SQLite.
- A default conversation row.
- Optional pending prompt and metadata.
- A project directory under `.od/projects/<projectId>/` when needed.

## Message Send Path

`ProjectView.handleSend` does the important browser-side coordination:

- Creates the user `ChatMessage`.
- Creates an assistant placeholder with `startedAt`, `agentId`, `agentName`, and status.
- Persists both messages through daemon-backed message APIs.
- Builds `nextHistory`.
- Creates abort controllers for browser subscription and explicit run cancellation.
- Creates handlers for text deltas, agent events, run status, errors, and completion.

Two execution branches exist:

- Local CLI path: `streamViaDaemon` from `apps/web/src/providers/daemon.ts`.
- Direct API fallback path: `streamMessage` from `apps/web/src/providers/anthropic.ts`.

The direct API path still uses the same artifact parser and preview components. Only the transport and prompt delivery differ.

## Modern Daemon Run API

The modern local CLI path uses `/api/runs` instead of only the older `/api/chat` stream:

```text
POST /api/runs
GET  /api/runs
GET  /api/runs/:id
GET  /api/runs/:id/events
POST /api/runs/:id/cancel
```

`/api/chat` still exists as a compatibility route that creates a run and immediately streams it.

The run service in `apps/daemon/src/runs.ts` is in-memory by design. It provides:

- Run ids and metadata.
- Status transitions: queued, running, succeeded, failed, canceled.
- Event buffer with replay by event id.
- SSE client fan-out.
- Cancellation by killing the child process.
- Waiters and terminal cleanup with TTL.

The event replay behavior is why the web can reattach to active runs. `streamViaDaemon` tracks `lastRunEventId`, reconnects with `?after=<id>`, and falls back to `GET /api/runs/:id` if a stream disconnects near completion.

## Daemon Prompt And CWD Setup

Inside `apps/daemon/src/server.ts:startChatRun`, the daemon:

1. Validates `agentId` and `message`.
2. Resolves the selected agent definition from `apps/daemon/src/agents.ts`.
3. Ensures the project directory if `projectId` exists.
4. Lists existing project files and tells the agent not to overwrite unless asked.
5. Validates project attachments so only files inside the project directory survive.
6. Composes the daemon system prompt with `composeSystemPrompt`.
7. Folds the instructions and user request into one composed prompt, because local CLIs usually do not accept a clean separate system channel.
8. Builds extra allowed directories for skills and design systems, so agents can read seed templates and references outside the project cwd when their CLI supports allow-list flags.
9. Sanitizes user-selected model and reasoning options.
10. Chooses prompt delivery strategy.

The cwd hint is important:

```text
Your working directory: <.od/projects/projectId>
Write project files relative to it.
Files already in this folder ...
```

That sentence is the bridge between the chat UI and the filesystem. It gives the agent a real workspace and tells the UI where to look for results.

## Prompt Delivery Strategy

The daemon handles Windows and CLI-specific limits carefully:

- Preferred path for agents that support it: prompt via stdin.
- Fallback path for long prompts on Windows: write a temporary prompt file into the project cwd and pass a short bootstrap instruction telling the agent to read it.
- Direct argv path only when it is small enough and the selected CLI expects prompt text in args.

This avoids `spawn ENAMETOOLONG` and Windows `.cmd` shell-length failures. The code explicitly warns future contributors not to add new prompt-bearing argv flags when stdin is available.

## Agent Spawn

Agent definitions live in `apps/daemon/src/agents.ts`. Each definition specifies:

- `id`, `name`, `bin`
- model options and optional live model discovery
- reasoning options when supported
- argument builder
- whether prompt travels through stdin
- stream format/parser
- optional capability probes

The daemon resolves the actual binary path before spawning. If the binary is missing, the run emits a friendly SSE error instead of falling back to `spawn(def.bin)` and producing an opaque `ENOENT`.

The daemon injects media-related env vars into agent runs:

- `OD_BIN`
- `OD_DAEMON_URL`
- `OD_PROJECT_ID`
- `OD_PROJECT_DIR`

Those let a code agent call `od media generate ...` against the same daemon and project.

## Stream Parsing

The daemon can stream several formats:

- `claude-stream-json`: parsed by `apps/daemon/src/claude-stream.ts`.
- `copilot-stream-json`: parsed by `apps/daemon/src/copilot-stream.ts`.
- `json-event-stream`: parsed by `apps/daemon/src/json-event-stream.ts` for Codex, Gemini, Cursor Agent, and OpenCode-like JSONL.
- `acp-json-rpc`: attached through `apps/daemon/src/acp.ts`.
- `pi-rpc`: attached through `apps/daemon/src/pi-rpc.ts`.
- plain stdout: forwarded as `stdout` chunks.

The shared web/daemon shape is defined in `packages/contracts/src/sse/chat.ts`:

- `start`
- `agent`
- `stdout`
- `stderr`
- `error`
- `end`

The browser translates `agent` payloads into UI `AgentEvent` objects in `apps/web/src/providers/daemon.ts`.

## File Detection And Auto-Open

`ProjectView` listens for tool events. When a `tool_use` looks like a write and has a `file_path`, it records the tool id and target basename. When the matching `tool_result` arrives successfully, it refreshes project files and opens the written file in a tab.

This is deliberately event-driven. It avoids waiting until the full turn ends and avoids flickering between synthetic live tabs and real files.

## File Serving And Preview

Project files are managed by `apps/daemon/src/projects.ts` and routes in `server.ts`:

- `GET /api/projects/:id/files`
- `GET /api/projects/:id/raw/*`
- `GET /api/projects/:id/files/:name`
- `PUT /api/projects/:id/files/:name`
- `DELETE /api/projects/:id/files/:name`
- `GET /api/projects/:id/files/:name/preview`

All inbound file paths are validated to prevent path traversal:

- No absolute paths.
- No drive-letter paths.
- No null bytes.
- No `.` or `..` path segments.
- Resolved target must remain under the project directory.

`FileViewer.tsx` selects a viewer based on `ProjectFile.kind` and optional artifact manifest:

- HTML/deck HTML: iframe preview.
- Markdown: safe markdown renderer.
- SVG: SVG viewer.
- Image/video/audio: native media viewers.
- PDF/docx/pptx/xlsx: daemon-produced document preview summary.
- Text/code: text viewer.
- Binary: download/open fallback.

## Persistence Model

There are two persistence planes:

- SQLite for product metadata and chat history.
- Filesystem for generated artifacts and uploads.

SQLite tables are created in `apps/daemon/src/db.ts`:

- `projects`
- `templates`
- `conversations`
- `messages`
- `tabs`
- `deployments`

Generated files stay as files. This is the right tradeoff for a local design product: users can open, diff, export, zip, deploy, and inspect actual artifacts.

## Cancellation

The stop button aborts the browser subscription and sends `POST /api/runs/:id/cancel`. The daemon marks `cancelRequested`, sends `SIGTERM` to the child process if present, and finishes the run as canceled. The UI updates the assistant message with terminal status and `endedAt`.

## Failure Surfaces

Common failure surfaces and where they are handled:

- Daemon unreachable: `daemonIsLive`, bootstrapping state, settings/API fallback.
- Unknown or unavailable agent: `/api/runs` emits `AGENT_UNAVAILABLE`.
- CLI exits non-zero: daemon emits `end` with failure status; browser surfaces stderr tail.
- Prompt too long on Windows: stdin or prompt-file fallback.
- SSE disconnect: browser reconnects by event id, then checks run status.
- Bad project path: project file APIs reject traversal.
- Media provider not configured: media dispatcher returns clear provider configuration errors unless stubs are explicitly enabled.

