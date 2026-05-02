# Open Design — explainer

Open Design is the open-source, local-first alternative to Anthropic's Claude Design: a web app that turns a natural-language brief into an editable, previewable design artifact by orchestrating the coding agent already on your machine.

---

## What's in this explainer

| Doc | What it answers |
|-----|----------------|
| [What is Open Design?](what-is-open-design.md) | The mental model, the problem it solves, what it is not |
| [How it clones Claude Design](cloning-claude-design.md) | Loop-by-loop mapping: what Claude Design does, what OD substitutes, where they diverge |
| [The prompt stack](the-prompt-stack.md) | How the daemon composes the system prompt that turns a generic CLI into a senior designer |
| [Key concepts](key-concepts.md) | Glossary of every term used across the codebase and docs |

> **New here?** Start with [What is Open Design?](what-is-open-design.md), then [How it clones Claude Design](cloning-claude-design.md).

## Related docs

The explainer layer sits on top of the existing technical docs. Once you have the mental model, these are the authoritative references:

| Doc | What it covers |
|-----|---------------|
| [`docs/spec.md`](../spec.md) | Product definition, scenarios, non-goals, positioning |
| [`docs/architecture.md`](../architecture.md) | System topology, component diagram, data flows |
| [`docs/skills-protocol.md`](../skills-protocol.md) | The SKILL.md format, OD extensions, discovery and testing |
| [`docs/agent-adapters.md`](../agent-adapters.md) | Per-CLI adapter interface, detection, capability matrix |
| [`docs/modes.md`](../modes.md) | Prototype / Deck / Template / Design System modes in detail |
