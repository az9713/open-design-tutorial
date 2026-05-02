# How Open Design clones Claude Design

The word "clone" here is architectural, not literal. Open Design replicates the *user experience loop* that Claude Design introduced: brief → discovery → direction → artifact → preview → refine → export. The machinery is completely different.

This doc maps each step of that loop — what Claude Design does (from its public description), what OD substitutes, and where the two deliberately diverge.

> **Caveat on Claude Design internals.** Claude Design is closed-source. Everything here about its user-facing behavior comes from [Anthropic's published description](https://www.anthropic.com/news/claude-design-anthropic-labs). Its internal implementation is unknown. Comparisons are behavioral, not architectural.

---

## The loop, step by step

### Step 1 — Brief intake

**Claude Design:** User types a brief or uploads content (images, DOCX, PPTX, XLSX, website captures).

**Open Design:** Same inputs. The difference is what happens immediately after the user hits Send.

OD's first agent output is never code. It is a `<question-form id="discovery">` — a structured JSON form rendered in the web UI as radio buttons and text fields. The form locks in surface (what are we making?), platform (mobile / desktop / fixed canvas), audience, visual tone, brand context, and scale before the agent writes a pixel.

```
Turn 1 output (always):
  one prose line
  + <question-form id="discovery">
  + STOP — no code, no tools, no Bash
```

This rule is enforced in the system prompt as RULE 1 with no override except explicit "skip questions" from the user. The reason: a detailed brief still leaves visual tone, color stance, and scale undecided — exactly the things that cause a wrong direction. Locking them in 30 seconds of radio clicks is cheaper than fixing a finished deck.

**Divergence:** Claude Design has no equivalent discoverable question form in its public description. OD treats the discovery form as a hard architectural gate.

---

### Step 2 — Brand / direction setup

**Claude Design:** Builds a design system automatically from uploaded brand files and design assets during onboarding. The design system is then available for all subsequent generations.

**Open Design:** Turn 2 branches on the `brand` answer from the discovery form.

**Branch A — user has no brand:**
OD emits a second question form (`<question-form id="direction">`) with five curated visual directions, each rendered as a card with palette swatches, type sample, mood description, and real-world references:

| Direction | Palette | Personality | References |
|-----------|---------|-------------|-----------|
| Editorial — Monocle/FT | Ink · cream · warm rust | Print magazine | Monocle · FT Weekend · NYT Magazine |
| Modern minimal — Linear/Vercel | Cool · structured · minimal accent | Sparse, precise | Linear · Vercel · Stripe |
| Tech utility | Dense · monospace · terminal | Information density | Bloomberg · Bauhaus tools |
| Brutalist | Raw · oversized type · harsh accents | No shadows, confrontational | Bloomberg Businessweek · Achtung |
| Soft warm | Generous · peachy neutrals · low contrast | Approachable | Notion marketing · Apple Health |

The user picks one radio. The daemon looks up the direction spec (OKLch palette + font stacks + layout posture cues) in `apps/daemon/src/prompts/directions.ts` and binds it verbatim into the seed template's `:root` block. No model freestyle on color. No AI-improvised palette. Deterministic.

**Branch B — user has a brand:**
The agent runs a five-step extraction protocol before touching any layout file:
1. Locate the source (attached files or a known URL pattern like `<brand>.com/brand`)
2. Download styling artifacts (CSS files, brand PDF, screenshots)
3. Extract real values (`grep -E '#[0-9a-fA-F]{3,8}'` on the CSS, never guess colors from memory)
4. Write `brand-spec.md` to the project root with six OKLch color tokens, font stacks, and layout posture rules
5. Vocalize the system in one sentence so the user can redirect cheaply

**Divergence:** Claude Design's design-system builder analyzes team codebases and design files automatically. OD's is explicit and reviewable — brand-spec.md lands in the project folder as a plain file you can read, edit, and commit. Both approaches aim for the same result (consistent tokens across all generations); OD's is more auditable.

---

### Step 3 — Seed setup and agent dispatch

**Claude Design:** Generation happens inside Anthropic's platform, with proprietary tooling and access patterns.

**Open Design:** The daemon assembles the environment before spawning the agent.

1. A project folder is created at `.od/projects/<id>/` if it doesn't already exist.
2. The skill's seed files are copied into the folder: `assets/template.html` (the starting scaffold), `references/layouts.md` (paste-ready section/screen/slide skeletons), `references/checklist.md` (P0/P1/P2 gates).
3. The composed system prompt is built (see [The prompt stack](the-prompt-stack.md)).
4. The daemon spawns the CLI: `spawn(bin, args, { cwd: projectFolder, stdio: 'pipe' })`.

The agent receives:
- Its working directory set to the project folder
- The full composed system prompt as its system context
- The discovery form answers + user brief as the user message
- Real filesystem access (`Read`, `Write`, `Bash`, `WebFetch`) scoped to the project folder

The agent can `Read` the seed template, grep the CSS, write files, and reference anything in the project directory. It operates on an actual disk, not a synthetic in-memory environment.

---

### Step 4 — Generation with a live plan

**Claude Design:** Generation is opaque from the outside; the user sees a loading state and then a result.

**Open Design:** The agent's first tool call after brand setup is `TodoWrite` — a structured plan of 5–10 imperative steps written to a live "Todos" card in the web UI. Steps are marked `in_progress` → `completed` in real time as the agent works through them.

Typical plan:
```
- 1.  Read DESIGN.md + skill assets (template.html, layouts.md, checklist.md)
- 2.  Bind chosen direction's palette to :root
- 3.  Plan section/slide/screen list (state aloud before writing)
- 4.  Copy seed template to project root
- 5.  Fill planned layouts/screens/slides
- 6.  Replace [REPLACE] placeholders with real copy
- 7.  Self-check: checklist.md P0 gates
- 8.  5-dim critique (philosophy/hierarchy/execution/specificity/restraint) — fix any < 3/5
- 9.  Emit single <artifact>
```

The user can redirect mid-flight by sending a new message. The daemon cancels the in-flight run and restarts with the correction.

Alongside the todo card, the web UI streams a live tool-call feed — every `Read`, `Write`, `Bash` the agent executes appears as a line item in the side panel. The user sees exactly what the agent is doing.

**Divergence:** The live plan and tool-call feed are OD-specific. Claude Design has no equivalent public-facing visibility into the generation process.

---

### Step 5 — Anti-slop quality gates

**Claude Design:** Output quality comes from Anthropic's training and internal tooling.

**Open Design:** Output quality is enforced through the prompt stack — two gates before the agent can emit a result.

**Gate 1 — P0/P1/P2 checklist.** Every skill ships a `references/checklist.md`. The agent must read it and pass every P0 item before emitting `<artifact>`. P0 failures block the emit.

**Gate 2 — 5-dimensional self-critique.** After the checklist, the agent scores its own output silently across five dimensions:

| Dimension | What it checks |
|-----------|---------------|
| Philosophy | Does the visual posture match what was asked (editorial / minimal / brutalist)? |
| Hierarchy | Is there one obvious focal point per screen, or are all elements competing? |
| Execution | Typography, spacing, alignment, contrast — right, or just close? |
| Specificity | Is every word and number specific to this brief, or did filler creep in? |
| Restraint | One accent, one decisive flourish — or three competing flourishes? |

Any dimension under 3/5 is a regression. The agent fixes the weakest and rescores. Two passes is normal before the artifact is emitted.

A slop blacklist is also enforced in the system prompt: aggressive gradient backgrounds, generic emoji icons, Inter/Roboto as a display face, invented metrics, lorem ipsum filler — all explicitly forbidden.

---

### Step 6 — Artifact and sandboxed preview

**Claude Design:** Rendered inline in the claude.ai interface.

**Open Design:** The agent's final output wraps the primary file in an `<artifact>` XML tag:

```xml
<artifact identifier="saas-landing" type="text/html" title="SaaS landing page">
<!doctype html>
<html>... complete standalone document ...</html>
</artifact>
```

The web app's parser (`apps/web/src/artifacts/parser.ts`) extracts the artifact content and renders it in a sandboxed iframe:

```html
<iframe sandbox="allow-scripts" srcdoc="...artifact HTML...">
```

`allow-same-origin` is deliberately absent — the artifact has no access to the host app's cookies, localStorage, or DOM. The preview hot-reloads on every file write from the agent (debounced 100 ms).

JSX artifacts use the same pattern but with vendored React 18 + Babel standalone injected into the `srcdoc` so JSX evaluates without a build step.

---

### Step 7 — Refinement

**Claude Design:** Inline comments, direct edits, and custom adjustment controls.

**Open Design:** Three refinement surfaces are available, depending on the active agent's capabilities.

**Chat:** Free-text follow-up ("move the CTA above the fold", "add a pricing section between hero and features"). The agent receives the message, reads the current files from disk, and edits in place.

**Comment mode (requires `surgicalEdit` capability):** The user clicks an element in the preview iframe → a popover appears → they type a note ("make this card glassmorphic") → the daemon sends a targeted edit instruction to the agent. Claude Code supports surgical edits natively via its `Edit` tool. Agents without this capability (e.g. Gemini CLI) get a whole-file regeneration with a "only change element X" constraint — functionally similar, but less precise.

**Parameter sliders:** When a skill declares `od.parameters` in its SKILL.md frontmatter (e.g. `accent_hue`, `section_spacing`), the UI renders live sliders after the first generation. Moving a slider re-prompts the agent with just the changed parameter — no full regeneration. The agent reads the current file, adjusts the specific CSS custom property, and writes back.

---

### Step 8 — Export

**Claude Design:** Canva, PDF, PPTX, HTML, or internal URLs.

**Open Design:** HTML (self-contained, all assets inlined) · PDF (browser print, deck-aware) · PPTX (agent-driven via skill, or page-capture fallback) · ZIP (full project archive) · Markdown.

The export pipeline lives in `apps/daemon/src/` and serves each format from a different code path. HTML inlines all CSS and rewrites asset URLs to `data:` URIs. PDF uses puppeteer's `page.pdf()` on the rendered artifact. PPTX uses `pptxgenjs` against a `slides.json` intermediate when the skill produces one (guizang-ppt does), or falls back to page-capture for skills that don't. ZIP runs `archiver` over the project folder.

---

### Step 9 — Claude Design ZIP import

OD adds one direction with no Claude Design equivalent: `POST /api/import/claude-design`.

Drop a Claude Design export ZIP onto the welcome dialog. The daemon extracts it into a real project folder, opens the entry file as a tab, and stages a "continue where Anthropic left off" prompt for your local agent. The project becomes an OD project — fully editable with any supported CLI.

This closes the gap between the two tools: work that started on Claude Design's paid platform can continue on OD locally.

---

## Comparison summary

| Aspect | Claude Design | Open Design |
|--------|--------------|-------------|
| License | Closed-source | Apache-2.0 |
| Form factor | Web (claude.ai) | Web app + local daemon |
| Vercel-deployable | No | Yes |
| Agent runtime | Bundled (Opus 4.7) | Delegated to your existing CLI (11 supported) |
| Model flexibility | Anthropic only | 11 CLIs + any OpenAI-compatible BYOK endpoint |
| Skills | Proprietary | File-based SKILL.md bundles — drop a folder, restart daemon |
| Design system | Proprietary, ephemeral | `DESIGN.md` text files — versionable, shareable, 72+ bundled |
| Discovery form | Not described publicly | Hard RULE 1 — cannot be skipped for new briefs |
| Direction picker | Not described publicly | 5 deterministic directions with OKLch palettes |
| Live todo progress | No | Yes — TodoWrite card, updated in real time |
| Streaming tool feed | No | Yes — every Read/Write/Bash appears live |
| Sandboxed iframe preview | Inline rendering | `srcdoc` iframe, `allow-scripts` only |
| 5-dim self-critique | Not described publicly | Hard pre-emit gate (fail → fix → rescore) |
| Artifact lint API | No | `POST /api/artifacts/lint` |
| Comment mode | Yes | Yes (when adapter has `surgicalEdit: true`) |
| Export formats | Canva · PDF · PPTX · HTML | HTML · PDF · PPTX · ZIP · Markdown |
| Claude Design ZIP import | n/a | `POST /api/import/claude-design` |
| Sidecar IPC + headless desktop | No | Yes — `tools-dev inspect desktop status/eval/screenshot` |
| Minimum billing | Claude Pro subscription | BYOK — paste any OpenAI-compatible `baseUrl` |
| Self-hostable | No | Yes — `pnpm tools-dev run web` |

---

## What OD deliberately does not copy

**The bundled agent model.** Claude Design's quality comes in part from Opus 4.7's vision capabilities and Anthropic's model training. OD delegates to whatever agent you already have. A user running Gemini CLI will get different output than one running Claude Code. OD trades peak quality ceiling for maximum flexibility — you keep your existing agent subscription and don't pay twice.

**The proprietary design-system builder.** Claude Design's design-system generation analyzes codebases and design files using Anthropic's internal tooling. OD's design-system mode produces a plain-text `DESIGN.md` via whatever agent you have — correct and portable, but not necessarily as comprehensive. The tradeoff: OD's output is auditable, versionable, and shareable as a text file.

**Real-time collaboration.** Claude Design supports organization-scoped sharing and collaborative review. OD is single-user in v1. Collaboration is explicitly post-v1 scope.

---

## Further reading

- [The prompt stack](the-prompt-stack.md) — how the system prompt is composed from six layers
- [`docs/skills-protocol.md`](../skills-protocol.md) — SKILL.md format, OD extensions, skill testing
- [`docs/agent-adapters.md`](../agent-adapters.md) — per-CLI adapter interface and capability matrix
- [`docs/modes.md`](../modes.md) — Prototype / Deck / Template / Design System modes in detail
