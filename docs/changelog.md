# Changelog

---

## [Unreleased]

---

## [1.0.0] — Initial Release

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
- Dialogue Preview Window (`Shift+Alt+P`) — step through any graph in the editor without entering Play mode, seed variable values, see condition evaluation live

**Node Types**
- **Start Node** — default entry point of every graph
- **NPC Node** — multiple lines per node, optional audio clips, speaker assignment, node events (local + global), Set Variable actions (Set/Add/Subtract/Toggle), Animator parameter actions (Trigger/Bool/Int/Float)
- **Player Choice Node** — up to any number of choices per node; inline variable conditions (Equal/NotEqual/GreaterThan/GreaterOrEqual/LessThan/LessOrEqual); Hide and NOT per condition row; custom ConditionDefinition slot with Negate toggle; AND logic across all rows
- **End Node** — closes the conversation; optional `Set Entry Point On Complete` for auto-switching
- **Branch Node** — evaluates conditions, routes to True or False output port
- **Set Variable Node** — standalone variable modification (same operators as NPC node)
- **Random Node** — routes execution to one of its outputs with equal probability
- **Jump Node** — teleports execution to any node tagged with a Return Tag; enables loops without spaghetti wires
- **Debug Node** — logs a message at Info/Warning/Error level; optional Watch Variables list printed alongside the message
- **Wait Node** — pauses execution for a configurable number of seconds (coroutine; skipped in preview window)
- **Fire Event Node** — fires a named string event; local or global
- **Play Audio Node** — plays an AudioClip at a speaker's position (spatial) or 2D fallback; optional Advance Immediately
- **Animator Trigger Node** — sets any Animator parameter on any registered speaker by name

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
- `IDialogueActor` interface — `SetEntryPoint`, `ResetEntryPoint`, `ActiveEntryPointKey`, `StartDialogue`, `Graph`
- `DialogueGraph.ResolveEntryPointGuid` — safe fallback to start node with console warning
- Auto-switching via End node `Set Entry Point On Complete` field

**Components**
- `DialogueTrigger` — trigger-collider driven NPC; 3D and 2D physics; interact prompt; configurable key; Start On Enter mode; full `IDialogueActor` implementation
- `NPCDialogue` — code-driven NPC; `IsInteractable` flag; per-key `NodeEventResponse` list with Inspector-wired `UnityEvent`; `IDialogueActor` implementation
- `DialogueManager` — singleton; `OnNPCLine`, `OnNodeEvent`, `OnGlobalNodeEvent`, `OnChoiceNode`, `OnDialogueEnd`, `OnDialogueStartedEvent` C# events; `OnDialogueStarted`/`OnDialogueEnded` Unity Events; `RegisterSpeaker`/`UnregisterSpeaker`; `RegisterBlockable`/`UnregisterBlockable`; spatial + 2D audio routing

**Interfaces**
- `IDialogue` — `Block()`/`Unblock()` for movement/input controllers
- `IDialogueFocus` — `FocusOn(Transform)`/`ReleaseFocus()` for camera rigs
- `IDialogueReady` — `IsDialogueReady` gate for transition animations

**Choice History**
- `DialogueChoiceHistory` static class — `MarkVisited` (auto), `IsVisited`, `GetSaveData` / `LoadSaveData`, `Clear`
- Key format: `"{choiceNodeGuid}:{choiceIndex}"` — stable across sessions if graph structure doesn't change

**In-editor Help Window**
- Built-in documentation panel (Window → Threader → Help / `Shift+Alt+H`)
- Tabs: Quick Start, Graph Editor, Node Reference, Variables, Conditions, Entry Points, Saving, API Reference

---
