# Key concepts

Definitions for every term used across the OD codebase and docs. Alphabetical within each group.

---

## Core abstractions

**`<artifact>` tag** — The XML wrapper the agent emits at the end of every generation turn. It contains a complete, standalone HTML document (or references to project files for multi-file artifacts). The web app's parser (`apps/web/src/artifacts/parser.ts`) extracts the content and renders it in the sandboxed preview iframe. After `</artifact>`, the agent stops — no narration, no summary.

**DESIGN.md** — A 9-section plain-text design system specification: Visual Theme & Atmosphere · Color Palette & Roles · Typography Rules · Component Stylings · Layout Principles · Depth & Elevation · Do's and Don'ts · Responsive Behavior · Agent Prompt Guide. The daemon injects the active DESIGN.md into the system prompt as authoritative tokens. Any agent editing the project can also `Read` it from the project's working directory. Format originated from [`VoltAgent/awesome-design-md`](https://github.com/VoltAgent/awesome-design-md); 72 systems ship with OD.

**Design system** — The combination of a `DESIGN.md` file and (optionally) its rendered preview. In OD, "the active design system" is the one currently loaded for a project. Switching it changes the tokens injected into the next generation's system prompt. Compare to "design system" in Claude Design, which is generated from team files and stored inside Anthropic's platform.

**Direction** — One of five curated visual packages offered when the user has no brand. Each direction is a deterministic spec: OKLch palette (background, surface, foreground, muted, border, accent), display + body + mono font stacks, and layout posture cues. The five directions are Editorial/Monocle, Modern Minimal, Tech Utility, Brutalist, and Soft Warm. Defined in `apps/daemon/src/prompts/directions.ts`. Picking one radio binds the entire spec verbatim into the seed template's `:root` — no model improvisation.

**Mode** — The top-level workflow container that determines what kind of artifact OD produces. Four modes: `prototype` (single editable HTML/JSX screen), `deck` (multi-slide HTML presentation), `template` (pre-designed artifact the agent populates), `design-system` (produces a `DESIGN.md` file). Each mode uses a different skill type and has different export options. See [`docs/modes.md`](../modes.md).

**Project** — The top-level OD work unit. Each project has a unique ID, a working directory at `.od/projects/<id>/`, and a row in the SQLite `projects` table. A project has one mode, one active skill, one active design system, a conversation history, and an open-files list. Projects persist across daemon restarts.

**Skill** — A folder containing `SKILL.md` + `assets/` + `references/`. Skills are the atomic unit of design capability: they define what kind of artifact to produce and how to produce it (step-by-step workflow, seed template, layout library, quality checklist). OD follows the Claude Code [SKILL.md convention](https://docs.anthropic.com/en/docs/claude-code/skills) with optional `od:` frontmatter extensions. 31 skills ship with OD. See [`docs/skills-protocol.md`](../skills-protocol.md).

**SKILL.md** — The manifest file for a skill. The YAML frontmatter declares name, description, triggers, and optional `od:` extensions (mode, preview type, design-system sections required, typed input schema, live parameter sliders, required agent capabilities). The Markdown body is the workflow the agent follows — the numbered step list, principles, and constraints. Skills written for plain Claude Code run in OD without modification.

---

## Architecture

**Daemon** — The local Node 24 + Express process that is OD's privileged layer. It owns: agent detection and spawning, skill registry, design-system resolver, SQLite database (`projects`, `conversations`, `messages`, `tabs`, `templates`), artifact lint, export pipeline, and the BYOK proxy. It listens on `localhost:7456` by default. Secrets never leave it.

**Daemon SSE** — Server-Sent Events stream from `POST /api/chat`. The daemon opens an SSE connection for each chat turn and pushes typed events as the agent runs: `text_delta`, `tool_call`, `tool_result`, `thinking`, `todo_update`, `artifact`, `done`, `error`. The web app consumes this stream to update the UI in real time.

**Namespace** — A runtime isolation identifier for a daemon + web + desktop sidecar group. Used by `tools-dev` to run multiple independent OD instances (e.g. Playwright test isolation, beta channel, production) without sharing a SQLite file or port. Set via `--namespace <name>`.

**Prompt stack** — The ordered assembly of up to seven layers the daemon composes into the system prompt before dispatching to an agent. In priority order: DISCOVERY_AND_PHILOSOPHY → OFFICIAL_DESIGNER_PROMPT → active DESIGN.md → active SKILL.md + pre-flight → project metadata → DECK_FRAMEWORK_DIRECTIVE (decks) → MEDIA_GENERATION_CONTRACT (media). See [The prompt stack](the-prompt-stack.md).

**Sidecar** — A lightweight process that wraps an app (daemon, web, desktop) and exposes a JSON-RPC IPC channel at `/tmp/open-design/ipc/<namespace>/<app>.sock`. The sidecar carries five fields: `app`, `mode`, `namespace`, `ipc`, and `source`. Used by `tools-dev inspect desktop status|eval|screenshot` for headless E2E automation. Defined in `packages/sidecar-proto/` and `packages/sidecar/`.

**Stamp** — The five-field process identifier attached to every OD process: `app` (daemon/web/desktop), `mode` (dev/prod), `namespace`, `ipc` (socket path), `source` (process origin). Stamps are how `tools-dev` finds and controls the right process without port-scanning.

**Surgical edit** — An agent capability (`AgentCapabilities.surgicalEdit`) that allows targeted modification of a specific region of a file without rewriting the whole file. Claude Code supports this natively via its `Edit` tool. Agents without this capability (e.g. Gemini CLI) fall back to regenerating the whole file with a scoped instruction. The UI hides comment mode when the active agent lacks this capability.

---

## Agent adapters

**ACP (Agent Client Protocol)** — A JSON-RPC over stdio protocol used by Hermes, Kimi, and Kiro CLI. The daemon drives an `initialize` → `session/new` → `session/prompt` lifecycle and maps ACP events to typed `AgentEvent` objects for the UI. Stream format: `acp-json-rpc`.

**BYOK (Bring Your Own Key)** — OD's model for API access: you supply credentials directly to the daemon (in `~/.open-design/config.toml`) or the browser (in Settings). OD has no servers and makes no requests on your behalf to any AI provider.

**BYOK proxy** — `POST /api/proxy/stream`. Accepts `{ baseUrl, apiKey, model, messages }`, normalizes the path to `<baseUrl>/v1/chat/completions`, forwards SSE chunks to the browser. Rejects loopback / link-local / RFC1918 destinations to prevent SSRF. Any OpenAI-compatible provider works: Anthropic (via OpenAI shim), DeepSeek, Groq, OpenRouter, self-hosted vLLM.

**`pi-rpc`** — The stdio JSON-RPC protocol used by the Pi CLI. The daemon sends a `prompt` command on stdin and Pi streams back typed events. Distinct from ACP — Pi uses a simpler two-message RPC, not the full ACP session lifecycle. Stream format: `pi-rpc`.

**Stream format** — How the daemon interprets a spawned agent's stdout. Five formats are in use:
- `claude-stream-json` — line-delimited JSON emitted by Claude Code's `--output-format stream-json`
- `copilot-stream-json` — similar JSONL from GitHub Copilot CLI's `--output-format json`
- `json-event-stream` — structured JSON from Codex, Gemini, OpenCode, Cursor Agent (each with a per-CLI `eventParser`)
- `acp-json-rpc` — JSON-RPC over stdio (Hermes, Kimi, Kiro)
- `pi-rpc` — Pi's simpler stdio JSON-RPC
- `plain` — raw stdout chunks (Qwen Code)

---

## UI and workflow

**Discovery form** — The `<question-form id="discovery">` the agent emits on turn 1 of every new project. Captured by the web app's artifact parser and rendered as interactive radio/checkbox/text fields. Locks in surface, platform, audience, tone, brand context, scale, and constraints before any code is written.

**Five-dimensional critique** — A mandatory pre-emit self-assessment the agent runs before wrapping output in `<artifact>`. Scores output 1–5 on philosophy (posture match), hierarchy (focal clarity), execution (typography/spacing precision), specificity (copy is specific to this brief, not filler), and restraint (one accent, one flourish). Dimensions under 3/5 trigger a fix-and-rescore loop.

**`od.parameters`** — Optional SKILL.md frontmatter that declares live-tunable sliders (e.g. `accent_hue`, `section_spacing`, `font-scale`). After the first generation, the UI renders these as sliders. Moving a slider re-prompts the agent with just the changed parameter — no full regeneration.

**Pre-flight directive** — An instruction injected above the skill body in Layer 4 of the prompt stack, generated by `derivePreflight(skillBody)`. It lists the specific side files (`assets/template.html`, `references/layouts.md`, `references/checklist.md`, etc.) the agent must `Read` before doing anything else. Prevents the agent from skipping Step 0 under context pressure.

**Sandboxed preview** — The `<iframe sandbox="allow-scripts" srcdoc="...">` that renders artifact HTML. `allow-same-origin` is absent — the artifact has no access to the host app's cookies, localStorage, or parent DOM. Hot-reloads (debounced 100 ms) on every file write from the agent. JSX artifacts use the same iframe but inject vendored React 18 + Babel standalone into `srcdoc` for evaluation.

**`TodoWrite`** — The Claude Code tool that creates a structured task list displayed as a live "Todos" card in the OD web UI. OD enforces that the agent's first tool call after brand setup is `TodoWrite` with a 5–10 step plan. Steps are marked `in_progress` → `completed` in real time. The card is the primary way users see agent progress and redirect cheaply mid-flight.

---

## Infrastructure

**`tools-dev`** — The single local development lifecycle entry point for OD. `pnpm tools-dev` (no subcommand) starts daemon + web (+ desktop if installed). Other subcommands: `start`, `stop`, `run`, `status`, `logs`, `inspect`, `check`. Port configuration: `--daemon-port` and `--web-port`. Namespace configuration: `--namespace`. Do not use `pnpm dev`, `pnpm daemon`, or `pnpm start` — they bypass `tools-dev`'s port and namespace management.

**`.od/` directory** — The local runtime folder created by the daemon at the project root. Contains: `app.sqlite` (projects / conversations / messages / tabs / templates), `projects/<id>/` (per-project working directory, also the agent's `cwd`), `artifacts/` (one-off saved renders). Gitignored. Reset with `pnpm tools-dev stop && rm -rf .od`.

**`OD_DATA_DIR`** — Environment variable that relocates the `.od/` tree to a custom path relative to the repo root. Used by Playwright tests to isolate test data from the real project database. Set it to a temp path in CI.
