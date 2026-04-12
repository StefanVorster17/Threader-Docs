# Threader - Dialogue Kit

**Visual Dialogue System for Unity** — build branching conversations with a node-based graph editor, no boilerplate required.

<span style="background:#2d6a9f;color:#fff;padding:3px 10px;border-radius:4px;font-size:0.85em;font-weight:bold;">v1.0 (Unreleased)</span>

![Threader graph editor](assets/images/hero-graph.png){ width="720" }

---

## Features

- **Visual graph editor** — drag, connect, and organise nodes on an infinite canvas
- **12 node types** — NPC lines, player choices, branching, variables, audio, animator triggers, events, waits, and more
- **Built-in variable store** — Bool / Int / String variables with inline conditions on choices — no C# needed
- **Text token substitution** — embed variable values directly in dialogue: `You have {gold} {gold:name}`
- **Entry points** — jump into any branch of a graph based on story state
- **Spatial audio** — per-speaker 3D audio source positioning
- **Animator integration** — set parameters on any registered speaker's Animator
- **Custom conditions** — register C# delegates or a `ConditionProvider` asset for complex game state
- **Edit-mode preview window** — step through any graph without entering Play mode
- **Choice history** — track visited choices with save/load serialization
- **Singleton `DialogueManager`** — one component drives everything; subscribe to events from any script

---

## Minimum setup

Five steps to a working dialogue:

1. **Create** a `Dialogue Graph` asset — right-click in the Project window → **Create → Threader → Dialogue Graph**
2. **Build** the conversation — double-click the asset to open the graph editor
3. **Add** a `DialogueManager` component to an empty GameObject in the scene
4. **Call** `DialogueManager.Instance.StartDialogue(graph)` from your interaction code
5. **Wire** the UI — add `DialogueUI` to a GameObject, or subscribe to `OnNPCLine` / `OnChoiceNode` for a fully custom UI

→ See [Quick Start](quick-start.md) for the full walkthrough.

---

## Navigation

| Section | What's in it |
|---|---|
| [Quick Start](quick-start.md) | End-to-end setup in ~10 minutes |
| [Tutorial](tutorial.md) | Guided walkthrough building a complete dialogue scene |
| [Graph Editor](graph-editor.md) | Canvas controls, sidebar, shortcuts, tools |
| [Dialogue Preview](preview-window.md) | Step through graphs in edit mode without entering Play mode |
| [Node Reference](nodes.md) | All 12 node types with field descriptions |
| [Variables](variables.md) | Variable store, set actions, text tokens |
| [Conditions](conditions.md) | Inline conditions and C# custom conditions |
| [Events](events.md) | Node events, global events, and C# event subscriptions |
| [Speaker Roster](speaker-roster.md) | Speaker name assets, NPCDialogue setup, and name resolution |
| [UI](ui.md) | Built-in DialogueUI, Canvas/uGUI custom UI, and event hooks |
| [Entry Points](entry-points.md) | Multi-branch graphs and state persistence |
| [Saving](saving.md) | Persisting variables, choices, and entry points |
| [API Reference](api-reference.md) | Events, interfaces, and code API |
| [Troubleshooting](troubleshooting.md) | Solutions to common problems and how to report bugs |
| [Roadmap](roadmap.md) | What's coming in upcoming releases |
| [Changelog](changelog.md) | Full version history |
