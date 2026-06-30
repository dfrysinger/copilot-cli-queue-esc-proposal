# Queued Messages + Esc Behavior: Deep Dive Report

_Compiled 2026-06-24 against Copilot CLI `v1.0.64-3` on macOS._
_Source: official docs (`docs.github.com/.../cancel-and-roll-back` + `code.claude.com/docs/en/interactive-mode`), 21 open + 1 closed issue in `github/copilot-cli`._

---

## 1. What the docs say (the contract)

From [Canceling a GitHub Copilot CLI operation and rolling back changes](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/cancel-and-roll-back):

### Esc — single press

| Current state | Documented behavior |
|---|---|
| Active, **no** queued prompts | Cancels the running operation |
| Active, **with** queued prompts | **Clears the queued prompts WITHOUT stopping the current operation** |
| Dialog / overlay / picker is open | Closes the dialog/overlay/picker |
| Idle | Shows hint: press again quickly to open rewind picker |

### Ctrl+C — hard stop
- Immediately cancels the active operation AND clears any queued prompts in a single keypress.
- A file write already in progress completes; planned remaining writes are abandoned.
- A second Ctrl+C within 2s on an empty input area exits the session.

### Esc-Esc when idle + empty input
- Opens the rewind picker (workspace snapshot list).

### Doc-vs-reality flag
The docs make a strong promise: Esc with queued prompts should clear ONLY the queue, never stop the current op. Multiple open bug reports say this contract is not honored in practice. See §3.

---

## 2. The complaint surface (22 issues)

Grouped by theme. ★ = highest signal / most-discussed.

### A. Queue control primitives missing

| # | Title | State | Summary |
|---|---|---|---|
| ★ #2055 | Queued messages cannot be cleared without canceling running job | open | Both Esc and Ctrl+C cancel the running op; no way to surgically clear queue. Ctrl+U doesn't help. Direct contradiction of the documented Esc contract. |
| ★ #1857 | Allow users to cancel or remove enqueued messages before they are executed | open | Proposes `/queue` slash command, `/queue cancel <n>`, or `Ctrl+Shift+Q` for clear-all. Has 7 comments; one suggests Esc already works, others confirm Esc cancels ALL queued, not selectively. |
| #2378 | Cancel queued messages without stopping current op | open | Same root request as #2055, framed as feature. |
| ★ #1781 | Interactive queue manager — browse, inspect, remove with confirmation | open | Proposes TUI overlay with ↑/↓, Enter/Space, Delete with (y/N), Esc to close. Same primitive gap as #1857. |

### B. Up-arrow edit semantics (cross-tool parity)

**[#2905](https://github.com/github/copilot-cli/issues/2905) — "Up arrow to edit a queued message adds duplicate instead of replacing" (open).**

Filed as a Claude-Code parity bug, but actually a misdiagnosis. Copilot CLI's Up-arrow does what it always does — recall the last-sent message from history. When the user has just queued exactly one message, the last-sent and the queued message happen to be the same text, which gives the illusion of "Up recalled my queued message." Enter then sends a new message normally — there is no edit-mode at all, so it lands as a second queue entry. Claude Code, by contrast, has an actual edit-mode for queued messages bound to Up. So the real gap is **"no edit-mode for queued messages exists in Copilot CLI"** — not "the edit-mode is buggy." This reframes the fix as a feature add rather than a bug fix; see §4 proposal #4.

### C. Esc keybinding overload / state-machine issues

| # | Title | State | Summary |
|---|---|---|---|
| ★ #3692 | Esc should cancel current task AND focus pending queued prompt | open | Opposite direction from #2055/#1857: user wants Esc to stop current task AND surface the queued prompt for editing. Notes that "Esc twice to cancel" hint doesn't match reality — often takes multiple presses to actually stop. Comment from @IanGraingerGMSL pushes back: Esc should mean STOP, not redirect tokens; needs a separate "interrupt-and-send" key. |
| #2508 | Esc to cancel is accidentally triggered too much | open | User wants a config to disable or require Esc-Esc, citing wrong-split-window keypresses. |
| #2972 (closed) | Esc cancels agent but clears typed input buffer | closed | Auto-closed by @examon citing v1.0.36 fix (double-Esc + preserve input). Worth verifying still holds. |
| #3430 | Esc in `/tasks` overlay also dismisses active question prompt | open | Esc keypress handled twice (overlay close + question dismiss). Focus/event-propagation bug. |
| #2703 | Session hangs after work appears complete; Escape recovery enters permanent "Cancelling" | open | Recovery path deadlock. |
| ★ #2770 | CLI gets stuck on "Cancelling" and stops accepting Enter | open | Input state-machine deadlock during rate-limit/server-hang. Slash commands become unusable. 8–10 min recovery. |
| #3720 | Esc-Esc does not save half-typed command in history | open | Rewind-picker entry loses input buffer. |
| #3826 | "Operation cancelled by user" is re-injected as a user message | open | After cancel, this string appears as a fresh user turn with timestamp; agent "replies" to it. Reported as regression in 1.0.63. |

### D. Ctrl+C overload

| # | Title | State | Summary |
|---|---|---|---|
| #1422 | Ctrl+C cancels operation even though UI states it is Esc | open | Long thread debating SIGINT-vs-copy precedent. Real grievance: status bar says "Esc to cancel" but Ctrl+C also cancels (and second Ctrl+C exits). |
| #3198 | Ctrl+C too efficient — cancels whole session | open | Same family. |
| ★ #3620 | Ctrl+C has too many overloads (copy, clear prompt, cancel) | open | Triple-meaning keybinding makes copy-text-from-question impossible without cancelling the question. |

### E. Enqueue keybinding mismatch

| # | Title | State | Summary |
|---|---|---|---|
| ★ #3760 | Status bar shows "ctrl+enter enqueue" but ctrl+enter adds newline; ctrl+q is the real enqueue | open | Documentation/UI lies about which key enqueues. Windows 11. v1.0.61. |
| #1437 | Ctrl+Enter hotkey not working in TUI (inserts newline instead) | open | Cousin of #3760. |
| #2624 | Consistent enqueue shortcut to match VS Code's Alt+Enter | open | Asks for cross-product consistency. |
| #2128 | Change model for enqueued prompts | open | Mid-queue model reassignment. |

### F. Queue durability / lifecycle bugs

| # | Title | State | Summary |
|---|---|---|---|
| #3184 | When Copilot crashes with enqueued messages, you lose them | open | No persistence across crash. |
| ★ #3344 | Messages stranded in `Queued (N)` UI region during bg-subagent runs | open | Deep technical report (root cause: `hasActiveBackgroundWork()` guard in `enqueueItem` refuses to start `processQueue` while ANY bg agent is running; drain depends on bg-task completion callback). "Submit another message" is a placebo workaround; the real driver is bg agent completion. Affects ~38% of turns in long autopilot sessions. |
| ★ #3517 | Queued user messages + system_notifications delivered out of send order | open | FIFO contract violated. Two queues (user msgs + sys_notifications) interleave non-deterministically. Causes assistant to answer stale messages. |
| #3879 | Status line conflates "actively generating" with "idle + background work running" | open | Cosmetic busy state when input is functionally free; users can't tell when typing will queue vs. run immediately. Compounds the N-1 lag pattern. |
| #3915 (you filed) | /compact activity indicator fires immediately when next message is queued | open | Visual-only; functionally correct. |
| #3919 (you filed) | Docs/UX: queued vs pending terminology | open | Two terms, no documented distinction. |

### G. Adjacent

| # | Title | State | Summary |
|---|---|---|---|
| #3265 | Add `/pause` or `/stop` slash command to gently interrupt an agent | open | Non-Esc, non-Ctrl+C path that stops at next reasoning-end without mid-token cancel. |

---

## 3. Doc-vs-reality contradictions (the core bug class)

| Documented behavior | Reported reality | Issue(s) |
|---|---|---|
| Esc with queued prompts clears ONLY the queue, current op continues | Esc cancels current op AND clears queue (or just cancels op) | #2055, #1857 (multiple commenters), #2378 |
| Esc closes dialog/overlay only | Esc in `/tasks` overlay also dismisses the underlying question prompt | #3430 |
| Ctrl+C clears queued prompts in a single keypress | Ctrl+C cancels current op but queue interaction unclear | #1422, #3198 |
| Status bar advertises Esc to cancel | Ctrl+C also cancels (undocumented in status bar) | #1422, #3620 |
| Status bar advertises ctrl+enter enqueue | ctrl+enter adds newline; ctrl+q is the real enqueue | #3760, #1437 |
| Queue drains at next turn boundary | Queue strands behind bg-subagent runs; only drains on bg-task completion callback | #3344 |
| (Implicit) FIFO delivery | Out-of-order delivery between user msgs and system_notifications | #3517 |
| (Implicit) cancel is silent | "Operation cancelled by user" injected as a phantom user turn | #3826 |

---

## 3a. Source-code analysis: pending vs queued (from app.js v1.0.65)

I extracted the shipped JS bundle (`@github/copilot-darwin-arm64@1.0.65 → package/app.js`, 8.1 MB minified) and traced the data structures:

### Two distinct storages exist

| Storage | What it holds | Where it surfaces |
|---|---|---|
| `this.itemQueue` (`Array`, push/shift FIFO) | User-typed messages, slash commands, model-change requests, resume-pending wakes — anything coming from the user input box while the agent is busy | **`Queued (N)` UI block** at the bottom of the screen |
| `this.immediatePromptProcessor.messageQueue` | Internal "immediate" messages: system notifications, post-tool-use failure context, post-hook injections | Renders inline in the transcript flow, not in `Queued (N)` |

Both fire the same `pending_messages.modified` event when modified, which the UI subscribes to and re-renders.

### What the code does

`enqueueUserMessage()` always pushes to `itemQueue` (`unshift` if `prepend=true`, else `push`). User-typed messages submitted with Enter or Ctrl+Enter go through this path.

```js
enqueueItem(e) {
  this.addItemToQueue(e);
  this.emitEphemeral("pending_messages.modified", {});
  if (!this.isProcessing) {
    if (e.kind !== "resume_pending" && this.hasNotifyingBackgroundWork()) return;  // ← #3344 root cause still here
    this.processQueue().catch(...);
  }
}

hasNotifyingBackgroundWork() {
  return this.hasRunningAgents() ? true : (this.inheritedShellContext ? false : this.getSessionShellContext()?.hasCommandsAwaitingNotification() ?? false);
}
```

The #3344 short-circuit is still present in v1.0.65 (slightly renamed from `hasActiveBackgroundWork` to `hasNotifyingBackgroundWork` since the v1.0.49 analysis).

### The `[pending]` inline decoration

The string `[pending]` is rendered by a transcript-item component when an internal "loading" flag is set on the item AND the item is NOT a user-typed message with a particular sub-condition met:

```js
r && !(t === "user" && Q) && X.push("[pending]")
```

Where `r` is the loading flag and `t` is the entry type. I cannot decode `Q` from the minified source without dynamic inspection, but the structure strongly suggests `[pending]` is a UI-level loading marker applied during the lifecycle of a SINGLE transcript entry being processed — not a queue indicator.

### RESOLVED — two distinct send modes, keyed by the submit keybind

**Confirmed empirically by @dfrysinger on v1.0.65 (2026-06-24):**

| Submit key | Lands in | Send semantics |
|---|---|---|
| **Enter / Ctrl+Enter** | inline `[pending]` markers in the transcript | All pending messages send as **one batched prompt** when the current turn finishes |
| **Ctrl+Q** | bottom `Queued (N)` block | Each queued message sends as **its own separate turn**, one at a time, only after all pending are drained first |

So `[pending]` and `Queued (N)` aren't the same data shown twice (H1), and they aren't two lifecycle stages of the same item (H2). They are **two completely different submission paths with different batching semantics** — but the UI explains none of this.

### What this means for the design

The distinction is potentially valuable:
- **"Pending" (Enter)** = "extend my current message to the agent" — useful for adding context/corrections you thought of after hitting send
- **"Queued" (Ctrl+Q)** = "give me N more turns in a specific order" — useful for scripting a sequence of independent operations

But the current UX has fatal problems:
- **Discoverability is zero.** Ctrl+Q is not in the docs, not in the visible hint bar (`Working esc cancel · ctrl+enter enqueue` shows ctrl+enter, which actually does the SAME thing as Enter — both go to `[pending]`).
- **Naming is misleading.** "Pending" and "Queued" both sound like "waiting to send" — neither name conveys the batch-vs-sequential distinction.
- **Esc-Esc behavior differs.** Esc-Esc peels from one of these stacks LIFO (need to verify which one — likely `[pending]` since `removeMostRecentPendingItem` is named for the pending side).
- **The hint bar is terminal-dependent and worse than misleading.** Code: `we = Te.disambiguateEscapeCodes ? "ctrl+enter" : "ctrl+q"`. The hint shows `ctrl+enter enqueue` only when the terminal supports the Kitty keyboard protocol. Observed behavior across terminals:
  - **Termius (verified)**: Ctrl+Enter is byte-identical to Enter at the input layer → falls through to Enter handler → pending. Ctrl+Q works.
  - **macOS Terminal.app (verified)**: Status bar shows `ctrl+enter enqueue` (Copilot CLI believes the terminal disambiguates), but Terminal.app's window-level shortcut steals Ctrl+Enter before it reaches the app — pastes clipboard into the input AND opens "New Tab with Profile" menu. The hint promises a key the OS hijacks. Ctrl+Q is the only working enqueue path despite not being advertised.
  - **Kitty-native terminals (Ghostty, WezTerm — unverified)**: Expected to disambiguate Ctrl+Enter and route it to the enqueue path correctly.
  - The hint is therefore worse than wrong: on Terminal.app, following the hint actively breaks your input (clipboard paste + modal menu) — the opposite of helpful.

The new design doc needs to either (a) make the distinction first-class and explained, or (b) collapse them into one mode.

### Confirmed from code

- **There is exactly ONE user-typed-message queue** (`itemQueue`). No "pending queue" + "queued queue" split for user messages.
- **`/queue` command is not in the source** (verified by absence of any `/queue` slash-command registration).
- **The clear-queue primitives exist**: `clearPendingItems()` (clears everything), `removeMostRecentPendingItem()` (peels the most recent — this is what Esc-Esc binds to, matching @dfrysinger's Q-A finding).
- **`hasNotifyingBackgroundWork()` still gates `processQueue` start** — #3344's root cause is unchanged in v1.0.65.
- **`onUserAbort` calls `clearPendingItems`** — confirms Ctrl+C clears the whole queue as documented.
- **No `enqueuedAt` timestamp** on items in `itemQueue` (per #3517 ordering complaint) — items are FIFO by insertion order via `nextPendingOrder++`.

---

## 4. Proposed fix surface (synthesis)

This is a list of distinct mechanisms reviewers/users have proposed. They're not mutually exclusive.

1. **Make Esc honor its documented contract.** With queued prompts: clear ONLY the queue, do not touch the running op. (Fixes #2055, #2378.)
2. **Add a `/queue` slash command** with subcommands: `list`, `cancel <n>`, `clear`. (#1857, #1781.)
3. **Add interactive queue manager TUI** (Ctrl+Q while queue non-empty opens a panel; ↑/↓/Enter/Delete/Esc keybindings). (#1781.)
4. **Add explicit edit-mode for queued messages** (Claude-Code parity) — because Up-arrow recall is and should remain history navigation, this needs a separate keybinding (e.g. `Alt+Up` or a `/queue edit <n>` action). (#2905, reframed.)
5. **Separate keybindings** for:
   - cancel current task (Esc)
   - clear queue (configurable; doc currently says Esc, code says no)
   - interrupt-and-send-queued (new) — addresses #3692 without violating "Esc means stop"
   - gentle pause at next reasoning boundary (`/pause` or `/stop`) — #3265
6. **Status-line truth** — fix the "ctrl+enter enqueue" label that doesn't match the actual ctrl+q binding. (#3760, #1437.)
7. **Distinguish busy states** — "generating" vs "idle + bg work" so users know when typing will queue. (#3879.)
8. **Remove the `hasActiveBackgroundWork()` short-circuit in `enqueueItem`** — let queue drain at the main agent's turn boundary regardless of bg agents. (#3344.)
9. **Strict FIFO across user messages and system_notifications**, OR expose a monotonic `enqueuedAt` so the assistant can re-sort. (#3517.)
10. **Suppress phantom "Operation cancelled by user" user turn** after cancel. (#3826.)
11. **Persist queue across crash/resume.** (#3184.)
12. **Local slash commands (`/restart`, `/session info`) remain usable while a request is cancelling/hung** — break the "Cancelling" deadlock. (#2770, #2703.)

---

## 4a. Comparison: Claude Code's keybinding model

Source: [code.claude.com/docs/en/interactive-mode](https://code.claude.com/docs/en/interactive-mode) (canonical Anthropic docs).

### The big design difference

**Claude Code has no multi-message queue concept.** There is one input buffer. When Claude is working, you can type in it; Enter sends it as the next turn after Claude finishes. There is no `Queued (N)` stack, no `ctrl+q enqueue`, no `/queue` command. This sidesteps the entire complexity surface that produces §2-A, §2-B, §2-E, and §2-F bugs in Copilot CLI.

This is a real architectural fork. Copilot CLI's queue is more powerful (line up several follow-ups, let them stream) but pays for it with the bug class above. Before adding more queue-management surfaces (§4 proposals #2 and #3), it's worth asking whether the queue should exist at all, or be limited to a single pending message.

### Direct keybinding comparison

| Action | Claude Code | Copilot CLI (docs) | Copilot CLI (reality, per bugs) |
|---|---|---|---|
| Interrupt running turn | `Esc` — "Stop the current response or tool call mid-turn so you can redirect. Claude keeps the work done so far" | `Esc` (when no queue) | Often takes multiple presses (#3692); can get stuck "Cancelling" (#2770) |
| Hard stop | `Ctrl+C` — "Interrupt, or clear input" | `Ctrl+C` — hard stop | Same; status bar doesn't advertise it (#1422) |
| Exit session | `Ctrl+C` twice on empty input, OR `Ctrl+D` | `Ctrl+C` twice within 2s on empty input | Same |
| Clear input line (with text) | `Esc + Esc` — clears AND saves draft to history so `Up` recalls it | Not documented; `Ctrl+U`/`Ctrl+W` work but no draft-save | Not surfaced |
| Open rewind / checkpoint picker | `Esc + Esc` when input is empty | `Esc + Esc` when idle + input empty | Same; loses half-typed input (#3720) |
| Enqueue message (during turn) | N/A — no queue concept | `Ctrl+Q` (real); status bar lies and says `Ctrl+Enter` (#3760) | Same lie |
| Edit queued message | N/A — no queue concept | Not implemented | Up arrow recalls history, not the queue (clarified in §2-B) |
| Clear queue | N/A — no queue concept | `Esc` per docs; `Ctrl+C` per docs (also clears) | Reportedly cancels current op too (#2055) |
| Stop background subagents | `Ctrl+X Ctrl+K` (press twice within 3s to confirm) | `/tasks` overlay (no global keybind) | #3845 is the parity ask |
| Multiline newline | `Shift+Enter` (native in iTerm2/WezTerm/Ghostty/Kitty/Warp/Apple Terminal/Windows Terminal), `Option+Enter`, `\\+Enter`, or `Ctrl+J` | Unclear; status bar says `Ctrl+Enter` newline (#3760) | `Ctrl+Enter` does insert newline |
| Pause at next reasoning boundary | Not bound | Not implemented (#3265 asks for `/pause`) | — |
| Cycle command history | `Up`/`Down` — cursor nav first when input wraps, then history | `Up`/`Down` — straight history recall | Same as docs |
| Reverse search history | `Ctrl+R` | Not documented | — |
| Edit prompt in `$EDITOR` | `Ctrl+G` or `Ctrl+X Ctrl+E` | Not documented | — |
| Paste image from clipboard | `Ctrl+V` (Cmd+V in iTerm2, Alt+V in WSL) — inserts `[Image #N]` chip | Not documented for terminal; only file attach | — |

### Five idioms Claude Code gets right (worth borrowing)

1. **`Esc + Esc` with text = clear-and-save-to-history.** Makes the destructive action recoverable: press Up to bring the draft back. Copilot CLI's `Esc + Esc` does nothing useful when input has text (#3720 captures the related "doesn't save half-typed" complaint).
2. **`Esc` interrupts but explicitly keeps work-so-far.** Copilot CLI's #3826 shows the opposite — cancel injects a phantom `Operation cancelled by user` user turn, which is worse than Claude's "stop quietly, you can redirect" semantics.
3. **No multi-message queue.** Single-buffer model removes 4 entire bug categories from the surface. If Copilot CLI keeps the queue, it should be explicit about why the cost is worth it.
4. **`Ctrl+X Ctrl+K` (press twice) for background subagent stop.** A dedicated kill-bg-only keybind instead of overloading Esc/Ctrl+C with "and also touch bg work." See #3845 in Copilot CLI for the parity ask.
5. **Documentation honesty.** Claude Code's docs list every keybinding with exact context strings. Copilot CLI's status bar advertises `Esc to cancel` and `Ctrl+Enter enqueue` neither of which fully match reality (#1422, #3760).

### Three Claude Code idioms that *don't* solve Copilot CLI's problems

1. **No queue ≠ no value of having one.** Multi-message queueing is a real workflow: line up "now do X, then commit, then run tests" while the agent grinds. Don't drop the feature; fix the manager.
2. **Claude Code's `Ctrl+C` overload is the same as Copilot CLI's** ("interrupt OR clear-input"). The Claude docs just call it out clearly. Copilot CLI could close half its complaints by documenting the same dual meaning in the status bar instead of hiding it.
3. **`Up`/`Down` cursor-then-history behavior in wrapped input.** Useful for multiline editing, but orthogonal to the queue-edit complaint.

### Implication for §4 proposals

| §4 proposal | Claude Code parity verdict |
|---|---|
| #1 Esc honors docs (clear queue only) | Claude has no queue; the doc contract is Copilot-specific. Worth keeping IF queue stays. |
| #2 `/queue` slash command | No Claude parity; Copilot-specific feature add. |
| #3 Interactive queue manager TUI | No Claude parity; Copilot-specific feature add. |
| #4 Edit-mode for queued msgs | No Claude parity. If queue stays, needs explicit keybind (e.g. `Alt+Up`), since Up is history. |
| #5 Separate keybindings (cancel / clear / interrupt-and-send / pause) | Claude has Esc (interrupt) + Esc-Esc (clear/rewind) + Ctrl+X Ctrl+K (bg stop). Cleaner than the proposed 4-way split. |
| #6 Status-line truth | Direct port from Claude Code's "every keybind has exact context string" approach. Cheap win. |
| #7 Distinguish busy states | Claude doesn't have this problem because no bg-subagent + main-agent overlap. Copilot-specific. |
| #8 Remove `hasActiveBackgroundWork()` short-circuit | Copilot-specific (#3344). |
| #9 Strict FIFO for queue + sys_notifications | Copilot-specific (#3517); no queue in Claude. |
| #10 Suppress phantom "cancelled" turn | Direct port of Claude's "Esc keeps work done so far, stops quietly" semantics. |
| #11 Persist queue across crash | Copilot-specific. |
| #12 Local slash commands work during "Cancelling" | Copilot-specific bug. |

### Open architectural question for you

Before filing any of the proposal-bucket PRs, the meta-question is:

> **Should Copilot CLI's queue stay multi-message, become single-pending-only (Claude parity), or become opt-in?**

Pick one and the §4 surface collapses accordingly. The current state — multi-message queue with no manager — is the worst of both worlds.

---

## 5. Open questions — please test on current build (v1.0.65)

Tested live by @dfrysinger on v1.0.65 (macOS, Termius); ⚠ = needs follow-up.

### Q-A — Esc with queued prompts: ANSWERED

| Situation | Actual behavior |
|---|---|
| Single Esc | Shows a hint to press it twice; takes no action on its own |
| Esc-Esc, queue non-empty, agent working | **Clears the most-recently-queued message** (one at a time, peeling the stack). The current op KEEPS RUNNING. |
| Esc-Esc, queue empty, agent working | Stops the agent's current operation |
| Esc-Esc, agent working, text in input | Stops the agent's current operation (text in input is preserved) |
| Esc-Esc, agent idle, text in input | Clears the input |
| Esc-Esc, agent idle, input empty | Opens rewind picker (in this test it surfaced "Rewind is not available because you're not in a git repository") |

**Verdict:** The CLI **does** honor a contract similar to the docs, but it's a different contract than the docs describe:
- Docs say single Esc with queued prompts clears the queue. Reality: single Esc only shows a hint. **Esc-Esc** is the actual binding for everything (queue clear, op cancel, input clear, rewind).
- Docs say Esc with queue clears the *whole* queue. Reality: Esc-Esc clears *one entry at a time* (LIFO/peel — last queued first).
- The behavior is internally consistent and arguably nice (one-at-a-time peel respects user intent), but the docs are wrong about both the keypress count and the all-or-one semantics. **This is a docs bug, not a code bug.**

### Q-B — Up-arrow with a queued message [RESOLVED]

**Resolution:** Up does what it always does — recalls the last-sent message from history. The illusion of "Up brings up the queued message" appears because the queued message *is* the last-sent message when you just queued one. There is no edit-mode for queued messages in Copilot CLI today; submitting from the input always sends a new message, which is why the supposed "edit" lands as a duplicate. See §4 proposal #4 for the feature add.

### Q-C — Input buffer preservation across Esc-cancel: ANSWERED

Confirmed: when Esc-Esc cancels the agent and there is text in the input box, **the typed text is preserved**. The #2972 fix from v1.0.36 still holds in v1.0.65.

### Q-D — Status bar enqueue hint vs reality: ANSWERED

From the screenshot, the status bar during agent work reads:

```
● Working   esc cancel · ctrl+enter enqueue                                        Claude Opus 4.8 · (14%)
```

**On macOS (v1.0.65), confirmed:**
- Plain `Enter` while agent working → enqueue (marked `[pending]`)
- `Ctrl+Enter` while agent working → same behavior, also enqueue (marked `[pending]`)

So `Ctrl+Enter` is redundant on macOS — plain Enter does the same thing. The status-bar hint `ctrl+enter enqueue` is technically correct but misleading because it omits the plain-Enter binding. Combined with #3760 (which reports `Ctrl+Enter` inserts a newline on Windows + PowerShell + Windows Terminal), the picture is:

- macOS: both Enter and Ctrl+Enter enqueue; the hint mentions only one of them.
- Windows: Ctrl+Enter reportedly inserts a newline (#3760), so the hint is actively wrong on that platform.

Recommended fix per platform:
- **macOS**: change status bar hint to `enter enqueue` (or `enter / ctrl+enter enqueue`) — plain Enter is the discoverable binding.
- **Windows**: investigate why Ctrl+Enter doesn't enqueue (per #3760); if Ctrl+Q is the actual binding there, status bar should say `ctrl+q enqueue`.

### Q-D-extension — "Pending" vs "Queued" both appear in the UI [BIG FINDING]

Screenshot 1 (v1.0.65 during a multi-message test) shows BOTH:
- Multiple lines decorated with `└ [pending]` inline in the conversation flow
- A separate `Queued (2)` header at the bottom containing entries

It's possible these are the same data shown in two places, OR (more likely, given #3344) they are different states — `[pending]` = will run at the next turn boundary; `Queued (N)` = blocked behind background work, drains only when bg agents complete. Either way, this is exactly the confusion #3919 flagged: users can see "pending" and "queued" simultaneously with no documented distinction. **Documentation gap confirmed.**

### Q-E — Phantom "Operation cancelled by user" turn: PARTIAL

Confirmed: `Operation cancelled by user` DOES appear in the transcript with the cyan-bullet styling of a system/user message. **BUT**: in this test, the agent did NOT respond to it. So #3826's specific symptom ("agent then 'replies' to it as though it were a new instruction") was not reproduced on v1.0.65 in this session.

Possibly already fixed since 1.0.63 (the version #3826 was filed against). Worth commenting on #3826 with the v1.0.65 negative repro.

The cosmetic concern remains: the line is rendered styled like a real conversation entry, which is potentially confusing. Could be downgraded to a system note or rendered differently from user/agent content.

### Q-F — `/queue` slash command: ANSWERED

Confirmed: `/queue` does nothing. No such command exists. Validates every queue-management feature request (#1857, #1781, #2378, etc.).

### Q-G — Double-Esc when idle (rewind picker): ANSWERED

Confirmed: Esc-Esc when idle + empty input does attempt to open the rewind picker. In this test the session wasn't in a git repo, so the picker surfaced `! Rewind is not available because you're not in a git repository`. **Behavior matches docs**, just blocked by the non-git environment.

### Q-H — Ctrl+C behavior: ANSWERED

Confirmed:
- Single Ctrl+C with agent working → cancels the current op and shows a hint that pressing Ctrl+C again quickly closes the app.
- Matches the documented behavior.

### Q-I — Stuck "Cancelling" state

Not encountered in this session. Existing evidence (#2770) stands.

### ⚠ Q-D-followup — RESOLVED

(See Q-D above; Ctrl+Enter on macOS behaves identically to plain Enter.)

---

## 6. Remaining empirical unknowns

The pending-vs-queued question is resolved (see §3a). A few smaller questions remain that could be answered with quick tests:

1. **Esc-Esc selectivity** — does Esc-Esc peel from the `[pending]` stack, the `Queued (N)` stack, or both LIFO regardless of which? Test: submit `P-1`, `P-2` (Enter) and `Q-1`, `Q-2` (Ctrl+Q) during a long task; Esc-Esc once and observe which marker disappears.
2. **Pending batching format** — when N pending messages flush as one batched prompt to the agent, are they concatenated with newlines, sent as separate user turns in one transcript snapshot, or merged into a single string? Test: send `PEND-A`, `PEND-B`, `PEND-C` then `please quote back exactly the user message(s) you received`.
3. **Pending vs queued ordering invariant** — if you have `P-1`, `Q-1`, `P-2` (alternating modes) does the agent actually see all-pending-first-then-all-queued, or does temporal order survive? Test: send in that interleaved order with `please quote back the order you received user messages`.
4. **Crash/restart persistence** — does either stack survive a `Ctrl+C Ctrl+C` exit + restart? (Issue #1437 suggests no.) Test: build up both stacks, exit, restart, check `/resume`.

These are nice-to-have for the design doc but not blockers. Move to §7 / design.

---

## 7. Recommended next steps

Once tests in §6 are run and §3a hypotheses resolved:

1. **Document the actual pending/queued lifecycle** in §3a (replace H1/H2 with the verified model).
2. **Triage the proposal surface** in §4 into "ship as one PR" buckets. Strong candidates for a single coherent fix:
   - **Queue manager PR**: `/queue list|cancel|clear` + interactive TUI overlay + edit-mode keybind for queued messages + persistence-across-crash + status-line truth. Resolves: #1781, #1857, #2055, #2378, #2905, #3184, #3760, #1437, #2624.
   - **Keybinding separation PR**: distinct Esc vs cancel-current vs interrupt-and-send vs `/pause`. Resolves: #3692, #2508, #2972 (verify), #3265, #1422, #3198, #3620.
   - **Drain/order correctness PR**: remove the `hasNotifyingBackgroundWork()` short-circuit in `enqueueItem`, single FIFO with `enqueuedAt`, no phantom "cancelled by user" turn. Resolves: #3344, #3517, #3826.
   - **Status-line truth PR**: generating vs idle+bg, accurate hint text. Resolves: #3879, #3760, #1422.
3. **Move into the design doc** (separate from this research doc) once the proposals are bucketed.

---

## 8. Inventory: full issue list with URLs

Filed by us this session:
- [#3915](https://github.com/github/copilot-cli/issues/3915) `/compact` activity indicator
- [#3919](https://github.com/github/copilot-cli/issues/3919) queued vs pending terminology

Existing — queue control:
- [#2055](https://github.com/github/copilot-cli/issues/2055), [#1857](https://github.com/github/copilot-cli/issues/1857), [#2378](https://github.com/github/copilot-cli/issues/2378), [#1781](https://github.com/github/copilot-cli/issues/1781), [#2905](https://github.com/github/copilot-cli/issues/2905), [#2128](https://github.com/github/copilot-cli/issues/2128)

Existing — Esc:
- [#3692](https://github.com/github/copilot-cli/issues/3692), [#2508](https://github.com/github/copilot-cli/issues/2508), [#2972 (closed)](https://github.com/github/copilot-cli/issues/2972), [#3430](https://github.com/github/copilot-cli/issues/3430), [#2703](https://github.com/github/copilot-cli/issues/2703), [#2770](https://github.com/github/copilot-cli/issues/2770), [#3720](https://github.com/github/copilot-cli/issues/3720), [#3826](https://github.com/github/copilot-cli/issues/3826)

Existing — Ctrl+C overload:
- [#1422](https://github.com/github/copilot-cli/issues/1422), [#3198](https://github.com/github/copilot-cli/issues/3198), [#3620](https://github.com/github/copilot-cli/issues/3620)

Existing — enqueue keybinding:
- [#3760](https://github.com/github/copilot-cli/issues/3760), [#1437](https://github.com/github/copilot-cli/issues/1437), [#2624](https://github.com/github/copilot-cli/issues/2624)

Existing — queue lifecycle:
- [#3184](https://github.com/github/copilot-cli/issues/3184), [#3344](https://github.com/github/copilot-cli/issues/3344), [#3517](https://github.com/github/copilot-cli/issues/3517), [#3879](https://github.com/github/copilot-cli/issues/3879)

Adjacent:
- [#3265](https://github.com/github/copilot-cli/issues/3265) `/pause` or `/stop`
