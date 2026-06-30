# Copilot CLI: queue + escape unified redesign

**Status**: external design proposal — derived from the research in `queue-esc-report.md`.
**Audience**: copilot-cli maintainers (for discussion if shared).

> **Framing**: This is a proposal from an external user, not a team commitment to ship. It identifies real correctness and discoverability bugs and proposes a coherent fix surface. The unified-list direction in §3.1 is a recommendation with rationale — the team may have shipping reasons to keep the existing two-storage model that aren't visible from outside, and most of the individual fixes (§3.2 Esc-Esc, §3.3 Up-arrow edit, §3.4 wiring `/queue`, §3.5 status bar, §3.8 inline hints) are valuable independently and work under either model.

---

## 1. Problem statement

The current pending-vs-queued model is good underneath but unusable on top.

Confirmed in the source (`queue-esc-report.md §3a`) and live testing on v1.0.65:

- **Pending** (Enter) batches into one prompt that flushes when the current turn ends.
- **Queued** (Ctrl+Q) sends each item as its own separate turn after pending drains.
- **Slash commands during a working agent always land in queued, never pending** — confirmed via test on v1.0.65: submit a story request, then `/compact`, then another story request while the agent works → both stories land in pending, `/compact` lands in queued, and the stories run BEFORE `/compact` despite being interleaved temporally.

The hidden third mode (slash-command-forced-queued) plus the all-pending-first-then-all-queued ordering creates a real correctness bug: `/compact` between two messages no longer means "compact between them" — it means "compact after both", which defeats the user's intent.

These are two underlying mechanisms that do useful work — bundling consecutive text messages and giving slash commands their own dispatch — but the UX exposes them as two separate "storage areas" instead of as bundle-vs-split flags on a single ordered list.

1. **Esc-Esc destroys pending instead of stopping the agent.** The user usually wants the agent to stop so the new message can go through; current behavior cancels the message and lets the agent keep running.
2. **Pending isn't editable.** Up-arrow recalls history, not pending. Users assume their typed messages are still editable; they aren't.
3. **No queue management.** `/queue` does nothing. The only way to alter queued items is Esc-Esc (which destroys them LIFO rather than letting the user pick).
4. **Hint bar lies across terminals.** Shows `ctrl+enter enqueue` based on terminal capability detection, but the actual key varies: works on Kitty-native terminals, falls through to Enter on Termius, gets hijacked by the OS on macOS Terminal.app.
5. **`/queue` is a registered slash command that does nothing** — pure dead code from the user's perspective.
6. **Temporal order is violated when slash commands interleave with messages.** Confirmed bug, breaks `/compact` semantics specifically.

## 2. Design principles

1. **Each key has one intent.** Esc, Ctrl+C, `/queue`, and Up arrow each mean one thing and don't overlap.
2. **The user's work is sacred.** Typed input is never destroyed silently. Cancellation is the user's explicit choice.
3. **Cross-terminal honesty.** The status bar shows keys that actually work on this terminal — no advertising hijacked or no-op keys.
4. **Discoverability.** Both modes (pending and queued) and their separate semantics are surfaced in `/help`, in the status bar, AND inline in the input box (placeholder hints + mode headers when browsing).

## 3. Proposed behavior

### 3.1 Recommended: route slash commands into pending, preserve temporal order

**Scoped recommendation — preserves both existing surfaces (`[pending]` inline and `Queued (N)` block) and changes only the routing rule for slash commands.** The current implementation auto-routes slash commands to the queued stack regardless of the user's intent; this causes the temporal-order bug confirmed in §1. The fix is to let slash commands sit in pending alongside text messages, with bundling logic that splits at slash-command boundaries.

The recommended model:

| Submission | Lands in | Flush behavior |
|---|---|---|
| Plain text + Enter | **Pending** | Merges with adjacent pending text into one LLM prompt at flush time |
| Slash command (any submission key) | **Pending** (new — was queued) | Acts as a split point — runs as its own client operation in temporal order between adjacent text bundles |
| Anything + Ctrl+Q | **Queued** (unchanged) | Each queued item is its own dispatch, sequentially, AFTER pending fully drains |

The "queued" stack now means exactly what its name suggests: items the user explicitly chose to send as separate sequential turns. Slash commands are no longer silently shuffled there.

**Flush rules for pending (the change):**

1. Walk the pending list in submission order.
2. Group consecutive plain-text items into one batched LLM prompt.
3. Each slash command is its own client dispatch (inline, in order).
4. Dispatch each group/command in order, THEN start draining the queued stack.

**Example:** User submits during a working agent, in this order:
- "tell me a story" (Enter)
- "actually make it about pirates" (Enter)
- `/compact` (Enter — auto-pending under the new rule)
- "now another story" (Enter)
- "fix the typo" (Ctrl+Q)

When the agent finishes, pending flushes as:
1. `batch("tell me a story" + "actually make it about pirates")` → one LLM call
2. `/compact` → client operation (in temporal order)
3. `"now another story"` → one LLM call (split because /compact broke the bundle)

Then the queued stack drains:

4. `"fix the typo"` → one LLM call

This fixes the existing bug where `/compact` runs AFTER all messages even when submitted between them, without removing the queued surface that some users may rely on.

**Why this matches Claude Code's model.** Claude Code has no separate queue concept — slash commands are submitted into the same buffer as text and split the bundle in place. This proposal brings Copilot CLI's pending behavior to parity while preserving the explicit Ctrl+Q "give me separate sequential turns" affordance, which Claude doesn't have.

### 3.2 Escape semantics — three escape hatches, three distinct intents

| Action | Intent | Effect |
|---|---|---|
| **Esc-Esc** | "Stop / undo, based on agent state." | State-aware — see Esc-Esc state matrix below. |
| **Ctrl+C** (single press) | "Abandon everything." | Stop agent + clear pending + clear queued. Existing semantics. |
| **`/queue`** | "Surgical edit of queued items." | Opens an interactive picker: list / cancel single / cancel all / reorder. Doesn't touch the agent. |

Single Esc still shows the "press twice" hint — keep the current safety pattern.

**Esc-Esc state matrix.** Esc-Esc means different things based on what the agent is doing right now. Each state is mutually exclusive, and the action is the most-likely-desired one for that state:

| Agent state | Esc-Esc effect | Why |
|---|---|---|
| **In pre-flight gap** — user just hit Enter, message dispatched, but agent hasn't started streaming yet | Pull the in-flight message back into the input field as editable text. Agent never started, so nothing to cancel. | Closes the current "esc doesn't reliably work right after I hit Enter" complaint cluster. Matches Claude Code. |
| **Streaming a response** | Stop the agent. Pending (still on canvas) flushes as the next turn. Queued items fire after. Nothing in input or canvas is destroyed. | This is the original §3.2 behavior. |
| **Idle, input empty** | Open rewind/checkpoints (existing behavior, leave alone). | Already works correctly. |
| **Idle, input non-empty** | Clear the input field (existing behavior, leave alone). | Already works correctly. |

The pre-flight case is the new fix and resolves the existing "Esc-Esc doesn't reliably work" issues. Currently Esc-Esc in that window does nothing useful because the system thinks "no active turn to cancel" — but the user obviously wants to undo their just-pressed Enter. Pulling the message back as editable text is the natural undo.

**Implementation note**: the pre-flight state is already detected — that's *why* Esc-Esc currently no-ops in that window (no in-flight turn to cancel, no pending to peel, no input to clear, so the handler falls through). The engineering work is just routing the already-detected state to the new "pull message back" action instead of dropping it on the floor. The state machine doesn't need to learn anything new; it needs to stop discarding what it already knows.

**Why Esc-Esc no longer peels pending LIFO**: the existing behavior solves the wrong problem. When a user has typed a pending message and the agent is still working, the user almost always wants the agent to stop so their new context can land — not to delete their message and let the agent keep going. The current behavior actively destroys the user's typed work. Pending is now edited via Up arrow (§3.3) or canceled by editing → clearing the input.

### 3.3 Editing pending (steal Claude Code's UX)

Pending is one ordered list of items waiting to be sent (text + slash commands). Treat it as one editable text buffer:

- **Up arrow when pending is non-empty** → loads ALL pending items into the input box as multi-line text in submission order, each on its own line (slash commands keep their `/` prefix). **The `[pending]` markers disappear from the canvas at the same moment** — the user now owns those items again, so they're no longer pending.
- **Up arrow when pending is empty** → history recall (current behavior, unchanged).
- **Enter while editing pending** → re-submits the edited block to pending (replaces whatever pending set was there before). Bundling rules apply at flush time as usual.
- **Ctrl+Q while editing pending** → submits the entire edited block as queued items (one per line) instead. Useful when the user changes their mind and wants the whole thing to run as separate turns.
- **Clearing the input field** (Ctrl+U, Backspace-all, or just deleting the text) → cancels all the (formerly) pending items. No separate "cancel" command needed.
- **Esc while editing pending** → discards the edit and restores the original pending set to the canvas.

Queued items are NOT pulled into pending by Up arrow — they're managed separately via `/queue` (§3.4). Pending and queued remain two distinct UI surfaces; Up just edits pending.

The disambiguation is unambiguous because pending only exists while the agent is busy, and Up's two behaviors trigger in mutually exclusive states.

### 3.4 Queue management — `/queue` and Ctrl+Q (state-aware)

`/queue` currently registers as a slash command but does nothing. Wire it to manage the queued stack (Ctrl+Q items only — pending is managed via Up):

- `/queue` (no args) → interactive TUI picker showing all queued items with: index, age, and preview. Arrow keys to select, Delete/d to cancel, e to edit, q to close.
- `/queue list` → text listing.
- `/queue cancel N` → cancel item by index.
- `/queue clear` → cancel all queued items (does NOT touch pending or the agent).

**Make Ctrl+Q state-aware (symmetric with Up).** Ctrl+Q already exists as the enqueue key. Extend its behavior:

| Input field state | Ctrl+Q action |
|---|---|
| **Non-empty** | Enqueue the typed text as a queued item (current behavior). |
| **Empty + queued stack non-empty** | Open the `/queue` interactive picker. |
| **Empty + queued stack empty** | No-op or brief hint ("Queue is empty"). |

This gives Up and Ctrl+Q symmetric state-aware semantics:

- Up: pending non-empty → edit pending; pending empty → history.
- Ctrl+Q: input non-empty → queue; input empty → manage queue.

Both keys do "the most likely intent for the current state" and the states are mutually exclusive. Removes the need to type `/queue` for the common queue-management case, while keeping the slash command available for keyboard-driven users and scripts.

### 3.5 Status bar honesty

Current code:
```js
we = Te.disambiguateEscapeCodes ? "ctrl+enter" : "ctrl+q"
```

Two problems:

**(a) Terminal-capability detection is wrong for several common cases:**
- Termius reports support → hint says ctrl+enter, but ctrl+enter falls through to Enter (pending).
- macOS Terminal.app reports support → hint says ctrl+enter, but the OS hijacks the keystroke entirely.

**(b) The hint doesn't reflect state-aware keybinds.** Up, Ctrl+Q, and Esc-Esc all do different things in different states (per §3.2, §3.3, §3.4) — the user can't tell what's active right now.

Proposed replacement:

- **Always show `ctrl+q` as the canonical key** (works in every terminal). Optionally show `ctrl+enter` IN ADDITION on terminals where the app has actually received a Ctrl+Enter keystroke during this session — i.e., dynamic detection by observation, not by capability advertisement.
- **Use verbs that reflect the current state**, so what the hint promises is what the key does:

| State | Hint should reflect |
|---|---|
| Working, input non-empty | `esc-esc stop · ctrl+q queue` |
| Working, input empty, queued items exist | `esc-esc stop · ctrl+q manage queue` |
| Working, input empty, pending exists | `esc-esc stop · ↑ edit pending` |
| Idle, input non-empty | `esc-esc clear · ctrl+q queue · ↑ history` |
| Idle, input empty | `esc-esc rewind · ↑ history` |

Hint bar real estate is limited; show at most 2–3 chips, prioritized by which keys do something useful right now and which behavior is surprising/new (e.g. `↑ edit pending` outranks `↑ history` because pending-edit is the unfamiliar mode).

Add the full keybind list with terminal-compatibility notes to `/help` always.

### 3.6 Naming

Internal names (`pending` and `queued`) describe storage, not behavior. Surface-level names should describe what happens at flush time.

Suggested rename (UI strings only; internal symbols can stay):

- `[pending]` → `[batched]` or `[add to current]` (whichever fits the screen)
- `Queued (N)` → `Next turns (N)` or `Up next (N)`

Open to alternatives. The principle: a user reading the label should understand what happens when the agent goes idle.

### 3.7 Help surface

Add a `/help queue` (or similar) section that explains the two modes, when to use each, the four keys (Enter, Ctrl+Q, Esc-Esc, Ctrl+C), and the `/queue` command. Without this, the model stays as opaque as it is today.

### 3.8 Inline UX hints (steal Claude Code's affordances)

Discoverability is the single biggest gap today. Two specific Claude Code patterns to copy, plus one mechanic that falls out for free:

**Placeholder hint in the empty input field — only when surprising.** Terminal users already know Up recalls history; that's universal shell behavior. Don't waste the placeholder advertising it. Show the placeholder ONLY when the new, surprising mode is available:

- Pending exists → `Press up to edit pending messages` (dimmed placeholder inside the input)
- Pending empty → no placeholder (let history-recall stay implicit, like every other shell)

The placeholder disappears the moment the user types or presses Up.

**Mode header above the input when browsing.** When the user presses Up and lands in a recallable state, show a header strip above the input box telling them what they're looking at and where they are. Claude Code uses `── History 27/29 ──`. We need at least:

- Browsing history → `── History {position}/{total} ──`
- Editing pending → `── Editing pending ({N} message{s}) ──`
- Editing queued (if `/queue edit` is invoked) → `── Editing queued item {index}/{total} ──`

The header is self-explaining; combined with users already knowing about history, it tells them what just happened without needing a hint up front.

**Pulling pending into the input IS the cancellation mechanic.** When the user presses Up to edit pending:

1. The `[pending]` markers disappear from the canvas/transcript — the user now owns that text again, so it's no longer pending.
2. The text loads into the input field with the `── Editing pending (N messages) ──` header.
3. From here, every normal input operation does the right thing:
   - **Type/edit freely** → revise the text.
   - **Backspace/Ctrl+U to clear the field** → cancels all the (formerly) pending messages. No separate cancel action needed.
   - **Delete one line** → cancels just that message.
   - **Enter** → re-submits as pending (replaces the old pending set with the edited version).
   - **Ctrl+Q** → submits as queued items instead (one per delimiter line).
   - **Esc** → discards the edit and restores the original pending set to the canvas.

No separate "cancel pending" command is needed because Up→clear is already the natural sequence. The mental model is: pending lives on the canvas only as long as you haven't reached up to grab it back.

**Edge case — agent finishes while user is editing pending.** The agent has nothing to flush (pending is now in the input field, not in `itemQueue`). The agent just goes idle. If the user hits Enter while editing, it sends as a normal idle-mode user turn. Clean — no special-case logic needed.

**Status bar working-state hint** — covered in §3.5. The placeholder hint and mode header above only handle the in-input affordances; the status bar at the bottom is the always-on contract about what keys do right now.

## 4. Out of scope (file separately, don't expand this design)

- Persistence across crash/restart (issue #1437) — orthogonal, can layer on top.
- The phantom "Operation cancelled by user" transcript line (partially fixed in #3826; separate cleanup pass).
- System notifications interleaving with the queue (issue #3517) — separate ordering problem.
- The Esc-Esc-with-empty-queue → rewind escape hatch (works as documented, leave alone).
- Plugin/marketplace management UX (separate issues #3917, #3918).

## 5. Migration / compatibility

- The submission modes don't change underneath. `itemQueue` and `processQueuedItems` stay as-is.
- Esc-Esc behavior change is the only user-visible regression: users who relied on Esc-Esc to peel pending will need to learn Up→edit→delete-line instead. Worth a one-time `/release-notes`-style callout on first run after upgrade.
- `/queue` going from no-op to interactive is purely additive.
- Status bar string changes — visible but trivial.

## 6. Open questions

1. **Should Up-arrow's load-pending behavior cycle?** E.g. press Up once → load pending; press Up again → recall history. Or should it be modal (only one behavior per state)? Lean toward modal — pending exists OR history is loaded, never both at the input.
2. **The inter-queued-item pause** — is 500ms the right number, or should it be configurable? Maybe just ship a sensible default and add a `pauseBetweenQueued` setting if anyone complains.
3. **Naming** — `[batched]` / `Next turns (N)` is one suggestion; better names welcome.
4. **`/queue` keybind** — worth adding Ctrl+L (or similar) for direct invocation, or is the slash command enough?

## 7. References

- Research: `queue-esc-report.md` (this folder).
- Source analysis: `queue-esc-report.md §3a` (extracted from app.js v1.0.65).
- Cross-terminal evidence: `queue-esc-report.md §3a` (verified Termius + Terminal.app).
- Claude Code comparison: `queue-esc-report.md §4a`.
- Related copilot-cli issues: see `queue-esc-report.md §8` for full inventory.
