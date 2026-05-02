# What is Open Design?

**Open Design (OD) is a web app that turns natural-language briefs into editable, previewable design artifacts — prototypes, decks, dashboards, marketing pages — by orchestrating the coding agent already installed on your machine.**

---

## The problem it solves

In April 2026, Anthropic released [Claude Design](https://www.anthropic.com/news/claude-design-anthropic-labs): a product that takes a brief and produces realistic prototypes, pitch decks, and design collateral using Claude Opus 4.7. It went viral because it genuinely worked — the output quality was dramatically above typical AI-generated UI, and the feedback loop (brief → questions → direction → artifact → refine) felt like working with a senior designer.

Claude Design is closed-source, cloud-only, locked to Anthropic's model, and available only on paid Claude plans. There is no self-host, no BYOK, no swap-in-your-own-agent, no way to version-control your skills or design systems.

Open Design is the open-source answer to that. Same user experience, same artifact quality target, none of the lock-in.

---

## How it works — the mental model

OD has three moving parts:

**1. The web app** is the interface. It runs in a browser (locally via `pnpm tools-dev`, or deployed to Vercel). It shows the chat panel, the live streaming agent feed, the sandboxed artifact preview, and the file workspace. It is a Next.js 16 App Router app — stateless, Vercel-deployable, no secrets.

**2. The local daemon** is the brain. It is a long-running Node 24 + Express process that owns everything privileged: agent detection, skill registry, design-system resolver, SQLite project database, file I/O, artifact lint, and export pipeline. It runs on your machine; secrets never leave it.

**3. Your coding-agent CLI** is the engine. OD does not ship a model. On startup, the daemon scans your `PATH` for any of 11 supported CLIs — Claude Code, Codex, Gemini CLI, Cursor Agent, OpenCode, Qwen Code, GitHub Copilot CLI, Hermes, Kimi, Pi, and Kiro. Whichever ones it finds become candidate design engines, switchable from the model picker in the UI. No CLI? An OpenAI-compatible BYOK proxy is available as a fallback.

The daemon spawns your CLI with its working directory set to a real project folder under `.od/projects/<id>/`. The agent gets real `Read`, `Write`, `Bash`, and `WebFetch` access against a real filesystem. It reads the skill instructions, reads the design-system tokens, writes files to disk, and the daemon streams its output back to the web app in real time.

```
Browser (Next.js 16)
  ↓  /api/chat  →  SSE
Local daemon (Node 24 + Express + SQLite)
  ↓  spawn(cli, args, { cwd: .od/projects/<id>/ })
Your agent CLI (claude / codex / gemini / opencode / ...)
  →  reads  SKILL.md + DESIGN.md
  →  writes  index.html, assets/, brand-spec.md, ...
```

---

## The four modes

Every design task in OD maps to one of four modes. Each mode uses a different **skill type** and produces a different artifact shape.

| Mode | What you get | Typical time |
|------|-------------|-------------|
| **Prototype** | Single editable HTML/JSX screen (landing, dashboard, app flow) | 60–120 s |
| **Deck** | Multi-slide HTML presentation (pitch deck, weekly update, product demo) | 90–180 s |
| **Template** | Populated copy of a curated design — agent fills content, doesn't design | 20–40 s |
| **Design System** | A `DESIGN.md` file + sample component preview — the meta-mode that feeds all others | 60–180 s |

Modes compose. Run Design System mode once; every subsequent Prototype / Deck / Template automatically picks up the tokens from `DESIGN.md`.

---

## Six load-bearing ideas

### 1. We don't ship an agent. Yours is good enough.

The daemon PATH-scans for 11 supported CLIs on startup. Each becomes a design engine, driven over stdio. Swap with one click in the UI. No CLI? The BYOK proxy (`POST /api/proxy/stream`) accepts any OpenAI-compatible endpoint — Anthropic, DeepSeek, Groq, OpenRouter, or your self-hosted vLLM.

### 2. Skills are files, not plugins.

Each skill is a folder: `SKILL.md` + `assets/` + `references/`. Drop one into `skills/`, restart the daemon, it appears in the picker. Skills follow the [Claude Code SKILL.md convention](https://docs.anthropic.com/en/docs/claude-code/skills) — any skill written for Claude Code runs in OD without modification.

### 3. Design systems are portable Markdown.

The 9-section `DESIGN.md` format (color, typography, spacing, layout, components, motion, voice, brand, anti-patterns) is a plain text file you can review in a PR, fork, version, and share. 72 systems ship out of the box — Linear, Stripe, Vercel, Apple, Notion, Tesla, Anthropic, and more.

### 4. The discovery form prevents 80% of redirects.

Every new project brief triggers a structured question form before the agent writes a single pixel. Surface, audience, tone, brand context, scale — locked down in 30 seconds of radio clicks. The cost of a wrong direction is one chat round, not one finished deck.

### 5. The daemon makes the agent feel like it's on your laptop, because it is.

The agent's working directory is a real folder under `.od/projects/<id>/`. It can `Read` the skill's seed template, `grep` your CSS for brand hex values, write a `brand-spec.md`, drop generated images, and produce files that appear as download chips in the UI. Nothing is synthesized in memory; everything is on disk.

### 6. The prompt stack is the product.

What reaches the agent on every turn is not a single system prompt — it is six composable layers: discovery directives + identity charter + active DESIGN.md + active SKILL.md + project metadata + (for decks) a fixed deck framework. Each layer is a file you can read and edit. See [The prompt stack](the-prompt-stack.md) for the full walkthrough.

---

## What Open Design is not

- **Not a Figma replacement.** Output is code (HTML/JSX, self-contained) and design-system files (DESIGN.md, Markdown, PPTX). Not editable vector canvases.
- **Not a model router.** OD doesn't layer its own provider abstraction on top of your agent's. If your agent supports 20 providers, great. If it only supports Anthropic, that's the ceiling.
- **Not a hosted service.** OD has no servers, no user accounts, no billing. You run it; your secrets stay with you.
- **Not a desktop-only app.** The web layer is Vercel-deployable. The daemon runs locally and can be exposed over a tunnel for remote access.

---

## Next steps

- [How it clones Claude Design](cloning-claude-design.md) — loop-by-loop mapping of what OD copies, what it diverges on, and why
- [The prompt stack](the-prompt-stack.md) — how the system prompt is composed from six layers
- [Key concepts](key-concepts.md) — glossary of all terms
- [`docs/architecture.md`](../architecture.md) — full component diagram and data flows
