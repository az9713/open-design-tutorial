# Product And Repo Map

## Product In One Sentence

Open Design is a local-first design product that lets a web UI orchestrate the coding agent already installed on the user's machine. The output is not a black-box design blob. It is a real project folder of files that the user can inspect, preview, export, deploy, and continue editing.

The current implementation is broader than the older draft product spec in `docs/spec.md`: this snapshot has a Next.js web app, an Express daemon, an Electron desktop shell, and a packaged Electron runtime that launches daemon and web sidecars. For current behavior, treat `AGENTS.md` and source code as the source of truth.

## Mental Model

Open Design has five layers:

1. Product shell: Next.js UI in `apps/web`.
2. Local authority: Express daemon in `apps/daemon`.
3. Agent bridge: daemon child-process spawning and stream parsers in `apps/daemon/src/agents.ts`, `claude-stream.ts`, `copilot-stream.ts`, `json-event-stream.ts`, `acp.ts`, and `pi-rpc.ts`.
4. Shared contracts: DTOs, SSE event shapes, prompt composition, and metadata types in `packages/contracts`.
5. Runtime orchestration: sidecar protocol, generic sidecar runtime, process primitives, `tools-dev`, packaged runtime, and desktop IPC.

The product works because those layers stay separated. The UI does not spawn agents. The daemon does not own desktop discovery. The generic sidecar package does not know which Open Design app is running. Process-scanning code does not hand-write Open Design stamp flags.

## Workspace Map

`pnpm-workspace.yaml` includes:

- `apps/*`
- `packages/*`
- `tools/*`
- `e2e`

Active top-level packages:

| Area | Package | Main responsibility |
|---|---|---|
| Web app | `apps/web` | Next.js 16 App Router UI, settings, project creation, chat, file workspace, preview/export surfaces, browser-side provider paths. |
| Daemon | `apps/daemon` | Express API, SQLite, project file store, agent detection/spawning, SSE runs, skills/design systems, media generation, deploy, artifact manifest validation, static serving. |
| Desktop | `apps/desktop` | Electron host that discovers the web URL through sidecar IPC and exposes desktop inspection methods. |
| Packaged runtime | `apps/packaged` | Electron packaged entry that prepares namespace paths, starts daemon/web sidecars, registers `od://`, then delegates desktop behavior to `apps/desktop`. |
| Contracts | `packages/contracts` | Pure TypeScript shared web/daemon DTOs, SSE unions, errors, task metadata, prompt composer. |
| Sidecar protocol | `packages/sidecar-proto` | Open Design sidecar constants, app/source/mode enums, five-field stamp schema, IPC message schema, status shapes. |
| Sidecar runtime | `packages/sidecar` | Generic runtime path resolution, IPC server/client, port allocation, launch env, runtime file helpers. |
| Platform | `packages/platform` | Generic process stamping, spawn helpers, process listing, process tree cleanup, HTTP wait, log tail. |
| Dev tool | `tools/dev` | Local lifecycle control plane for daemon, web, and desktop. |
| Pack tool | `tools/pack` | Build/install/start/stop/logs/uninstall/cleanup/list/reset and release artifact prep for packaged mac/Windows lanes. |
| E2E | `e2e` | Playwright UI specs, Vitest/jsdom integration tests, case docs, custom reports. |

Removed or inactive boundaries matter:

- Do not recreate `apps/nextjs`; current web runtime is `apps/web`.
- Do not recreate `packages/shared`; shared web/daemon contracts belong in `packages/contracts`, sidecar protocol in `packages/sidecar-proto`, generic sidecar runtime in `packages/sidecar`, generic process primitives in `packages/platform`.
- `.od/`, `.tmp/`, `e2e/.od-data`, Playwright reports, and agent scratch directories are runtime data, not source.

## What The User Sees

The core UI path lives in:

- `apps/web/src/App.tsx`: bootstraps app config, daemon health, agent list, skill list, design systems, projects, templates, prompt templates, and version info.
- `apps/web/src/components/EntryView.tsx`: landing/workspace entry.
- `apps/web/src/components/NewProjectPanel.tsx`: turns a user's project choice into project metadata.
- `apps/web/src/components/ProjectView.tsx`: owns conversations, messages, streaming state, project files, open tabs, run reattachment, and send/stop/export actions.
- `apps/web/src/components/ChatPane.tsx` and `ChatComposer.tsx`: chat surface and attachment staging.
- `apps/web/src/components/FileWorkspace.tsx` and `FileViewer.tsx`: tabbed file browser and renderer selection.

The user chooses or inherits:

- Execution mode: local daemon/agent path or direct API fallback.
- Agent: Claude Code, Codex CLI, Gemini, OpenCode, Qwen, Cursor Agent, Copilot, Pi, ACP-compatible tools, etc., depending on detection.
- Skill: file-based `SKILL.md` workflows under `skills/`.
- Design system: `DESIGN.md` bundles under `design-systems/`.
- Project kind: prototype, deck, template, image, video, audio, or other.

## What The Daemon Owns

The daemon is the local authority. It owns:

- REST and SSE routes under `/api/*` in `apps/daemon/src/server.ts`.
- SQLite migrations and persistence in `apps/daemon/src/db.ts`.
- Project file safety, MIME/kind inference, and project folder layout in `apps/daemon/src/projects.ts`.
- Agent definitions, detection, model options, and CLI argument builders in `apps/daemon/src/agents.ts`.
- Run state, event replay, cancellation, and cleanup in `apps/daemon/src/runs.ts`.
- Skill and design-system registries in `apps/daemon/src/skills.ts` and `design-systems.ts`.
- Artifact manifest validation in `apps/daemon/src/artifact-manifest.ts`.
- Media generation dispatch and provider adapters in `apps/daemon/src/media.ts`.
- Vercel self-deploy helper logic in `apps/daemon/src/deploy.ts`.
- Sidecar wrapper in `apps/daemon/sidecar/server.ts`.

The daemon writes runtime data under `.od/` by default:

- `.od/app.sqlite`: projects, templates, conversations, messages, tabs, deployments.
- `.od/projects/<projectId>/`: generated files, uploads, sketches, images, documents, deck HTML, etc.
- `.od/artifacts/`: older/static artifact save path used by artifact save endpoints.

## What Contracts Own

`packages/contracts` is intentionally pure TypeScript. It defines shapes, not runtime behavior:

- API request/response DTOs in `src/api/*`.
- SSE event unions in `src/sse/*`.
- Error shapes in `src/errors.ts`.
- Task and project metadata in `src/tasks.ts` and `src/api/projects.ts`.
- Prompt composition in `src/prompts/system.ts` and related prompt modules.

This package must not import Next.js, Express, Node filesystem/process APIs, browser APIs, SQLite, daemon internals, or sidecar control-plane details.

## What Sidecar Layers Own

The sidecar stack exists so local development and packaged runtime can start and inspect daemon, web, and desktop consistently.

- `packages/sidecar-proto`: Open Design-specific constants and validation.
- `packages/sidecar`: generic IPC, runtime path, launch env, and port utilities.
- `packages/platform`: process stamps, process discovery, spawning, cleanup.
- `tools/dev`: local process manager that consumes the above primitives.
- `apps/packaged`: packaged process manager that consumes the same primitives.

The five stamp fields are load-bearing:

```text
app, mode, namespace, ipc, source
```

If a process cannot be described by that stamp, `tools-dev` and `tools-pack` cannot reliably discover or clean it up.

## Repo-Wide Invariants

- Use Node `~24` and `pnpm@10.33.2`.
- Use `pnpm tools-dev` as the local lifecycle entry point.
- Use `OD_PORT` for daemon port and `OD_WEB_PORT` for web port. Do not introduce `NEXT_PORT`.
- Keep new project-owned source TypeScript-first. New `.js`, `.mjs`, or `.cjs` files need an explicit generated/vendor/compat reason and must pass `pnpm check:residual-js`.
- Update `packages/contracts` before divergent web/daemon shape changes.
- Sidecar awareness belongs in app sidecar wrappers or desktop sidecar entry code, not ordinary app business logic.
- Packaged paths must be namespace-scoped and independent from port numbers.
- Git commits must not include co-author trailers.

## Current Product Shape Compared To Early Docs

The draft spec in `docs/spec.md` says "we do not ship a desktop app." The current repo does ship:

- `apps/desktop`: Electron shell.
- `apps/packaged`: packaged Electron runtime entry.
- `tools/pack`: packaged build/start/inspect flows.

That is not a contradiction to paper over in onboarding. It is a snapshot evolution. Use early docs for rationale and high-level product bets; use code and `AGENTS.md` for present-day boundaries.

