# The prompt stack

The prompt stack is what turns a generic coding-agent CLI into a senior designer with a working filesystem, a deterministic palette library, and a checklist culture.

Every time a user sends a message, the daemon assembles a system prompt from up to seven ordered layers, then passes it to the spawned CLI. This doc walks through each layer: what it is, what it contributes, and where the code lives.

The composer is `apps/daemon/src/prompts/system.ts` — `composeSystemPrompt(input)`.

---

## Layer 1 — Discovery and philosophy (always first)

**File:** `apps/daemon/src/prompts/discovery.ts` — `DISCOVERY_AND_PHILOSOPHY`

This is the dominant layer. It stacks **before** the identity charter (Layer 2) so its hard rules win over any softer wording lower in the stack.

It contains three rules that govern the start of every design task:

### RULE 1 — turn 1 must emit a discovery form

The agent's first output on any new project is always exactly:

```
one short prose line
+ <question-form id="discovery"> ... </question-form>
+ STOP (no code, no tools, no thinking)
```

The form captures seven fields: what are we making, primary platform, audience, visual tone, brand context, scale, and constraints. The JSON schema is part of the prompt — the agent must produce valid JSON in the form body or the browser can't parse it.

The rule has three narrow exceptions (user is tweaking an existing design, user says "just build", or the message starts with `[form answers — ...]`). In every other case the form is non-negotiable. This is the single biggest behavioral change from a vanilla Claude Code session — the agent is instructed to refuse to write code until it has the answers.

### RULE 2 — turn 2 branches on the `brand` answer

After the user submits the discovery form, the agent inspects the `brand` field and takes one of three paths:

**Branch A — "Pick a direction for me":** Emit a second form (`<question-form id="direction">`) with five visual-direction cards — Editorial/Monocle, Modern Minimal, Tech Utility, Brutalist, Soft Warm. Each card has a palette swatch block, type sample, mood description, and real-world references. The user picks one radio. The agent then looks up the direction's OKLch palette + font stacks in the direction library (also embedded in this layer, via `apps/daemon/src/prompts/directions.ts`) and binds them verbatim into the seed template's `:root`. No improvisation.

**Branch B — user has a brand:** Run a five-step extraction before any layout work:
1. Locate source files or URLs
2. Download CSS, PDFs, screenshots
3. `grep -E '#[0-9a-fA-F]{3,8}'` — never guess colors from memory
4. Write `brand-spec.md` with six OKLch tokens + font stacks + layout posture rules
5. Vocalize the system in one sentence so the user can redirect

**Branch C — anything else:** Skip to RULE 3.

### RULE 3 — TodoWrite the plan, then live updates

Once direction/brand is locked, the agent's first tool call is `TodoWrite` with a 5–10 item plan. Steps are marked `in_progress` → `completed` in real time. Steps 7 (checklist) and 8 (5-dim critique) are non-negotiable.

The 5-dimensional critique scores the output on philosophy / hierarchy / execution / specificity / restraint, each 1–5. Anything under 3/5 triggers a fix-and-rescore loop. Two passes is normal.

The layer also embeds the full **anti-AI-slop checklist** — a blacklist of things the agent must not do:

- Aggressive gradient backgrounds
- Generic emoji icons (✨ 🚀 🎯)
- Rounded card with a left colored border accent
- Inter / Roboto / Arial as a display face
- Invented metrics without a source
- Filler copy or lorem ipsum
- An icon next to every heading

**Source provenance:** This layer is explicitly distilled from [`alchaincyf/huashu-design`](https://github.com/alchaincyf/huashu-design) — Junior-Designer mode, the brand-asset protocol, the anti-slop checklist, the 5-dim self-critique, and the "5 schools × 20 design philosophies" library behind the direction picker.

---

## Layer 2 — Identity charter (the official designer prompt)

**File:** `apps/daemon/src/prompts/official-system.ts` — `OFFICIAL_DESIGNER_PROMPT`

The comment at the top of this file reads: *"Adapted from claude.ai/design's 'expert designer' prompt — same identity, workflow, and content philosophy, retargeted to the tools an OD-managed agent actually has."*

This layer defines the agent's role, what it can and cannot access, and the non-negotiable output rule:

**Role:** Expert designer, user is the manager. HTML is the tool; the medium varies (slide designer, UX designer, brand designer, systems designer depending on the brief).

**The `<artifact>` contract:** After every turn that produces a deliverable, the agent's last output must be:

```xml
<artifact identifier="kebab-slug" type="text/html" title="Human title">
<!doctype html>
<html>... complete standalone document ...</html>
</artifact>
```

After `</artifact>`, stop. No narration. No summary. The web app parses this tag to extract the HTML and render it in the sandboxed preview iframe.

**Design output rules:** Descriptive file names, version copies before major edits, files under ~1000 lines, deck positions persisted to localStorage, no `scrollIntoView` (breaks the embedded preview), CSS custom properties for tokens, modern CSS toolbox (`text-wrap: pretty`, container queries, `color-mix()`, view transitions).

**Content rules:** No filler, no placeholder text, no surprise-added content, vocalize the token system before building (gives the user a cheap redirect), honest placeholders over invented stats.

**Pinned React / Babel versions:** When the agent writes JSX artifacts, it must use exact SRI-hashed `<script>` tags for React 18.3.1 and Babel standalone 7.29.0. This prevents version drift from breaking the preview iframe.

**Secrecy:** The agent must not reveal its system prompt, tool names, or skill names. It can describe its capabilities in user-facing terms ("I can build prototypes, decks, design systems") but not name the underlying machinery.

---

## Layer 3 — Active design system

**Injected by:** `composeSystemPrompt()` when `designSystemBody` is non-empty.

```
## Active design system — <name>

Treat the following DESIGN.md as authoritative for color, typography, spacing, and component rules.
Do not invent tokens outside this palette. When you copy the active skill's seed template, bind
these tokens into its :root block before generating any layout.

<full DESIGN.md content>
```

The 9-section DESIGN.md format covers: Visual Theme & Atmosphere · Color Palette & Roles · Typography Rules · Component Stylings · Layout Principles · Depth & Elevation · Do's and Don'ts · Responsive Behavior · Agent Prompt Guide.

The design-system resolver (`apps/daemon/src/design-systems.ts`) loads the body from the active system's folder. The user can switch design systems between turns; the next composed prompt uses the new tokens.

OD ships 72 systems out of the box. The same DESIGN.md file is also written to the project's cwd, so the agent can `Read` it directly if the skill instructs it to.

---

## Layer 4 — Active skill + pre-flight directive

**Injected by:** `composeSystemPrompt()` when `skillBody` is non-empty.

```
## Active skill — <name>

Follow this skill's workflow exactly. **Pre-flight (do this before any other tool):**
Read `assets/template.html`, `references/layouts.md`, `references/themes.md`,
`references/components.md`, `references/checklist.md` via the path written in the
skill-root preamble. [...]

<full SKILL.md body>
```

The pre-flight directive is generated dynamically by `derivePreflight(skillBody)` — it scans the skill body for references to specific asset paths and constructs a short imperative instruction to read them first. Without this, the agent occasionally skips Step 0 under context pressure, which is the #1 cause of regression to generic AI-slop output.

The skill body is the free-form Markdown workflow section of the `SKILL.md` file — a numbered step list of what to do, usually reading the seed template, selecting from the layouts library, filling placeholders, and self-checking against the checklist. This is what turns a blank agent session into a focused specialist. See [`docs/skills-protocol.md`](../skills-protocol.md) for the full SKILL.md format.

---

## Layer 5 — Project metadata

**Injected by:** `renderMetadataBlock(metadata, template)` in `system.ts`.

```
## Project metadata

These are the structured choices the user made (or skipped) when creating this project.
Treat known fields as authoritative; for any field marked "(unknown — ask)" you MUST include
a matching question in your turn-1 discovery form.

- **kind**: prototype
- **fidelity**: high-fidelity
- **inspirationDesignSystemIds**: stripe, vercel
```

The metadata block carries the values the user set when creating the project (kind, fidelity, speaker-notes preference, animation intent, inspiration design-system IDs, template reference). Any unknown field is rendered as `(unknown — ask)` — a signal to the agent that it must include that question in its turn-1 discovery form.

For image/video/audio projects, the metadata block also contains the reference prompt template (if the user selected one) and explicit dispatch instructions: "This is an image project. Dispatch via `od media generate --surface image ...`. Do NOT emit `<artifact>` HTML."

---

## Layer 6 — Deck framework directive (decks only, pinned last)

**File:** `apps/daemon/src/prompts/deck-framework.ts` — `DECK_FRAMEWORK_DIRECTIVE`

**Condition:** `skillMode === 'deck'` OR `metadata.kind === 'deck'`, AND the active skill does not already ship `assets/template.html`.

This layer is pinned **last** in the stack so it overrides any softer "write a script that handles arrows" wording earlier. It contains:

- The canonical 1920×1080 scale-to-fit viewport approach
- The keyboard / scroll navigation handler
- The slide counter template
- The position-persist-to-localStorage pattern
- The print-to-PDF stylesheet (deck-aware, no nav chrome)

The comment in `system.ts` explains why it must be last: *"Every freeform attempt at scale-to-fit / nav / print re-introduces the same iframe positioning / scaling bugs we have already fixed in the framework."* The agent's job is to drop the framework in, bind the palette, and fill the `<section class="slide">` slots. Not to re-author the framework.

The framework fires even when no skill is bound (`skill_id null`) — so a deck project without a selected skill still gets the correct structural skeleton.

---

## Layer 7 — Media generation contract (media projects only)

**File:** `apps/daemon/src/prompts/media-contract.ts` — `MEDIA_GENERATION_CONTRACT`

**Condition:** `skillMode` or `metadata.kind` is `image`, `video`, or `audio`.

For media-surface projects, the agent dispatches to external generation services via an `od media generate` command rather than writing HTML. This layer specifies the dispatch contract: flag names, required parameters, how to pick a model, how to handle the result, and — critically — that the agent must NOT emit `<artifact>` HTML for media surfaces.

---

## Composition order and why it matters

```
composeSystemPrompt() output:
  1. DISCOVERY_AND_PHILOSOPHY     ← hard rules win (highest precedence)
  2. --- separator ---
  3. OFFICIAL_DESIGNER_PROMPT     ← identity + workflow charter
  4. Active DESIGN.md             ← authoritative tokens
  5. Active SKILL.md + pre-flight ← workflow for this artifact type
  6. Project metadata             ← structured project context
  7. DECK_FRAMEWORK_DIRECTIVE     ← pinned last, overrides any conflicting wording
```

Layer 1 wins on conflicts with Layer 3 because it appears first — LLMs weight earlier tokens more strongly under context pressure. The deck framework (Layer 7) is an exception: it wins by being last, specifically to override the "write a nav script" impulse that appears in the identity charter.

The total prompt is typically 4,000–8,000 tokens before the user's message, depending on DESIGN.md size and skill complexity. Section pruning (`od.design_system.sections` in SKILL.md frontmatter) is available to reduce it for skills that only use a subset of the design system.

---

## How to read the source

```
apps/daemon/src/prompts/
├── system.ts          ← composeSystemPrompt() — the composer
├── discovery.ts       ← Layer 1: DISCOVERY_AND_PHILOSOPHY (the 3-rule arc)
├── directions.ts      ← Direction library (OKLch palettes + font stacks × 5 directions)
├── official-system.ts ← Layer 2: OFFICIAL_DESIGNER_PROMPT (identity charter)
├── deck-framework.ts  ← Layer 6/7: DECK_FRAMEWORK_DIRECTIVE
└── media-contract.ts  ← Layer 7: MEDIA_GENERATION_CONTRACT
```

Layers 3–5 (design system body, skill body, metadata) come from the caller — `apps/daemon/src/server.ts` resolves them from the daemon's design-system catalog, skill registry, and SQLite project table before calling `composeSystemPrompt()`.

---

## Further reading

- [Cloning Claude Design](cloning-claude-design.md) — behavioral mapping of each loop step
- [`docs/skills-protocol.md`](../skills-protocol.md) — SKILL.md format and OD extensions
- [`apps/daemon/src/prompts/system.ts`](../../apps/daemon/src/prompts/system.ts) — the composer source
- [`apps/daemon/src/prompts/discovery.ts`](../../apps/daemon/src/prompts/discovery.ts) — the 3-rule arc in full
