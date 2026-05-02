# Contributor Playbook

This is a practical guide for changing the repository without violating its boundaries.

## Before Changing Code

Read, in order:

1. `AGENTS.md`
2. Relevant module `AGENTS.md`:
   - `apps/AGENTS.md`
   - `packages/AGENTS.md`
   - `tools/AGENTS.md`
3. The nearest package manifest.
4. Existing tests near the behavior you are changing.
5. Contract files before touching divergent web/daemon shapes.

This snapshot is not a Git checkout in this workspace, so use file inspection instead of relying on Git metadata.

## Standard Commands

Setup:

```bash
corepack enable
pnpm install
```

Run web plus daemon in foreground:

```bash
pnpm tools-dev run web --daemon-port 17456 --web-port 17573
```

Run full managed stack:

```bash
pnpm tools-dev
```

Inspect:

```bash
pnpm tools-dev status --json
pnpm tools-dev logs --json
pnpm tools-dev inspect desktop status --json
pnpm tools-dev inspect desktop screenshot --path /tmp/open-design.png
```

Validate:

```bash
pnpm typecheck
pnpm test
pnpm build
pnpm check:residual-js
```

For narrower work:

```bash
pnpm --filter @open-design/web typecheck
pnpm --filter @open-design/daemon test
pnpm --filter @open-design/desktop build
pnpm --filter @open-design/tools-dev build
pnpm --filter @open-design/tools-pack build
```

## Common Change Recipes

### Add A Web/Daemon API Field

1. Update `packages/contracts/src/api/...`.
2. Update daemon route handling in `apps/daemon/src/server.ts` or a daemon module.
3. Update web provider helper in `apps/web/src/providers/...`.
4. Update UI consumers.
5. Add or update tests.

Do not put Node, Express, browser, or daemon-specific imports into `packages/contracts`.

### Add An SSE Event

1. Add the event or payload to `packages/contracts/src/sse/...`.
2. Emit it from daemon run handling.
3. Translate it in `apps/web/src/providers/daemon.ts`.
4. Render it in chat/tool UI.
5. Add parser and provider tests.

### Add A New Agent CLI

1. Add an agent definition to `apps/daemon/src/agents.ts`.
2. Prefer stdin prompt delivery if the CLI supports it.
3. Decide stream parser:
   - Structured JSONL: extend `json-event-stream.ts`.
   - Custom stream: new parser module.
   - Plain text: stdout fallback.
4. Add model discovery only if the CLI has a stable command.
5. Add tests for args, model sanitization, and parser behavior.

### Add A New Skill

1. Create `skills/<id>/SKILL.md`.
2. Use front matter with `name`, `description`, `triggers`, and optional `od` metadata.
3. Add assets and references only when they genuinely improve repeatability.
4. If assets/references exist, use explicit paths in the skill body; the daemon will inject the skill-root preamble.
5. Verify with `/api/skills` and a real project run.

### Add A New Design System

1. Create `design-systems/<id>/DESIGN.md`.
2. Put a clean H1 title.
3. Add optional `> Category:` and `> Surface:` metadata near the top.
4. Include extractable hex colors with semantic names.
5. Verify it appears in `/api/design-systems` and the picker swatches look right.

### Add Media Provider Support

1. Add model metadata in `apps/daemon/src/media-models.ts`.
2. Add provider config surface in `apps/daemon/src/media-config.ts` and UI settings if needed.
3. Route provider behavior in `apps/daemon/src/media.ts`.
4. Keep project-relative output and input-image validation intact.
5. Do not silently return placeholder bytes unless `OD_MEDIA_ALLOW_STUBS=1`.

### Change Sidecar Lifecycle

1. Start in the right layer:
   - Protocol constants and stamp validation: `packages/sidecar-proto`.
   - Generic runtime paths/IPC/ports: `packages/sidecar`.
   - Process scanning/spawn/cleanup: `packages/platform`.
   - Dev CLI orchestration: `tools/dev`.
   - Packaged orchestration: `apps/packaged`.
2. Preserve the five stamp fields.
3. Do not hand-build `--od-stamp-*` args in orchestration code.
4. Validate two namespaces when namespace/path behavior changes.
5. Check logs with `pnpm tools-dev logs --namespace <name> --json`.

## Test Map

Important test locations:

- `apps/daemon/tests/agents.test.ts`
- `apps/daemon/tests/json-event-stream.test.ts`
- `apps/daemon/tests/system-prompt-template.test.ts`
- `apps/daemon/tests/artifact-manifest.test.ts`
- `apps/daemon/tests/server-paths.test.ts`
- `apps/web/src/providers/sse.test.ts`
- `apps/web/src/providers/registry.test.ts`
- `apps/web/src/artifacts/renderer-registry.test.ts`
- `packages/sidecar-proto/src/index.test.ts`
- `packages/sidecar/src/index.test.ts`
- `packages/platform/src/index.test.ts`
- `tools/dev/src/diagnostics.test.ts`
- `e2e/specs/app.spec.ts`

## High-Risk Areas

- Prompt composition: small wording/order changes can change every generation.
- Agent spawn args: Windows command-line length and `.cmd` shell behavior are easy to regress.
- SSE run reattachment: event ids and terminal status must remain coherent.
- File paths: traversal protections must not be weakened.
- Sidecar stamps: process discovery depends on exact field names and validation.
- Packaged paths: never tie data/log/cache/runtime roots to ports.
- Deck framework: deck export depends on predictable navigation and print behavior.

## Practical Debugging

If the web app cannot talk to daemon:

```bash
pnpm tools-dev status --json
pnpm tools-dev logs daemon
curl http://127.0.0.1:<daemon-port>/api/health
```

If an agent is unavailable:

```bash
curl http://127.0.0.1:<daemon-port>/api/agents
```

If media generation sees `OD_DAEMON_URL` with port `0`, restart managed runtime and start a fresh project run:

```bash
pnpm --filter @open-design/daemon build
pnpm tools-dev restart --daemon-port 7457 --web-port 5175
```

If desktop opens the wrong page, inspect web sidecar status first. Desktop should discover the web URL through IPC, not by guessing.

```bash
pnpm tools-dev inspect web status --json
pnpm tools-dev inspect desktop status --json
```

## Architecture Smell Checklist

Before submitting a change, ask:

- Did I update contracts before web/daemon shapes diverged?
- Did I keep `packages/contracts` pure?
- Did I keep sidecar concerns out of ordinary app business logic?
- Did I avoid adding root lifecycle aliases?
- Did I avoid restoring removed directories?
- Did I avoid hand-building stamp flags?
- Did I keep generated/runtime output out of source?
- Did I run the smallest relevant test plus broad validation when the boundary is shared?

