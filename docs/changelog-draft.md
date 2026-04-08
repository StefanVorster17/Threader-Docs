# Changelog Draft

*Internal working doc — not in nav. Copy entries to changelog.md before release.*

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
