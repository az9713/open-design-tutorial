# Agent, Skill, And Prompt Pipeline

The "magic" of Open Design is not a hidden model loop. It is prompt composition plus a local child process running in a real project folder. This document explains that pipeline.

Primary source files:

- `packages/contracts/src/prompts/system.ts`
- `packages/contracts/src/prompts/discovery.ts`
- `packages/contracts/src/prompts/official-system.ts`
- `packages/contracts/src/prompts/deck-framework.ts`
- `packages/contracts/src/prompts/media-contract.ts`
- `apps/daemon/src/skills.ts`
- `apps/daemon/src/design-systems.ts`
- `apps/daemon/src/agents.ts`
- `apps/daemon/src/server.ts`
- `apps/daemon/src/media.ts`
- `apps/daemon/src/media-models.ts`

## The Core Idea

Open Design does not ask a model to "be creative" in the abstract. It gives a local code agent a carefully layered instruction packet:

```text
Discovery and product-quality rules
+ identity/workflow charter
+ optional DESIGN.md
+ optional SKILL.md
+ optional project metadata
+ optional template snapshot
+ optional deck framework directive
+ optional media generation contract
+ cwd and file listing
+ user request
```

The agent then writes files in `.od/projects/<projectId>/`. The UI is mostly a stream reader, file browser, and preview host.

## Prompt Composition Stack

`packages/contracts/src/prompts/system.ts:composeSystemPrompt` builds the shared prompt stack. Both local daemon path and direct API path reuse this function.

The order matters:

1. `DISCOVERY_AND_PHILOSOPHY`
   - Forces discovery forms when metadata is incomplete.
   - Reinforces design critique and task planning.
   - Adds higher-level quality philosophy.
2. `OFFICIAL_DESIGNER_PROMPT`
   - Establishes the expert designer identity and artifact output contract.
3. Active design system body
   - Treats `DESIGN.md` as authoritative for color, typography, spacing, and component rules.
4. Active skill body
   - Tells the agent which workflow to follow for a prototype, deck, template, media surface, report, etc.
   - Adds preflight instructions when a skill ships side files like `assets/template.html` or `references/layouts.md`.
5. Project metadata block
   - Carries the structured choices from the New Project panel.
   - Marks missing choices as "unknown - ask".
   - Inlines selected media prompt-template snapshots.
   - Inlines saved template HTML snapshots for template projects.
6. Deck framework directive
   - Pinned last for deck projects without a skill seed, because deck navigation, counter, scroll, and print behavior are load-bearing for export.
7. Media generation contract
   - Pinned for image/video/audio projects so the agent calls `od media generate` instead of emitting HTML.

## Skills

Skills are folders under `skills/<id>/` with a `SKILL.md` file. The daemon scanner in `apps/daemon/src/skills.ts`:

- Reads `SKILL.md`.
- Parses front matter.
- Infers mode when missing.
- Normalizes surface, platform, scenario, featured priority, defaults, fidelity, speaker notes, and animation hints.
- Derives an example prompt.
- Detects whether the skill ships side files.
- Prepends a skill-root preamble when side files exist.

That preamble is essential. The agent cwd is the project folder, not the skill folder. If a skill says "read `assets/template.html`," the agent needs the absolute skill root to find it.

Current bundled skill categories include:

- Web/product surfaces: `web-prototype`, `saas-landing`, `dashboard`, `pricing-page`, `docs-page`, `blog-post`, `mobile-app`, `kanban-board`, `dating-web`, `gamified-app`.
- Decks: `simple-deck`, `guizang-ppt`, `replit-deck`.
- Office/ops docs: `finance-report`, `eng-runbook`, `pm-spec`, `meeting-notes`, `weekly-update`, `invoice`, `hr-onboarding`.
- Media: `image-poster`, `magazine-poster`, `social-carousel`, `video-shortform`, `hyperframes`, `audio-jingle`, `sprite-animation`, `motion-frames`.
- Utilities: `critique`, `tweaks`, `wireframe-sketch`, `design-brief`.

## Design Systems

Design systems are folders under `design-systems/<id>/` with `DESIGN.md`. The scanner in `apps/daemon/src/design-systems.ts`:

- Reads the first H1 as title.
- Extracts a `> Category:` line.
- Extracts a `> Surface:` line, defaulting to web.
- Summarizes the first paragraph under the title.
- Extracts four representative swatches for picker UI.

The prompt composer treats selected `DESIGN.md` content as authoritative. The user can also choose inspiration design systems through project metadata; those are treated as style references, not replacements for the primary design system.

## Agent Definitions

`apps/daemon/src/agents.ts` is the adapter catalog. Each entry says how to detect and run a CLI:

- Claude Code
- Codex CLI
- Gemini CLI
- OpenCode
- Cursor Agent
- GitHub Copilot CLI
- Pi
- ACP-compatible tools
- Qwen-like flows where present in the definitions

Definitions include:

- Binary name.
- Model options or live model discovery.
- Reasoning options when supported.
- `buildArgs` function.
- `promptViaStdin` boolean.
- Stream format.
- Event parser kind.
- Extra allowed-dir support.
- Capability probing, such as detecting supported Claude flags.

The daemon detects agents by probing binaries, versions, help output, and model commands. Results feed `/api/agents`, and the web UI uses that to populate the agent/model picker.

## Why Local Agents Need A Folded Prompt

Remote APIs often support separate system and user messages. Local coding CLIs vary. Some accept a prompt flag, some read stdin, some have native skills, some discover project instructions, and some emit JSON streams. Open Design normalizes this by composing one large instruction packet, then delivering it through the safest channel the selected agent supports.

This is why `startChatRun` folds:

```text
# Instructions (read first)
<composed system prompt>
<cwd hint>
<existing files list>

---

# User request
<chat transcript>
<attachment hints>
```

The chat transcript is collapsed by the web provider before the daemon receives it. That is a practical choice for single-turn CLI programs.

## Stream Normalization

The UI wants one broad event vocabulary:

- status
- text
- thinking
- tool use
- tool result
- usage
- raw

The daemon adapts per-CLI stdout into this vocabulary:

- Claude stream JSON: `claude-stream.ts`
- Copilot JSONL: `copilot-stream.ts`
- Codex/Gemini/Cursor/OpenCode style JSON events: `json-event-stream.ts`
- ACP JSON-RPC: `acp.ts`
- Pi RPC: `pi-rpc.ts`
- Plain stdout: `stdout` chunks

The browser translation layer is `apps/web/src/providers/daemon.ts:translateAgentEvent`.

## Media Generation Contract

Media is not a special web-only feature. It is agent-driven through the daemon's local CLI:

1. The prompt composer detects image/video/audio project metadata and appends `MEDIA_GENERATION_CONTRACT`.
2. The daemon injects `OD_BIN`, `OD_DAEMON_URL`, `OD_PROJECT_ID`, and `OD_PROJECT_DIR` into the agent environment.
3. The agent calls commands like:

```bash
od media generate --surface image --model <model> --output <file> --prompt <prompt>
```

4. `apps/daemon/src/cli.ts` posts to `/api/projects/:id/media/generate`.
5. `apps/daemon/src/media.ts` routes to a provider or fails with a clear provider configuration error.
6. Generated bytes are written into the project folder.
7. `FileViewer` renders the resulting image, video, or audio file.

This contract keeps media generation inside the same project-file and preview pipeline as HTML, Markdown, SVG, and documents.

## Safety And Trust Boundaries

Important safeguards:

- Attachments are only accepted if they resolve under the project cwd.
- `--image` inputs for media generation are capped, extension-allowlisted, and must stay under project cwd.
- Project file paths reject absolute paths, null bytes, drive letters, `.` and `..` segments.
- External API proxy blocks private/internal IPs.
- Prompt template markdown fences are escaped before insertion into the system prompt.
- Long prompt delivery avoids Windows shell length limits.
- Skills and design-system directories are only extra-readable when the selected agent supports that affordance.

The remaining trust boundary is inherent: the selected local coding agent can write files in its cwd and may execute commands depending on its own permissions. Open Design's posture is to constrain cwd, surface what is happening, and let the user's chosen agent enforce its tool policy.

## Where To Change Things

| Change | Best place |
|---|---|
| Add or adjust shared chat request shape | `packages/contracts/src/api/chat.ts` first, then daemon/web. |
| Add a new SSE event | `packages/contracts/src/sse/chat.ts`, daemon emitter/parser, web translator. |
| Add a new agent CLI | `apps/daemon/src/agents.ts`, stream parser if needed, tests in `apps/daemon/tests/agents.test.ts` or parser tests. |
| Change prompt stack | `packages/contracts/src/prompts/system.ts` and prompt tests. |
| Add a skill | `skills/<id>/SKILL.md` plus assets/references if needed. |
| Add a design system | `design-systems/<id>/DESIGN.md`. |
| Add media model/provider | `apps/daemon/src/media-models.ts`, `media-config.ts`, `media.ts`, UI model lists. |

