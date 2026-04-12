# Roadmap

This page tracks what's coming to Threader, what's being considered, and what's intentionally out of scope. It's updated as plans change.

If you have a feature request, raise it on the [GitHub repository](https://github.com/StefanVorster17/Threader-Docs/issues) — community requests that get traction move up into a planned release.

---

## Upcoming

### v1.1

Sub-graph support, bark system, and editor quality-of-life. See the [Changelog](changelog.md) for the full list on release.

#### Sub-Graph support

Call any `DialogueGraph` asset from within a running dialogue graph, then return to the calling graph when it ends.

**What ships:**

- **Sub Graph Node** — a new node type that delegates execution to another graph asset. Wire it like any other node; assign a target graph and an optional entry point key. The conversation resumes on the connected output when the sub-graph reaches its End node.

- **End Node sub-graph slot** — an optional graph reference on the End node. When set, that graph runs as a sub-routine before the conversation truly closes — useful for routing an NPC back to generic idle lines after a quest-specific conversation.

- **Caller speaker fallback** — NPC nodes in shared graphs resolve speaker name through a three-level chain: node → graph default → calling actor. Shared graphs with blank speaker fields automatically display and look at whoever triggered the dialogue.

- **Graph call stack with depth guard** — the runtime tracks nested sub-graph calls up to 16 levels deep and ends dialogue cleanly if the limit is exceeded.

- Existing graphs are completely unaffected — this is additive, nothing breaks.

**Use cases this enables:**
- Shared "rumour" or "ambient" dialogue that any NPC can invoke mid-conversation
- Reusable shop, trade, or minigame dialogue sequences called from multiple graphs
- Breaking very large story graphs into smaller chapter graphs that chain together
- Quest NPCs that fall back to generic idle lines automatically when the quest conversation ends

#### Bark system

Fire-and-forget ambient lines that play without entering a full conversation — an NPC mutters as you walk past, a guard reacts to a sound, a merchant calls out. Barks run in parallel to the world; they never block the player or trigger `OnDialogueStarted`.

**What ships:**

- **`IsBark` flag on `DialogueGraph`** — mark any graph as a bark graph. The validator enforces that bark graphs contain no Player Choice nodes (which would be unreachable in a non-blocking flow).

- **`BarkSource` component** — attach to any NPC alongside `NPCDialogue`. Holds a bark graph reference, a cooldown timer, and a trigger mode (`OnEnter`, `OnTimer`, `Manual`). Calls `DialogueManager.PlayBark()` automatically or on demand.

- **Non-blocking bark runner** — a second lightweight runner path in `DialogueManager` that traverses NPC, Random, Branch, Set Variable, and End nodes from a bark graph and fires `OnBark` instead of `OnNPCLine`. The main conversation runner is completely unaffected.

- **`OnBark` event** — a separate `Action<NPCLine>` event so bark output is handled independently of the main dialogue panel. Wire it to a floating world-space speech bubble, a HUD ticker, or nothing at all.

- Barks are suppressed automatically while a full conversation is active (configurable).

**Use cases this enables:**
- Ambient NPC chatter as the player moves through the world
- Guard alert or suspicion reactions without interrupting gameplay
- Merchant and shopkeeper callouts
- Randomised flavour lines driven by game state via Branch and condition nodes

**NPC-to-NPC / overheard dialogue** — because the bark runner waits for each audio clip to finish before the next line plays, a bark graph can script a full back-and-forth exchange between two NPCs (e.g. two guards talking as the player walks past). Wire `OnBark` to read the `SpeakerName` field and route each line to the right overhead bubble. No extra code or special mode needed.

---

## Considering

Features on the radar but not yet committed to a version. These may or may not ship depending on scope, community interest, and architectural fit.

### Localization support

Allow all dialogue text to be translated and swapped at runtime by language.

**Two approaches are being considered:**

**Option A — Built-in (no dependency)**
Threader ships its own `LocalizationTable` ScriptableObject: a simple line ID → translated string mapping. Swap the active table at runtime and all dialogue text updates automatically. No external packages required — works for everyone out of the box.

**Option B — Unity Localization package integration**
Threader reads from `LocalizedString` fields via Unity's official Localization package. Buyers who already use it get seamless integration with their existing workflow.

The most likely approach is to ship both: Option A as the default with zero setup friction, and Option B as an optional bridge assembly compiled only when the Unity Localization package is detected (`#if` conditional). This way existing buyers are never affected and no unwanted dependencies are introduced.

**Status:** Under consideration. Dependency strategy needs to be resolved before implementation begins.

---

## Out of scope

These are features that have been considered and deliberately won't be added to Threader core. If you need them, alternatives are noted.

### Built-in branching story format export (Ink, Yarn, Twine)

Threader's graph format is Unity-native and optimised for runtime execution, not text-based story formats. Converting between the two would require lossy translation of variable scopes, entry points, and events. Use Yarn Spinner or ink directly if you need a text-based authoring workflow.

### Multiplayer / networked dialogue synchronisation

Syncing dialogue state across a network is highly game-specific (what to sync, who is the authority, how to handle latency). Threader's runtime is single-player by design. Wire your own netcode layer on top of `OnNPCLine`, `OnChoiceNode`, and `SelectChoice` if you need multiplayer dialogue.
