# Quick Start

Get a working dialogue running in about 10 minutes. Follow these steps in order — every step builds on the last.

> **Already done this?** Skip straight to the [Tutorial](tutorial.md) to build a full quest-driven NPC with variables, conditions, events, and entry points.

---

## Before you begin

Make sure you have **imported the Threader package** into your Unity project. You should see a `Threader` folder in your Assets directory containing `Scripts`, `Editor`, and `UI` subfolders. If not, import the package first.

---

## Step 1 — Create a Speaker Roster

A [Speaker Roster](speaker-roster.md) is a small asset that declares the speaker names available in your project. It populates the speaker-name dropdowns throughout the graph editor so you pick from a list instead of typing names manually.

1. In the **Project** window, right-click → **Create → Threader → Speaker Roster**.
2. Name it something like `MyRoster`.
3. Select the asset. In the **Inspector**, click **+** and type your first speaker name — for example `Villager`.
4. Add more names if you already know them (e.g. `Guard`, `Merchant`). You can always add more later.

> Speaker names are **case-sensitive**. `Villager` and `villager` are treated as different speakers. Pick a convention and stick with it.

---

## Step 2 — Create a Variables Store (optional but recommended)

If your dialogue needs to check or change game state (e.g. "has the player found the key?"), create a [Variables](variables.md) asset now. If you only need a simple linear conversation, you can skip this step and come back later.

1. Right-click in the Project window → **Create → Threader → Variables Store**.
2. Name it `GameVariables` (or whatever suits your project).
3. In the Inspector, click **+ Add Variable** for each piece of state you need:

| Field | What to set |
|---|---|
| **Name** | The key you will reference everywhere — in the graph, in `{token}` text substitution, and in code. Case-sensitive. |
| **Type** | `Bool`, `Int`, or `String` |
| **Value** | The default starting value (e.g. `false`, `0`, or an empty string) |

You do not need to define all variables up front. You can add more at any time and they will appear in the graph editor dropdowns immediately.

---

## Step 3 — Create a Dialogue Graph asset

This is the asset that stores your entire conversation — all nodes, connections, and settings.

1. In the **Project** window, right-click → **Create → Threader → Dialogue Graph**.
2. Name it something descriptive, like `VillagerGraph`.

You now have a `.asset` file in your Project window. This single file holds the full node graph for one conversation (or one bark sequence).

---

## Step 4 — Build the conversation in the Graph Editor

Double-click the graph asset to open the **[Graph Editor](graph-editor.md)**. You can also open it via the menu: **Threader → Graph Editor**.

You will see a dark canvas with a left sidebar. The sidebar has collapsible sections: **PROJECT**, **LANGUAGE**, **GRAPH**, **BOOKMARKS**, **CREATE**, and **NODE TEMPLATES**.

### 4a — Set up the graph

In the **GRAPH** sidebar section:

| Field | What to set |
|---|---|
| **Default Speaker** | Select a speaker from the dropdown (populated from your Speaker Roster). This is the fallback name used by any NPC node whose own Speaker field is left blank. |
| **Graph Type** | Leave as **Dialogue** for a normal conversation. Change to **Bark** only if you are building ambient fire-and-forget lines (see [Bark System](bark.md)). |
| **Look At Speaker** | Leave ticked if you want the camera to rotate towards the active speaker automatically. Untick if your game does not use this. |

### 4b — Add nodes

Use keyboard shortcuts or drag pills from the **CREATE** sidebar panel:

| Shortcut | Node | What it does |
|---|---|---|
| `N` | **NPC Node** | One or more spoken lines from a character |
| `C` | **Player Choice Node** | A set of clickable choices for the player |
| `E` | **End Node** | Closes the conversation |
| `B` | **Branch Node** | Routes to True or False output based on variable conditions |
| `V` | **Set Variable Node** | Writes variables silently without showing dialogue |
| `R` | **Random Node** | Picks a random output port |
| `W` | **Weighted Random Node** | Picks a random output with configurable weights |
| `S` | **Switch Node** | Multi-case routing based on a variable value |
| `J` | **Jump Node** | Redirects to any tagged node in the graph |
| `D` | **Debug Node** | Logs a message to the Console |
| `F2` | **Fire Event Node** | Broadcasts named events to your game code |
| `F3` | **Play Audio Node** | Plays an AudioClip |
| `F4` | **Animator Trigger Node** | Sets Animator parameters on a registered speaker |
| `F5` | **Wait Node** | Pauses for a number of seconds before advancing |
| `F6` | **Sub Graph Node** | Calls another Dialogue Graph and returns when it ends |

See [Node Reference](nodes.md) for the full field descriptions of every node type.

### 4c — Build a minimal conversation

For your first graph, keep it simple:

1. **Press `N`** to create an NPC node. Click it to select it.
2. In the node view, type a line of dialogue — for example: `Hello, traveller! Welcome to the village.`
3. **Press `C`** to create a Player Choice node.
4. Click **+ Add Choice** twice to add two choices. Type text for each — e.g. `Tell me about this place.` and `I should get going.`
5. **Press `N`** twice to create two more NPC nodes — one response for each choice.
6. **Press `E`** twice to create two End nodes.

### 4d — Connect the nodes

**Drag from the output port** (small circle on the right side of a node) **to the input port** (small circle on the left side of another node) to create a connection.

Wire them like this:

```
NPC Node (greeting)
  └─→ Player Choice Node
        ├─ Choice 0 ─→ NPC Node (response A) ─→ End Node
        └─ Choice 1 ─→ NPC Node (response B) ─→ End Node
```

### 4e — Set the start node

**Right-click** the first NPC node (the greeting) → **Set as Start Node**. A green **▶ START** badge appears on it. This is where the conversation begins.

### 4f — Save

Press **Ctrl+S** to save the graph. The graph is saved inside the `.asset` file — there is no separate save step.

---

## Step 5 — Add a DialogueManager to the scene

The `DialogueManager` is the central runtime component. It drives conversations, fires events, manages audio, and coordinates everything. You need exactly **one** in your scene.

1. In your scene Hierarchy, create an empty GameObject: **GameObject → Create Empty**.
2. Name it `DialogueManager`.
3. **Add Component → DialogueManager** (search for "DialogueManager" in the component menu).

### Configure the DialogueManager Inspector

| Field | What to assign | Required? |
|---|---|---|
| **Speaker Rosters** | Drag your `MyRoster` Speaker Roster asset here. Click **+** to add more if you have multiple rosters. | Optional but strongly recommended — without it, speaker dropdowns in the graph editor are empty. |
| **Variables List** | Drag your `GameVariables` Variables Store asset here. Click **+** to add more if you split variables across multiple assets. | Only required if your graph uses variables, conditions, or `{token}` text substitution. |
| **Condition Provider** | Drag a `ConditionProvider` asset here if you use C# custom conditions. | Only if you use custom conditions. See [Conditions](conditions.md). |
| **Audio Source** | An `AudioSource` for 2D (non-spatial) voice lines. | Optional — one is created automatically at runtime if you leave this empty. |
| **Line Pause** | Seconds to wait after a line finishes before auto-advancing. Default: `1`. | Optional — adjust to taste. |

> **Singleton behaviour:** `DialogueManager` sets itself as `DialogueManager.Instance` in `Awake()`. If a second `DialogueManager` exists in the scene, it destroys itself automatically. You only ever need one.

---

## Step 6 — Add the NPC to the scene

You need a way for the player to start the conversation. Threader provides two components — pick the one that fits your project:

### Option A — NPCDialogue (recommended)

`NPCDialogue` registers the NPC as a named speaker in the dialogue system. This enables camera look-at, spatial audio, and Animator integration. Use this when the NPC is a visible character in the scene.

1. Select (or create) the NPC's GameObject.
2. **Add Component → NPCDialogue**.
3. Fill in the Inspector fields:

| Field | What to set | Notes |
|---|---|---|
| **Graph** | Drag your `VillagerGraph` asset here. | This is the conversation this NPC will run. |
| **Speaker Name** | Select from the dropdown — pick the same name you used in the graph's NPC nodes (e.g. `Villager`). | **Must match exactly.** This is how the runtime resolves which transform to point the camera at, which Animator to use, and which [Line Sheet](line-sheet.md) speaker entry to use for audio clips. |
| **Is Interactable** | Leave ticked. | Uncheck only for secondary speakers who appear in someone else's conversation but never initiate dialogue themselves. |
| **Node Events** | Leave empty for now. | Used for wiring Inspector-based event responses without code. See [Events](events.md). |

Then start the dialogue from your interaction code — a raycast hit, a UI button, a trigger zone, or whatever your game uses:

```csharp
// Example: on raycast hit
NPCDialogue npc = hit.collider.GetComponent<NPCDialogue>();
if (npc != null && npc.IsInteractable)
    npc.StartDialogue();
```

`NPCDialogue.StartDialogue()` calls `DialogueManager.Instance.StartDialogue(graph, entryPointKey, this)` internally, passing itself as the actor so the system knows which NPC is speaking.

> **Automatic speaker registration:** When `NPCDialogue` starts, it automatically calls `DialogueManager.RegisterSpeaker(speakerName, transform)` behind the scenes. You do **not** need to register speakers manually.

### Option B — DialogueTrigger (no code required)

`DialogueTrigger` starts dialogue when the player walks into a trigger zone and presses a key (or enters the zone, depending on settings). It does **not** register a speaker name, so camera look-at and spatial audio are not available with this component.

1. Select (or create) the NPC's GameObject.
2. **Add Component → DialogueTrigger**.
3. Fill in the Inspector fields:

| Field | What to set | Default | Notes |
|---|---|---|---|
| **Graph** | Drag your `VillagerGraph` asset here. | — | Required. |
| **Player Tag** | The tag on your Player GameObject. | `"Player"` | Must match your Player's tag exactly. The trigger uses this to detect the player entering the collider. |
| **Interact Key** | The key the player presses to start dialogue when standing in the trigger zone. | `E` | Ignored if **Start On Enter** is ticked. |
| **Start On Enter** | Tick to start dialogue automatically the instant the player enters the zone. | Off | Useful for cutscenes or area-triggered dialogue. |
| **Interact Prompt** | An optional GameObject (e.g. a floating "Press E" label) shown while the player is in range. Hidden automatically when the player leaves. | None | Drag any child GameObject here. |
| **Active Entry Point Key** | Leave empty to start from the graph's default start node. Set to an [entry point](entry-points.md) key to start from a specific branch. | Empty | See [Entry Points](entry-points.md). |

4. **Add a trigger Collider** to the same GameObject:
    - 3D: Add a `BoxCollider` (or `SphereCollider`, `CapsuleCollider`, etc.) and tick **Is Trigger**.
    - 2D: Add a `BoxCollider2D` (or `CircleCollider2D`, etc.) and tick **Is Trigger**.
5. **Make sure a Rigidbody exists** on either the player or the trigger object. Unity's `OnTriggerEnter` / `OnTriggerEnter2D` requires at least one Rigidbody in the collision pair. Your player typically already has one. If you get no trigger events, this is the most common cause.

> The `DialogueTrigger` component validates its setup and logs warnings to the Console if the trigger Collider or Rigidbody is missing.

---

## Step 7 — Set up the UI

Threader ships with a built-in `DialogueUI` component that provides a basic dialogue display using **UI Toolkit**. It is meant to get something on screen immediately for testing — most projects will eventually replace it with a custom UI. See [UI](ui.md) for custom UI instructions.

### Using the built-in DialogueUI

1. Create a new empty GameObject in the scene. Name it `DialogueUI`.
2. **Add Component → DialogueUI**. Unity automatically adds a `UIDocument` component because `DialogueUI` requires it.
3. On the **UIDocument** component (which was just added), set the **Source Asset** field to `UI_Dialogue.uxml`. You can find this file in `Assets/Threader/UI/`.
4. On the **DialogueUI** component, set the **Choice Button Template** field to `UI_ChoiceButton.uxml` (also in `Assets/Threader/UI/`). Without this, choice buttons will be created as plain unstyled buttons that work but look basic.

**That is all the wiring you need.** `DialogueUI` automatically subscribes to `DialogueManager.Instance.OnNPCLine`, `OnChoiceNode`, and `OnDialogueEnd` when it starts — no manual event hookup is required.

### Optional DialogueUI settings

| Field | Default | What it controls |
|---|---|---|
| **Chars Per Second** | `10` | Typewriter speed — how fast text appears character by character. |
| **Punctuation Pause** | `0.15` | Extra seconds added after `.` `,` `!` `?` `;` — makes the typewriter feel more natural. |
| **Choice Stagger Delay** | `0.5` | Seconds between each choice button fading in — creates a cascading reveal effect. |

### Key bindings (built-in DialogueUI)

- **Space** — skips the typewriter animation (reveals the full line instantly) or advances to the next line.
- **Escape** — cancels the dialogue (unless the current node has **Prevent Dialogue Exit** enabled).

---

## Step 8 — Test it

1. Press **Play** in Unity.
2. Walk your player into the NPC's trigger zone (if using `DialogueTrigger`) or trigger `StartDialogue()` from your interaction system (if using `NPCDialogue`).
3. The dialogue panel should appear with your NPC's first line.
4. Press **Space** to advance lines. When choices appear, click one.
5. The conversation should follow the branch you built and end cleanly.

### If nothing happens

- **No dialogue appears?** Make sure `DialogueManager` exists in the scene and has no errors in the Console. Check that the NPC's `Graph` field is assigned.
- **Trigger not firing?** Check that the Collider has **Is Trigger** ticked, the player has the correct tag, and at least one object in the pair has a Rigidbody.
- **UI not showing?** Make sure `DialogueUI` is in the scene and `UIDocument.Source Asset` is assigned to `UI_Dialogue.uxml`.
- **Speaker name warning?** The name on the `NPCDialogue` component must match the name used on the NPC nodes in the graph, character for character.

See [Troubleshooting](troubleshooting.md) for more solutions.

---

## What you just built

- A **[Speaker Roster](speaker-roster.md)** declaring valid speaker names for your project
- A **[Dialogue Graph](graph-editor.md)** asset with nodes, connections, and an assigned Start node
- A **DialogueManager** singleton in the scene driving conversations and firing events
- An **NPCDialogue** or **DialogueTrigger** component on your NPC that starts the conversation
- A **DialogueUI** component wired to the built-in UI Toolkit display

---

## Next steps

Now that you have a basic conversation working, explore the rest of Threader:

| What to learn | Page | Why |
|---|---|---|
| Build a full quest NPC with variables, conditions, and events | **[Tutorial](tutorial.md)** | Hands-on walkthrough that covers everything in one guided project |
| Gate choices behind game state | **[Variables](variables.md)** | Hide or grey out choices based on booleans, integers, or strings — no C# needed |
| Write C# conditions for complex logic | **[Conditions](conditions.md)** | Check inventory, quest state, reputation, time of day, etc. |
| Branch to different dialogue on repeat visits | **[Entry Points](entry-points.md)** | NPC says different things based on story progress |
| Add voice-over audio and lip-sync animation | **[Line Sheet](line-sheet.md)** | Per-speaker audio clips and Animator actions for every line |
| Support multiple languages | **[Translation](translation.md)** | One line sheet per language, switch at runtime |
| Fire-and-forget ambient barks | **[Bark System](bark.md)** | NPCs mutter as the player walks past |
| Build a custom UI (Canvas/uGUI) | **[UI](ui.md)** | Replace the built-in UI Toolkit display with your own |
| Save and load dialogue progress | **[Saving](saving.md)** | Persist variables, entry points, and choice history |
| Full shortcut and toolbar guide | **[Graph Editor](graph-editor.md)** | Canvas controls, sidebar sections, right-click menus |
| All 15 node types with field descriptions | **[Node Reference](nodes.md)** | Detailed reference for every node |
| Methods, events, and interfaces | **[API Reference](api-reference.md)** | Everything you can call from C# |
