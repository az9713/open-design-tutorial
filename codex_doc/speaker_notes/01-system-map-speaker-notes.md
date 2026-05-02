# Speaker Notes: Open Design System Map

Use with `codex_doc/infographics/01-system-map.svg`.

## Opening

"This diagram is the whole product on one page. Open Design is not one app process and it is not a hidden model service. It is a web experience, a local daemon, local coding agents, shared contracts, real files on disk, and sidecar orchestration that keeps those processes discoverable."

## 1. apps/web

Point to box 1.

"Start at the user's surface. `apps/web` is the Next.js 16 and React app. It owns the visible experience: project creation, chat, settings, agent picker, skill picker, design-system picker, file workspace, preview, export, and deploy controls."

"The key source path for understanding the UI is `apps/web/src/components/ProjectView.tsx`. That is where conversations, messages, streaming state, project files, tabs, run reattachment, and send/stop/export actions come together."

## 2. apps/daemon

Point to box 2.

"The daemon is the local authority. The browser does not spawn Claude, Codex, or any other CLI directly. The daemon owns Express routes, SQLite, project files, agent detection, child process spawning, stream parsing, skills, design systems, media generation, and deploy."

"The important file is `apps/daemon/src/server.ts`, but it delegates meaningful slices to `agents.ts`, `runs.ts`, `db.ts`, `projects.ts`, `skills.ts`, and `media.ts`."

## 3. Local Agent CLIs

Point to box 3.

"This box is the key product bet. Open Design does not implement its own full agent loop. It delegates to the user's installed coding agent. The daemon gives that agent a prompt, a project cwd, selected skill and design-system context, and then streams the child process output back to the UI."

"That means the agent's own tool loop, permission model, and model selection still matter. Open Design is the integration shell."

## 4. Storage

Point to box 4.

"Outputs are not trapped inside chat state. Product metadata goes into `.od/app.sqlite`, and generated deliverables become normal files under `.od/projects/<projectId>/`. That is why the UI can preview, open, export, deploy, and reattach context to generated work."

## 5. packages/contracts

Point to box 5.

"The contract package is the typed bridge between web and daemon. API DTOs, SSE event unions, errors, project metadata, and prompt composition live here."

"The constraint is important: this package must stay pure TypeScript. No Next, no Express, no Node filesystem, no SQLite, no sidecar internals. If a web/daemon shape changes, contracts change first."

## 6. Sidecar Packages

Point to box 6.

"Sidecars solve process identity. A local dev run or packaged app can have dynamic ports, but it still needs stable runtime identity. The sidecar protocol defines the five-field stamp: app, mode, namespace, ipc, source."

"`sidecar` resolves paths and IPC. `platform` handles process stamps, spawning, listing, cleanup, and logs."

## 7. tools-dev And Packaged Runtime

Point to box 7.

"`tools-dev` starts daemon, web, and desktop in the right order. The packaged runtime does a similar job inside Electron. Desktop does not guess the web port. It asks the web sidecar over IPC for the current URL."

"This keeps multiple namespaces and dynamic ports from colliding."

## 8. Skills

Point to box 8.

"Skills are `SKILL.md` folders. They are workflows for the agent: prototype, deck, media, report, and so on. If a skill ships references or templates, the daemon injects a skill-root preamble so the local agent can read those files from its project cwd."

## 9. Design Systems

Point to box 9.

"Design systems are `DESIGN.md` bundles. The scanner extracts title, category, surface, summary, and color swatches for the picker. The prompt composer treats the selected design system as authoritative tokens."

## 10. Prompt Templates

Point to box 10.

"Prompt templates are currently used for image and video surfaces. The user can pick and edit a template at project creation time; that edited prompt snapshot is stored in project metadata and folded into every relevant turn."

## 11. Tests And E2E

Point to box 11.

"The repo has multiple test layers: daemon tests for agents, stream parsers, system prompt, paths, artifact manifests; web tests for renderers, providers, i18n, file workspace; sidecar package tests; and Playwright e2e."

## Closing

"The most useful contributor rule is at the bottom: contracts first for shared shapes, daemon for local authority, web for user experience, sidecar layers for lifecycle, platform for process primitives. That separation is what keeps the repo understandable."

