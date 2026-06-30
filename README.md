# Copilot CLI — Queue + Esc Redesign Proposal

An external UX design proposal for GitHub Copilot CLI's message-queue and
escape-key behavior, with the research that backs it.

## Rendered proposal (GitHub Pages)

**[Read the proposal →](https://dfrysinger.github.io/copilot-cli-queue-esc-proposal/copilot-cli-queue-and-esc-redesign-propo-20260630T210853Z-w9rj3m/)**

## Contents

| File | What it is |
|---|---|
| [`proposal/queue-esc-design.md`](proposal/queue-esc-design.md) | The design proposal: unified pending/queued model, state-aware Up / Ctrl+Q / Esc-Esc, status-bar honesty, inline UX hints. |
| [`research/queue-esc-report.md`](research/queue-esc-report.md) | The research it's built on: 22-issue complaint surface, source-code analysis of the shipped `app.js` bundle, doc-vs-reality contradictions, cross-terminal evidence, and a Claude Code comparison. |

## Summary

Copilot CLI has two distinct ways to submit a message while the agent is
working — **pending** (Enter; batches into one prompt at the next turn
boundary) and **queued** (Ctrl+Q; each item runs as its own sequential turn).
The underlying mechanics are useful, but the surface hides the distinction,
fights the user on cancellation, routes slash commands in a way that breaks
temporal order (e.g. `/compact` between two messages runs *after* both), and
advertises keybinds that don't work on common terminals.

The proposal keeps both modes but makes them coherent and discoverable:

- **Route slash commands into pending** so temporal order is preserved.
- **State-aware Esc-Esc** — undo a just-sent message in the pre-flight gap;
  stop the agent (without destroying pending) while streaming.
- **Up arrow edits pending** as one editable buffer; pulling it into the input
  is also how you cancel it.
- **State-aware Ctrl+Q** — enqueue when typing, open the queue manager when the
  input is empty.
- **Wire `/queue`** (currently a no-op) to list / cancel / clear.
- **Honest, state-aware status bar** that shows keys that actually work on
  this terminal.

This is an external proposal, not a team commitment to ship. Each fix is
independently valuable and most work under the existing model.
