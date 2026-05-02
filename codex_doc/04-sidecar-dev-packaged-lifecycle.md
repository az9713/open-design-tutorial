# Sidecar, Dev, Desktop, And Packaged Lifecycle

Open Design has two runtime control planes:

- `tools-dev`: local development lifecycle.
- packaged runtime: Electron entry in `apps/packaged`.

Both rely on the same lower layers:

- `packages/sidecar-proto`
- `packages/sidecar`
- `packages/platform`

## Why Sidecars Exist

The app has several processes:

- daemon HTTP server
- web Next.js server
- desktop Electron process
- packaged Electron root

Those processes need to be started, inspected, logged, stopped, and separated by namespace. Ports are not stable identities. A process stamp and IPC path are stable identities.

## The Five-Field Stamp

`packages/sidecar-proto/src/index.ts` defines the stamp fields:

```text
app
mode
namespace
ipc
source
```

Examples:

- `app`: daemon, web, desktop
- `mode`: dev or runtime
- `namespace`: default or user-specified namespace
- `ipc`: named pipe on Windows or socket path on POSIX
- `source`: tools-dev, tools-pack, packaged

`packages/platform:createProcessStampArgs` serializes those fields into command-line flags. `readProcessStamp` parses them back from process args. `matchesStampedProcess` lets orchestration code find the right process without brittle regexes.

## Sidecar Protocol

`packages/sidecar-proto` also owns IPC message schemas:

- daemon sidecar: `status`, `shutdown`
- web sidecar: `status`, `shutdown`
- desktop sidecar: `status`, `shutdown`, `eval`, `screenshot`, `console`, `click`

The desktop inspection surface is why `tools-dev inspect desktop ...` can evaluate JS, capture screenshots, read console entries, and click selectors without guessing a browser port.

## Generic Sidecar Runtime

`packages/sidecar/src/index.ts` owns generic helpers:

- Resolve namespace from explicit option, env, or default.
- Resolve runtime roots under `.tmp/<source>/<namespace>/...`.
- Resolve IPC path per app and namespace.
- Allocate forced or dynamic ports.
- Create sidecar launch env.
- Bootstrap sidecar runtime and validate stamp/env consistency.
- Create newline-delimited JSON IPC server and client.
- Write JSON runtime files atomically.
- Clean stale Unix sockets.

This package intentionally does not know Open Design app constants. It consumes the protocol descriptor from `sidecar-proto`.

## Platform Process Primitives

`packages/platform/src/index.ts` owns:

- command invocation normalization, including Windows `.cmd`/`.bat`
- package-manager invocation
- logged and background spawn helpers
- process liveness checks
- process snapshot listing on Windows and POSIX
- process tree collection
- graceful and forced stop
- stamped process matching
- HTTP readiness wait
- log tail

This split keeps process handling reusable and keeps `tools-dev` from hand-writing process-scan logic.

## `tools-dev` Flow

`tools/dev/src/config.ts` resolves:

- workspace root
- namespace
- `.tmp/tools-dev/<namespace>` runtime root
- per-app IPC paths
- per-app log paths
- daemon sidecar entry
- web sidecar entry
- Electron binary
- web runtime Next dist and temporary tsconfig paths

`tools/dev/src/index.ts` exposes:

- `pnpm tools-dev start [app]`
- `pnpm tools-dev run [app]`
- `pnpm tools-dev status [app]`
- `pnpm tools-dev stop [app]`
- `pnpm tools-dev restart [app]`
- `pnpm tools-dev logs [app]`
- `pnpm tools-dev inspect <app> [target]`
- `pnpm tools-dev check [app]`

Dependency ordering is encoded in `resolveStartApps`:

- Starting `daemon` starts only daemon.
- Starting `web` starts daemon then web.
- Starting `desktop` starts daemon, web, then desktop.
- Default start starts all three.

Stop order is reversed: desktop, web, daemon.

## Daemon Sidecar

`apps/daemon/sidecar/server.ts` wraps `apps/daemon/src/server.ts`.

It:

- Parses `OD_PORT`, allowing `0` for dynamic port.
- Starts the Express server with `returnServer: true`.
- Creates a status snapshot with pid, state, timestamp, and URL.
- Opens a JSON IPC server at the stamped IPC path.
- Responds to `status` and `shutdown`.
- Monitors `OD_TOOLS_DEV_PARENT_PID` and exits when the parent tool dies.

The daemon itself still owns product API behavior; the sidecar wrapper only owns lifecycle status and shutdown.

## Web Sidecar

`apps/web/sidecar/server.ts` wraps Next.js.

It:

- Resolves the `@open-design/web` package root.
- Creates a Next server.
- Proxies `/api/*`, `/artifacts/*`, and `/frames/*` to the daemon origin.
- Listens on `OD_WEB_PORT`, allowing `0` for dynamic port.
- Creates status IPC.
- Preserves `next-env.d.ts` during `app.prepare()` to avoid source churn.

This is distinct from `apps/web/next.config.ts`, which handles dev rewrites when running Next directly. The sidecar path owns server-mode runtime for dev sidecars and packaged runtime.

## Desktop Sidecar

`apps/desktop/src/main/index.ts` is the desktop entry.

It:

- Reads the process stamp.
- Bootstraps sidecar runtime.
- Discovers web URL by asking the web sidecar over IPC.
- Creates the Electron runtime in `runtime.ts`.
- Exposes status/eval/screenshot/console/click/shutdown over IPC.
- Monitors parent process in tools-dev mode.

Desktop does not guess ports. It asks the web sidecar for the current URL. This is a central invariant.

## Packaged Runtime

`apps/packaged/src/index.ts` is the packaged Electron root.

It:

1. Reads packaged config.
2. Resolves namespace.
3. Resolves namespace-scoped paths.
4. Ensures runtime, data, cache, log, desktop log, and Electron user data dirs.
5. Writes packaged desktop identity.
6. Applies Electron path overrides.
7. Applies sidecar launch env.
8. Starts packaged daemon and web sidecars.
9. Registers the `od://` protocol.
10. Imports `@open-design/desktop/main` and runs desktop main.
11. On shutdown, closes sidecars and identity.

`apps/packaged/src/sidecars.ts` starts daemon first, waits for daemon status, extracts the daemon port, starts web with that daemon port, waits for web status, and returns a handle that can close children in reverse order.

Packaged path invariants:

- Data/log/cache/runtime paths are namespace-scoped.
- Paths do not depend on daemon or web ports.
- Ports are transient transport details.
- Packaged daemon gets `OD_DATA_DIR` and `OD_RESOURCE_ROOT`.
- Packaged web runs with `OD_WEB_OUTPUT_MODE=server`.

## Runtime Roots

Default dev runtime:

```text
<project-root>/.tmp/tools-dev/<namespace>/
  current.json
  runs/
  logs/
  daemon/
  web/
  desktop/
```

Packaged runtime uses its own namespace paths resolved by `apps/packaged/src/paths.ts`. The exact OS location depends on packaged config, but the same namespace principle applies.

POSIX IPC sockets default to:

```text
/tmp/open-design/ipc/<namespace>/<app>.sock
```

Windows IPC uses named pipes:

```text
\\.\pipe\open-design-<namespace>-<app>
```

## Where To Change Things

| Change | Best place |
|---|---|
| New sidecar app or stamp field | Start in `packages/sidecar-proto`, but be very cautious. Stamp fields are constrained to five. |
| Runtime path policy | `packages/sidecar` for generic resolution; `apps/packaged/src/paths.ts` for packaged specifics. |
| Process discovery/cleanup | `packages/platform`, not `tools-dev` ad hoc logic. |
| Local lifecycle command | `tools/dev/src/index.ts` and `tools/dev/src/config.ts`. |
| Packaged startup/shutdown | `apps/packaged/src/index.ts` and `sidecars.ts`. |
| Desktop inspection | `apps/desktop/src/main/index.ts` and `runtime.ts`. |

