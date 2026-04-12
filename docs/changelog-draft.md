# Changelog Draft

*Internal working doc — not in nav. Copy entries to changelog.md before release.*

---

## [1.1] — Unreleased

### Added

- **Sub Graph Node** — a new node type that delegates execution to another `DialogueGraph` asset, then returns and continues from the next node — exactly like a function call. Assign any graph asset and an optional entry point key directly on the node. Enables shared dialogue libraries: create a single "GenericGuardIdle" graph and reference it from any number of NPC-specific router graphs without duplicating content.
- **Graph stack with depth guard** — the runtime maintains a call stack when sub-graphs are nested. The stack supports up to 16 levels of nesting; exceeding this logs an error and ends dialogue cleanly to prevent infinite loops.
- **Caller speaker fallback** — NPC nodes and all speaker-dependent actions (camera look-at, animator triggers, positional audio) now resolve speaker name through a three-level hierarchy: **node speaker → graph `DefaultSpeakerName` → calling actor's speaker name**. Shared graphs with blank speaker fields automatically display and look at whoever triggered the dialogue — no extra setup required.
- **End Node: "On end →" sub-graph slot** — the End node now has an optional `DialogueGraph` object field and entry point key. When a graph asset is assigned, that graph runs as a sub-routine before the conversation truly ends. Useful for routing important NPCs back to generic idle lines at the conclusion of a quest conversation. The `Next entry:` switch fires first so the actor's state is already updated while the sub-graph plays.
- **`SpeakerName` on `IDialogueActor`** — the interface now exposes a `SpeakerName` property, implemented by both `NPCDialogue` and `DialogueTrigger`. This is what powers the caller fallback above and allows any custom actor to participate in speaker resolution without extra wiring.
- **Bark system** — fire-and-forget ambient lines that play without entering a full conversation and without blocking the player. Mark any `DialogueGraph` as a bark graph via the new **Is Bark Graph** toggle in the GRAPH sidebar. Bark graphs support NPC, Random, Branch, Set Variable, and End nodes — meaning barks can be condition-driven and randomised using the same graph tooling as normal conversations. `DialogueManager.PlayBark(graph, transform)` runs the graph on a second non-blocking runner and fires `OnBark` (a separate event from `OnNPCLine`) for each line so bark output never reaches the main dialogue panel. The new `BarkSource` component handles trigger modes (On Enter trigger collider, On Timer, Manual), cooldown, and automatic suppression during active conversations.
- **Bark validator check** — the graph validator now warns when a bark graph contains a Player Choice node, which would never be reached in a non-blocking bark flow.

### How to use Sub Graphs

**Unimportant NPC (no unique dialogue):** Assign your shared graph directly to the NPC's `NPCDialogue` component. Their own `Speaker Name` field is used as the fallback, so the UI and camera resolve to the correct individual automatically.

**Important NPC (quest-driven):** Give them a personal "router" graph containing only a Branch or Entry Point chain — no dialogue lines. Each branch leads to a Sub Graph Node pointing at the relevant asset (generic idle, quest intro, quest done, etc.). Your quest system calls `npc.SetEntryPoint("QuestDone")` to switch routing; the router graph handles the rest.

**End-of-quest fallback:** On the End node of a quest-specific graph, set `On end →` to your generic idle graph. After the unique lines finish, the player hears the NPC's normal idle response before dialogue closes — all without duplicating a single line.

---

## [1.0.1] — Unreleased

### Added

- **Cancel dialogue at any time** — players can now press Escape to exit a conversation immediately. `DialogueManager.CancelDialogue()` is the backing API for custom UIs. Per-node **Prevent Dialogue Exit** toggle (visible on every node in the graph editor, below the ⚐ Tag field) blocks cancellation while that node is active — use this to protect sections that write variables or grant rewards.
- **Show GUIDs toggle** (NAVIGATE sidebar) — displays the full node GUID in small text at the bottom of every node. Toggle persists across editor sessions via EditorPrefs.
- **Search GUID field** (NAVIGATE sidebar) — paste a full GUID or the 8-char prefix from an error message and press Go/Enter to jump directly to that node.
- **Copy GUID** (node right-click context menu) — copies the node's full GUID to the clipboard.
- **Zoom range expanded** — canvas now zooms from 0.05× to 5× (previously 0.15× to 3×).

### Improved

- **Richer runtime error messages** — all `Debug.LogError` / `Debug.LogWarning` calls now include the graph name, node type, and short GUID prefix. Clicking the console entry in the Project window pings the source graph asset.
- **Animator lookup cached at registration** — `DialogueManager` now caches each speaker's `Animator` component when `RegisterSpeaker` is called (once at scene start). Animator actions no longer walk the NPC's child hierarchy on every dialogue node.
- **Eliminated list allocations in `RunRandomNode` and `RunPlayAudioNode`** — both methods previously allocated temporary lists at runtime (via `.Where().ToList()` and `FindAll()`) just to read a count or pick an index. Replaced with direct iteration — no heap allocation.
- **Typewriter uses `StringBuilder`** — replaced `text.Substring(0, i+1)` with an appending `StringBuilder`. Previously allocated a new string every character reveal; now appends one character per tick.
- **Help & Documentation opens online docs** — the in-editor help window (1300+ lines of duplicated content) has been replaced with a direct link to the online documentation. `Threader > Help & Documentation` now opens the docs site in the default browser.
- **Graph Validator** — new checks added:
    - Jump node has no target tag set.
    - Jump node targets a tag that doesn't exist on any node in the graph.
    - NPC node references a speaker name not found in any Speaker Roster asset.
    - End node `Next entry` key references an entry point not defined on the graph.
    - Choice errors now report the choice index (e.g. "choice 2 has blank text").
- **Live validation** — the validator panel now re-runs automatically whenever nodes or edges change, as long as the panel is open. The Run button always re-opens and refreshes the panel; the ✕ button closes it.

### Added

- **Skip / Exit HUD elements** — `DialogueUI` now queries optional `"Skip"` and `"Exit"` named elements from the UXML and manages their visibility automatically. Skip is shown during NPC lines and hidden during choice nodes. Exit is shown whenever `DialogueManager.CanCancel` is true (dialogue active and current node does not have **Prevent Dialogue Exit** set).

### Improved

- **`DialogueUI` is now event-driven** — previously `DialogueManager` called `DialogueUI` static methods directly (`SetText`, `ShowChoices`, `HideUI`). `DialogueUI` now subscribes to `OnNPCLine`, `OnChoiceNode`, and `OnDialogueEnd` in `Start()` and drives itself through those events — exactly as documented in the custom UI guide. The manager no longer holds a direct dependency on `DialogueUI` for display logic.

### Fixed

- Show GUIDs state was lost when loading a different graph — now re-applied after every `LoadGraph` call.
- `DialogueLogContext` GUID stash was never cleared if no double-click followed a log — now auto-clears after the next log message fires.
- **Pressing Space to skip a line did not stop the audio clip** — the clip always played to completion before the skip was honoured. The `WaitWhile` audio waits now exit immediately when a skip is requested, and the audio source is stopped before moving to the next line.
- **`StartDialogue` called while dialogue already active** — previously would silently stomp the running conversation. Now logs a warning and returns early.
- **`_cancelLocked` state not reset on `Awake`** — if `DialogueManager` persisted across scenes, the cancel lock could carry stale state. Now explicitly reset to `false` in `Awake`.
- **Validator false positive — 'Prevent Dialogue Exit' with Jump Node** — the reachability walk did not follow Jump nodes, causing false "no End node reachable" warnings on graphs that route through a jump to a branch with an End node. Jump targets are now resolved correctly during traversal.
