# Changelog

---

## [1.0.0] — April 16, 2026

### Core Systems

**Graph Editor**
- Node-based visual graph editor (`DialogueGraphEditorWindow`)
- Canvas pan (middle-mouse / Alt+LMB), zoom (scroll wheel), multi-select, box-select
- Sidebar panels: PROJECT (open graphs), GRAPH (settings), NAVIGATE (jump to nodes), CREATE (spawn nodes)
- Find & Replace across all node text in the open graph
- Copy/paste nodes with Ctrl+C / Ctrl+V; duplicate with Ctrl+D
- Undo/redo support
- Comment / Group boxes for organisation (editor-only, no runtime cost)
- Sticky notes in five colour themes and three sizes (editor-only)
- Speaker Roster assets — speaker fields become type-safe dropdowns
- Per-graph default speaker name and Look At Speaker toggle
- Dialogue Preview Window (`Ctrl+Shift+P`) — step through any graph in the editor without entering Play mode, seed variable values, see condition evaluation live

**Node Types**
- **Start Node** — default entry point of every graph
- **NPC Node** — multiple lines per node, optional audio clips, speaker assignment, node events (local + global), Set Variable actions (Set/Add/Subtract/Toggle), Animator parameter actions (Trigger/Bool/Int/Float)
- **Player Choice Node** — up to any number of choices per node; inline variable conditions (Equal/NotEqual/GreaterThan/GreaterOrEqual/LessThan/LessOrEqual); Hide and NOT per condition row; custom ConditionDefinition slot with Negate toggle; AND logic across all rows
- **End Node** — closes the conversation; optional `Next entry` auto-switching and `On end →` sub-graph slot
- **Branch Node** — evaluates conditions, routes to True or False output port
- **Set Variable Node** — standalone variable modification (same operators as NPC node)
- **Random Node** — routes execution to one of its outputs with equal probability
- **Jump Node** — teleports execution to any node tagged with a Return Tag; enables loops without spaghetti wires
- **Sub Graph Node** — delegates execution to another `DialogueGraph` asset and returns; supports shared libraries and nested call stacks up to 16 levels
- **Debug Node** — logs a message at Info/Warning/Error level; optional Watch Variables list printed alongside the message
- **Wait Node** — pauses execution for a configurable number of seconds (coroutine; skipped in preview window)
- **Fire Event Node** — fires a named string event; local or global
- **Play Audio Node** — plays an AudioClip at a speaker's position (spatial) or 2D fallback
- **Animator Trigger Node** — sets any Animator parameter on any registered speaker by name
- **Switch Node** — evaluates multiple named cases top-to-bottom; the first case whose conditions all pass is followed, otherwise the Default output fires. Uses the same condition system as Branch nodes.
- **Weighted Random Node** — picks one output at random using weighted probability. Higher weights are selected more often. Weights are relative (do not need to sum to any specific value).

**Variables**
- `DialogueVariables` ScriptableObject — Bool, Int, String variables with design-time defaults
- Runtime dictionaries (never serialized back to the asset): `GetBool`/`GetInt`/`GetString`, `SetBool`/`SetInt`/`SetString`
- `InitRuntime()` for explicit reset (new game)
- Multiple assets supported; searched in order by `DialogueManager`
- Inline conditions on Player Choice nodes and Branch nodes read from all assigned assets

**Conditions**
- Inline variable conditions on choices and branch nodes (no C# required)
- `ConditionDefinition` ScriptableObject — Key, DisplayName, Category, ParameterHint, WhenMissing (Allow/Block)
- `ConditionProvider` abstract ScriptableObject — subclass to centralize evaluation logic
- `ConditionService` static class — `Register`/`Unregister` delegates, `SetProvider`, `ClearDelegates`, unified `Evaluate`
- `ConditionStore` static in-memory key/value store for condition state
- `ConditionStoreProvider` built-in provider that evaluates against `ConditionStore`
- `VariableConditionDefinition` — reusable condition asset that self-evaluates against a `DialogueVariables` asset

**Entry Points**
- Per-graph named entry points (right-click → Set as Entry Point); yellow badge in editor
- `IDialogueActor` interface — `SetEntryPoint`, `ResetEntryPoint`, `ActiveEntryPointKey`, `SpeakerName`, `StartDialogue`, `Graph`
- `DialogueGraph.ResolveEntryPointGuid` — safe fallback to start node with console warning
- Auto-switching via End node `Next entry` dropdown and `On end →` sub-graph slot

**Components**
- `DialogueTrigger` — trigger-collider driven NPC; 3D and 2D physics; interact prompt; configurable key; Start On Enter mode; full `IDialogueActor` implementation
- `NPCDialogue` — code-driven NPC; `IsInteractable` flag; per-key `NodeEventResponse` list with Inspector-wired `UnityEvent`; `IDialogueActor` implementation
- `DialogueManager` — singleton; `OnNPCLine`, `OnNodeEvent`, `OnGlobalNodeEvent`, `OnChoiceNode`, `OnDialogueEnd`, `OnDialogueStartedEvent`, `OnBark` C# events; `OnDialogueStarted`/`OnDialogueEnded` Unity Events; `RegisterSpeaker`/`UnregisterSpeaker`; `RegisterBlockable`/`UnregisterBlockable`; `PlayBark()`; spatial + 2D audio routing

**Interfaces**
- `IDialogue` — `Block()`/`Unblock()` for movement/input controllers
- `IDialogueFocus` — `FocusOn(Transform)`/`ReleaseFocus()` for camera rigs
- `IDialogueReady` — `IsDialogueReady` gate for transition animations

**Choice History**
- `DialogueChoiceHistory` static class — `MarkVisited` (auto), `IsVisited`, `GetSaveData` / `LoadSaveData`, `Clear`
- Key format: `"{choiceNodeGuid}:{choiceIndex}"` — stable across sessions if graph structure doesn't change

**In-editor Help Window**
- **Threader → Help & Documentation** opens the online documentation in the default browser (previously a 1300-line in-editor panel)

### Sub-Graph System

- **Sub Graph Node** — a new node type that delegates execution to another `DialogueGraph` asset, then returns and continues — exactly like a function call. Assign any graph asset and an optional entry point key directly on the node.
- **Graph call stack with depth guard** — the runtime supports up to 16 levels of nested sub-graphs and ends dialogue cleanly if the limit is exceeded.
- **Caller speaker fallback** — NPC nodes in shared graphs resolve speaker name through a three-level chain: node speaker → graph `DefaultSpeakerName` → calling actor's speaker name. Shared graphs with blank speaker fields automatically display and look at whoever triggered the dialogue.
- **End Node "On end →" sub-graph slot** — optional graph reference on the End node; that graph runs as a sub-routine before the conversation truly closes. Useful for routing NPCs back to generic idle lines after a quest conversation.
- **`SpeakerName` on `IDialogueActor`** — the interface now exposes a `SpeakerName` property, implemented by `NPCDialogue` and `DialogueTrigger`.

### Line Sheet System

- **`DialogueLineSheet` companion asset** — per-graph asset that stores per-speaker audio clips and animator actions for every NPC line. Multiple speakers can share the same graph and each plays their own voice lines.
- **Line Data button** — NPC nodes in the graph canvas have a **Line Data** button that opens a focused per-node editor (`NodeLineSheetPopup`).
- **Line Sheet Editor window** — whole-graph overview of all NPC nodes and lines; accessible from the graph editor sidebar (PROJECT → Line Sheet Editor).
- **Batch sync** — `Threader → Create & Sync All Line Sheets` creates and syncs sheets for every graph in the project at once.
- **Multi-language support** — `DialogueGraph` has a `LineSheets` list (`List<NamedLineSheet>`) allowing one line sheet per language. `DialogueManager.SetActiveLanguage(string)` selects the active language at runtime. The active sheet's `PreviewText` fields override NPC line text and choice text for localization. `DialogueManager.ActiveLanguage` reads the current language.
- **Language Library** — optional `LanguageLibrary` ScriptableObject (**Create → Threader → Language Library**) defines project-wide languages in a single asset. Assign to `DialogueManager` to auto-populate language slots on every graph Inspector with read-only labels instead of free-text entry.
- **Choice text localization** — `ChoiceSheetRow` class stores localized choice text per Player Choice node per language in the line sheet.
- **Redesigned `DialogueGraph` Inspector** — dark header panel, stats row (nodes, NPC lines, entry points, groups), a "Used By" section showing all scene objects referencing the graph, and an editable Line Sheet field.
- **Redesigned `DialogueLineSheet` Inspector** — stats row (lines, speakers, clips, orphaned rows), description, read-only source graph reference, and an **Open Line Sheet Editor** button.

### Bark System

- **`IsBark` flag on `DialogueGraph`** — mark any graph as a bark graph via the **Graph Type** dropdown in the GRAPH sidebar.
- **`BarkSource` component** — attach to any NPC. Holds a bark graph reference, cooldown, and trigger mode (`OnEnter`, `OnTimer`, `Manual`). Calls `PlayBark()` automatically or on demand. Suppresses barks automatically during active conversations.
- **`BarkSource` Speaker Name field** — a roster dropdown on `BarkSource` that feeds the third-level speaker resolution fallback for bark graphs. Allows a shared bark graph with no speaker set on nodes to resolve the correct voice clips and animator actions per NPC.
- **`DialogueManager.PlayBark(graph, transform, speakerName)`** — runs a bark graph on a second non-blocking runner; fires `OnBark` for each line instead of `OnNPCLine`. Optional `speakerName` parameter provides the third-level speaker fallback; `BarkSource` passes its Speaker Name automatically.
- **`OnBark` event** — separate `Action<NPCLine>` event for bark output (wire to a floating speech bubble, HUD ticker, or nothing).
- **Bark graph validator** — warns when a bark graph contains a Player Choice node.
- **Bark runner** — skips Wait nodes automatically; NPC, Random, Branch, Set Variable, and End nodes are all supported.
- **NPC-to-NPC / overheard dialogue** — because the bark runner waits for each audio clip to finish before advancing, sequential NPC exchanges can be scripted as a bark graph with no extra code.

### Node Templates

- **`DialogueNodeTemplate` ScriptableObject** — stores a deep-copy snapshot of one or more nodes and their internal connections. Created automatically when saving a selection, or via `Assets → Create → Threader → Dialogue Node Template`.
- **Save Selection as Template** — select nodes and click "Save Selection" in the NODE TEMPLATES sidebar panel. Internal connections are preserved; external connections are dropped.
- **Drag-to-stamp** — drag a template pill from the NODE TEMPLATES sidebar onto the canvas to instantiate it with fresh GUIDs and re-connected internal wiring.
- **Project window drag** — dragging a `.asset` directly from Unity's Project window onto the canvas also works.
- **NODE TEMPLATES sidebar panel** — one pill per discovered template asset showing name and node count `[N]`. Refresh button re-scans the project.
- **Rename and Delete** — right-click any template pill to rename (updates display name and `.asset` filename) or delete with a confirmation dialog.

### Bookmarks

- **Bookmark this Node** — right-click any node to add it to the BOOKMARKS sidebar panel.
- Click a bookmark row to select and frame that node on the canvas.
- **Custom bookmark names** — click the ✎ pencil button on a row to rename it inline. Press Enter or click away to confirm; Escape to cancel.
- Bookmarks are removed automatically when the bookmarked node is deleted.
- Persisted per-graph in `EditorPrefs` (keyed by graph asset GUID); survive editor restarts.

### Export Script

- **Export Script** button in the PROJECT sidebar — walks the graph from the start node in execution order and writes a plain-text `.txt` screenplay file.
- Output: `[Speaker Name]` / dialogue text / blank line. Player choices are numbered; each branch is indented below its option. Branch nodes show `(True path)` / `(False path)`. Silent nodes are invisible in the output.
- Nodes unreachable from the start node are appended at the bottom with a `--- (unreachable from start) ---` separator.
- File opens in Explorer/Finder immediately after export.

### Editor / UX

- **Graph Type dropdown** — replaces the old "Is Bark Graph" toggle. Options: `Dialogue` / `Bark`. Bark graphs automatically hide Choice Node, Wait Node, and Sub Graph Node pills in the sidebar and context menu; Look At Speaker row is also hidden.
- **Sidebar default width doubled** — 312 px default (was 156 px), 240 px minimum (was 120 px). Stored in `EditorPrefs`.
- **Custom right-click context menu** — styled dark panel grouped to match the sidebar: Dialogue / Logic / Data & Events / Utility / Selection. Bark-incompatible nodes hidden automatically when `IsBark` is true.
- **Jump node right-click → "Go to Target (tagname)"** — selects and frames the node that owns the jump's target tag.
- **End node right-click → "Go to Entry Point (keyname)"** — selects and frames the node that owns the entry point key.
- **Sidebar order** — PROJECT → GRAPH → NAVIGATE → BOOKMARKS → CREATE → NODE TEMPLATES. "Referenced by" moved into the PROJECT collapsible.
- Sidebar node pills and action pills are now **drag-only** — click-to-create removed.
- **GUID label** on nodes now appears above the Tag field.
- **Colored console log prefixes** — `[<color=cyan>Tag</color>]` for logs, yellow for warnings, red for errors. Applies to DialogueManager, DialogueRunner, DialogueGraph, DialogueTrigger, DialogueVariables, NPCDialogue, BarkSource, ConditionService, ConditionStore, and DebugNode.
- `BaseDialogueNodeView` exposes a `protected virtual AppendNodeContextMenu()` hook for subclass context menu extensions.
- **Zoom range** expanded to 0.05× – 5× (was 0.15× – 3×).

### Variable System

- **Type-aware `DialogueVariables` inspector** — shows only the relevant value field per variable: Bool → checkbox, Int → integer field, String → text field. Clean collapsible rows with remove button and `+ Add Variable` button.
- **Type-aware condition rows** — value field on Branch and Choice conditions now matches the variable type automatically. Bool → True/False dropdown; Int → IntegerField; String → TextField. Operators filtered by type (numeric ops hidden for Bool/String).
- **Type-aware Set Variable rows** — same type-aware value field in NPC node set-vars and Set Variable nodes. Operator dropdown hides `Add`/`Subtract` for non-Int and `Toggle` for non-Bool. Value field hidden entirely when operator is Toggle.
- Changing the variable name in any row immediately rebuilds the field to match the new type.

### Cancel & Safety

- **Cancel dialogue at any time** — press Escape to exit a conversation. `DialogueManager.CancelDialogue()` is the backing API.
- **Prevent Dialogue Exit** toggle — per-node toggle (below the ⚐ Tag field) to block cancellation while that node is active.
- **`CanCancel` property** — `true` while dialogue is active and the current node does not have Prevent Dialogue Exit set.
- **Skip / Exit HUD elements** — `DialogueUI` automatically manages optional `"Skip"` and `"Exit"` named UXML elements.

### GUID Tools

- **Show GUIDs toggle** (NAVIGATE sidebar) — displays the full node GUID above the Tag field. Persists via EditorPrefs.
- **Search GUID field** (NAVIGATE sidebar) — paste a GUID or 8-char prefix and press Go/Enter to jump directly to that node.
- **Copy GUID** (node right-click) — copies the full GUID to the clipboard.

### Graph Validator Improvements

- Jump node has no target tag set.
- Jump node targets a tag that doesn't exist in the graph.
- NPC node references a speaker name not in any Speaker Roster asset.
- End node `Next entry` key not defined on the graph.
- Choice errors now report the choice index (e.g. "choice 2 has blank text").
- **Reachability walk** now correctly follows Jump nodes (was producing false "no End node reachable" warnings).
- **Live validation** — the validator re-runs automatically whenever nodes or edges change while the panel is open.

### Bug Fixes

- `CloneNode` for NPCNode silently dropped `SpeakerName`, `SetActions`, and `AnimatorActions` on copy/paste, duplicate, and template save — all three are now correctly copied.
- Pressing Space to skip a line did not stop the active audio clip — the clip now stops immediately when skip is requested.
- `StartDialogue` called while dialogue already active now logs a warning and returns early instead of stomping the running conversation.
- `_cancelLocked` state not reset on `Awake` — could carry stale state across scenes; now explicitly reset to `false`.
- Show GUIDs state was lost when loading a different graph — now re-applied after every `LoadGraph` call.
- Snap to Grid now takes effect immediately on drag-end (was only rounding positions at save time).
- Nodes could not be removed from a Group box — **Remove from Group** is now available directly in the node right-click menu.
- Set Colour / Tag is now available directly in the node right-click menu.

### Performance

- Animator component cached at `RegisterSpeaker` time — no longer walked on every dialogue node.
- Eliminated temporary list allocations in `RunRandomNode` and `RunPlayAudioNode`.
- Typewriter now uses `StringBuilder` — no per-character string allocation.

---
