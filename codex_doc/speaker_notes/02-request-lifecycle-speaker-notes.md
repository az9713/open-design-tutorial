# Speaker Notes: Request Lifecycle

Use with `codex_doc/infographics/02-request-lifecycle.svg`.

## Opening

"This slide answers the practical question: what exactly happens after the user presses Send? We will move left to right and top to bottom from browser UI, through daemon, into a local CLI, then back into persisted chat and preview."

## 1. ProjectView.handleSend

Point to box 1.

"The browser starts in `ProjectView.handleSend`. It creates a user message, creates an assistant placeholder, persists both through the daemon, and prepares abort controllers and stream handlers."

"This is also where the UI decides whether the request goes through local CLI mode or direct API fallback."

## 2. streamViaDaemon

Point to box 2.

"In local CLI mode, `streamViaDaemon` posts a `ChatRequest` to `/api/runs`. The daemon responds immediately with a run id, then the browser opens `/api/runs/:id/events` to read the SSE stream."

"This split is important because the run can outlive a single fetch connection."

## 3. SSE Consumer

Point to box 3.

"The browser tracks the last event id. If the connection drops, it reconnects with `?after=<id>`. If it still cannot finish the stream, it asks `/api/runs/:id` for terminal status."

"That is how the UI can reattach to active runs instead of losing the turn."

## 4. createChatRunService

Point to box 4.

"On the daemon side, `createChatRunService` creates an in-memory run. It stores an event buffer, a client set, status, child process handle, and cancellation state. It fans each event out to connected SSE clients."

## 5. Compose Instructions

Point to box 5.

"Before spawning the agent, the daemon composes the instruction packet. It pulls in the active skill, the active design system, project metadata, template snapshots, deck or media directives when relevant, a cwd hint, existing file list, and attachment hints."

"This composed prompt is the core product intelligence."

## 6. Resolve Agent

Point to box 6.

"Next the daemon resolves the selected agent definition in `agents.ts`. It validates the model and reasoning choices against known options or a custom-model sanitizer."

"It also decides how the prompt will be delivered: stdin when possible, a temporary prompt file for long Windows prompts when needed, or argv only when safe."

## 7. Spawn Child Process

Point to box 7.

"The daemon spawns the local CLI with cwd set to `.od/projects/<projectId>`. That cwd is the output boundary. The daemon also injects `OD_BIN`, `OD_DAEMON_URL`, `OD_PROJECT_ID`, and `OD_PROJECT_DIR`, which lets the agent call back into Open Design for media generation."

## 8. Local CLI Runs

Point to box 8.

"At this point the selected coding agent is doing its normal work: reading files, planning, writing files, possibly running commands, and emitting stdout. Open Design is not controlling the internal model loop."

## 9. Writes Files

Point to box 9.

"The agent writes HTML, Markdown, SVG, media, Office-like files, or manifests directly into the project folder. Because the files are real, the UI can list them immediately and the user can inspect them outside the app."

## 10. Daemon Parses Stream

Point to box 10.

"The daemon parses stdout depending on the agent. Claude and Copilot have structured stream handlers. Codex, Gemini, Cursor, and OpenCode-like streams go through `json-event-stream.ts`. Plain CLIs fall back to stdout chunks."

"Everything gets normalized into start, agent, stdout, stderr, error, and end SSE events."

## 11. Media Side Path

Point to box 11.

"For image, video, and audio projects, the agent should not emit HTML. It calls `od media generate`. That command posts to the running daemon, the daemon routes to the provider, writes the bytes into the same project folder, and the same file viewer renders the result."

## 12. Chat Updates

Point to box 12.

"The browser translates SSE events into chat text, status rows, tool cards, thinking, usage, and raw lines. The assistant message is updated and persisted with run id, event id, status, content, and events."

## 13. File Refresh

Point to box 13.

"When the UI sees a successful write tool result, it refreshes the project files and opens the generated file in a tab. This is why files can pop into view before the full agent turn is complete."

## 14. Renderer Registry

Point to box 14.

"The file viewer consults the artifact manifest or legacy inference. It then chooses HTML, deck HTML, Markdown, SVG, image, video, audio, text, code, document preview, or binary fallback."

## 15. User Iterates

Point to box 15.

"The next prompt has the same project folder as context. The user can continue, attach generated files, request a PPTX export, deploy, save a template, or open files directly."

## Closing

"The end-to-end trick is simple but powerful: chat creates a run, the run creates files, the files become the product. The local agent is the worker, the daemon is the broker, and the web app is the operating surface."

