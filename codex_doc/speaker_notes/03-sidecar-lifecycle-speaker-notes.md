# Speaker Notes: Sidecar Lifecycle

Use with `codex_doc/infographics/03-sidecar-lifecycle.svg`.

## Opening

"This slide explains how Open Design starts and finds its own processes. The important point is that ports are unstable. Runtime identity comes from namespace roots, IPC paths, and process stamps."

## 1. sidecar-proto

Point to box 1.

"The protocol package owns Open Design-specific constants: app keys, modes, sources, env var names, IPC message schemas, and status shapes."

"It also defines the five stamp fields: app, mode, namespace, ipc, and source. Those five fields are the identity card for each managed process."

## 2. sidecar

Point to box 2.

"The generic sidecar package resolves namespace roots, runtime paths, IPC paths, launch env, and ports. It also implements the newline-delimited JSON IPC server and client."

"It does not hard-code Open Design app names. It consumes the contract descriptor from `sidecar-proto`."

## 3. platform

Point to box 3.

"The platform package handles process primitives: create stamp args, read stamps from command lines, spawn background or logged processes, list processes, collect process trees, stop processes, wait for HTTP, and read log tails."

"This keeps orchestration code from using brittle process-name regexes."

## 4. Resolve Config

Point to box 4.

"The dev path starts in `tools/dev/src/config.ts`. It resolves `.tmp/tools-dev/<namespace>`, IPC paths, log paths, sidecar entry paths, Electron binary, and web runtime paths."

## 5. Start Daemon

Point to box 5.

"`tools-dev` starts the daemon sidecar first. The daemon sidecar starts Express on the requested or dynamic `OD_PORT`, opens status/shutdown IPC, and reports the actual daemon URL."

## 6. Start Web

Point to box 6.

"The web sidecar starts next. It creates a Next server and proxies `/api`, `/artifacts`, and `/frames` to the daemon. It reports the web URL over IPC."

"This is why daemon and web can use dynamic ports while the rest of the system still knows where they are."

## 7. Start Desktop

Point to box 7.

"Desktop starts after web. It does not guess the web port. It asks the web sidecar for status and opens the reported URL. It exposes its own IPC inspection surface for status, eval, screenshot, console, and click."

## 8. Packaged Entry

Point to box 8.

"The packaged path starts in `apps/packaged/src/index.ts`. It reads config, resolves namespace paths, ensures data/log/cache/runtime directories, writes desktop identity, and applies Electron path overrides."

## 9. Spawn Daemon Sidecar

Point to box 9.

"Packaged runtime spawns daemon with managed paths: `OD_DATA_DIR` and `OD_RESOURCE_ROOT`. It sets `OD_PORT=0`, because the transport port can be dynamic. Then it waits for daemon status and captures the real URL."

## 10. Spawn Web Sidecar

Point to box 10.

"Packaged runtime then starts web with the daemon port extracted from status. Web runs in server output mode and reports its own URL."

## 11. Run Desktop Main

Point to box 11.

"Finally, packaged runtime registers the `od://` protocol and imports `@open-design/desktop/main`. On shutdown, desktop closes the sidecars and the identity handle."

## Load-Bearing Invariant

Point to the bottom banner.

"This is the invariant to remember: ports are transport details. Namespace plus IPC plus stamped process args identify the runtime. That is why desktop asks web for URL through IPC, and why packaged data/log/cache/runtime paths never include daemon or web ports."

## Closing

"When changing lifecycle code, start in the right layer. Protocol changes go to `sidecar-proto`; path and IPC primitives go to `sidecar`; process behavior goes to `platform`; dev orchestration goes to `tools-dev`; packaged orchestration goes to `apps/packaged`."

