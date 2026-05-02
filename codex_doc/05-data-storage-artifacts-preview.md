# Data, Storage, Artifacts, Preview, Export, And Deploy

Open Design is local-first because generated work becomes files on disk and product state is stored in a local SQLite database. This document explains the data plane.

Primary source files:

- `apps/daemon/src/db.ts`
- `apps/daemon/src/projects.ts`
- `apps/daemon/src/artifact-manifest.ts`
- `apps/web/src/artifacts/manifest.ts`
- `apps/web/src/artifacts/renderer-registry.ts`
- `apps/web/src/components/FileViewer.tsx`
- `apps/web/src/components/FileWorkspace.tsx`
- `apps/daemon/src/document-preview.ts`
- `apps/daemon/src/deploy.ts`

## Data Roots

Default daemon data root:

```text
.od/
  app.sqlite
  projects/
  artifacts/
```

`OD_DATA_DIR` can relocate daemon data relative to the repo root. Packaged runtime sets managed data paths explicitly so packaged data does not rely on inferred ports or bundle layout.

## SQLite Tables

`apps/daemon/src/db.ts` creates these tables:

| Table | Purpose |
|---|---|
| `projects` | Project cards, selected skill/design system, pending prompt, metadata, created/updated timestamps. |
| `templates` | User-saved templates built from project files. |
| `conversations` | One or more chat threads per project. |
| `messages` | Chat content, agent events, attachments, run id/status, produced files. |
| `tabs` | Persisted open tabs and active file per project. |
| `deployments` | Vercel self-deploy records, status, URLs, counts, timestamps. |

The run service is not SQLite-backed. It is in-memory and has a TTL because runs are process-local live streams. Persistent chat history stores the important run metadata and events.

## Project Files

Generated and uploaded files live under:

```text
.od/projects/<projectId>/
```

The daemon file registry in `apps/daemon/src/projects.ts` owns:

- Path validation.
- Directory creation.
- Recursive file collection.
- MIME inference.
- coarse file kind inference.
- Read/write/delete.
- Artifact manifest sidecar files.
- Unicode-aware filename sanitization.
- Multipart filename UTF-8 recovery.

The file listing hides:

- dotfiles
- `.artifact.json` manifest files

That keeps the user's file workspace focused on actual deliverables.

## File Kinds

`kindFor` maps file extensions to coarse UI buckets:

- `html`
- `sketch`
- `image`
- `video`
- `audio`
- `text`
- `code`
- `pdf`
- `document`
- `presentation`
- `spreadsheet`
- `binary`

These kinds decide which `FileViewer` branch renders the file.

## Artifact Manifests

Artifact manifests are optional JSON sidecars named:

```text
<file>.artifact.json
```

Daemon validation lives in `apps/daemon/src/artifact-manifest.ts`; frontend inference lives in `apps/web/src/artifacts/manifest.ts`.

Allowed manifest concepts include:

- `kind`: html, deck, react-component, markdown-document, svg, diagram, code-snippet, mini-app, design-system
- `renderer`: html, deck-html, react-component, markdown, svg, diagram, code, mini-app, design-system
- `exports`: html, pdf, zip, pptx, jsx, md, svg, txt
- `status`: streaming, complete, error
- title, source skill id, design system id, metadata, supporting files

If no manifest exists, legacy inference kicks in:

- HTML files are `html` or `deck` if their names include deck/slides/pitch.
- Markdown files are markdown documents.
- SVG files are SVG artifacts.

## Renderer Registry

`apps/web/src/artifacts/renderer-registry.ts` resolves file plus manifest into a renderer:

- `deck-html`
- `html`
- `markdown`
- `svg`

Then `FileViewer.tsx` handles the rest:

- Native image/video/audio preview.
- Sketch image preview.
- Text/code viewer.
- Document preview for PDF/docx/pptx/xlsx.
- Binary fallback.

## HTML And Deck Preview

HTML and deck files render in iframes. Deck HTML uses separate deck affordances and export options. The web runtime uses `buildSrcdoc` and raw project-file endpoints so generated HTML can run in a sandboxed preview rather than being merged into the host React app.

The deck framework directive in the prompt composer exists because deck HTML is not just a static page. It needs predictable navigation, counter, scroll behavior, and print layout so preview and export can work.

## Markdown Preview

Markdown uses `renderMarkdownToSafeHtml`. Markdown artifacts can stream partial content more usefully than HTML/deck renderers, so the renderer registry marks markdown as `supportsStreaming: true`.

## Documents

For PDFs and Office-like files, the daemon builds a lightweight preview summary in `apps/daemon/src/document-preview.ts`. The UI intentionally does not try to implement a full document renderer. It gives title and sections, plus open/download actions.

## Uploads And Attachments

The chat composer uploads pasted/dropped files through project-scoped endpoints. The daemon stores them flat in the project folder and returns the same metadata shape as `listFiles`, so the browser can stage attachments without an extra refresh.

When the user sends a message, attachments become project-relative paths in `ChatRequest.attachments`. The daemon validates each path again before including it as an attachment hint in the prompt.

## Exports

The UI includes browser-side export helpers in `apps/web/src/runtime/exports.ts` and file-specific actions in `FileViewer.tsx`:

- HTML
- PDF
- ZIP
- PPTX request path for deck-like files

PPTX export is agent-mediated in this snapshot: `ProjectView.handleExportAsPptx` sends a prompt like "Export @file.html as an editable PPTX..." with the file attached, letting the selected agent produce the editable deck.

## Deploy

`apps/daemon/src/deploy.ts` owns the Vercel self-deploy path:

- Reads and writes deploy config.
- Builds the deploy file set.
- Calls Vercel deployment APIs.
- Stores deployment records.
- Checks deployed links.

Routes include:

- `GET /api/deploy/config`
- `PUT /api/deploy/config`
- `GET /api/projects/:id/deployments`
- `POST /api/projects/:id/deploy`
- `POST /api/projects/:id/deployments/:deploymentId/check-link`

## Import

`apps/daemon/src/claude-design-import.ts` supports importing Claude Design zip files through:

```text
POST /api/import/claude-design
```

The import route creates a project, saves extracted files, and points the UI at the entry file.

## Failure And Safety Notes

- File write paths are sanitized before resolution.
- Deleting a project removes its project folder.
- Manifests are validated and ignored if malformed.
- Binary previews are intentionally conservative.
- Document preview is best-effort and should not be treated as content-preserving conversion.
- Deploy config masks tokens on read.
- Runtime `.od/` and `.tmp/` directories should not be committed.

