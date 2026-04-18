# Quick Start

Get a working dialogue running in about 10 minutes. Follow these steps in order — every step builds on the last.

---

## Step 1 — Create a Dialogue Graph asset

In the **Project** window, right-click → **Create → Threader → Dialogue Graph**.  
Name it something like *VillagerGraph*.  
This asset stores all nodes and connections for one NPC conversation.

---

## Step 2 — Build the conversation

Double-click the graph asset to open the **Graph Editor**.

Use keyboard shortcuts or the left sidebar **CREATE** panel to add nodes:

| Key | Node |
|---|---|
| `N` | NPC Node — spoken lines |
| `C` | Player Choice Node — branching options |
| `E` | End Node — closes the conversation |
| `D` | Debug Node — log message + watch variables |
| `J` | Jump Node — redirect to a tagged node |
| `B` | Branch Node — route by variable condition |
| `V` | Set Variable Node — write variables mid-flow |
| `R` | Random Node — pick a random output |
| `F2` | Fire Event Node — broadcast a named event |
| `F3` | Play Audio Node — play AudioClips |
| `F4` | Animator Trigger Node — set Animator parameters |
| `F5` | Wait Node — pause for N seconds |
| `F6` | Sub Graph Node — call another Dialogue Graph |
| `W` | Weighted Random Node — weighted random output |
| `S` | Switch Node — multi-case condition routing |

Drag from the **output port** of one node to the **input port** of the next to connect them.  
Right-click the first node → **Set as Start Node**.  
Press **Ctrl+S** to save.

> See the [Graph Editor](graph-editor.md) page for a full shortcut list, sidebar guide, and node details.

---

## Step 3 — Add a DialogueManager to the scene

Create an empty GameObject in the scene and add the **DialogueManager** component.  
This is a singleton — **one per scene is enough**. It drives conversations and fires events that your UI listens to.

The following Inspector fields are optional but commonly used:

| Field | What to assign |
|---|---|
| **Speaker Rosters** | One or more `SpeakerRoster` assets. Populates speaker name dropdowns in the graph editor and enables camera look-at. See [Speaker Roster](speaker-roster.md). |
| **Variables List** | One or more `DialogueVariables` assets. Required to use the variable system in your graph. See [Variables](variables.md). |
| **Condition Provider** | A `ConditionProvider` asset. Required for C# custom conditions. See [Conditions](conditions.md). |
| **Audio Source** | An `AudioSource` for 2D (non-spatial) voice lines. Added automatically if left empty — no action needed. |

---

## Step 4 — Start dialogue from your NPC

Choose the approach that fits your project:

### Option A — NPCDialogue component (recommended)

Add the **NPCDialogue** component to your NPC GameObject and fill in its fields:

| Field | What to set |
|---|---|
| **Graph** | Your Dialogue Graph asset |
| **Speaker Name** | Must match the speaker name used on NPC nodes in the graph. Registers this NPC's transform for camera look-at and spatial audio. |
| **Is Interactable** | Leave ticked. Uncheck only for secondary speakers who never initiate dialogue themselves. |

Then call `StartDialogue()` from wherever your game handles player input (raycast hit, button press, animation event, etc.):

```csharp
GetComponent<NPCDialogue>().StartDialogue();
```

### Option B — DialogueTrigger component (no code required)

Add the **DialogueTrigger** component to the NPC GameObject and fill in its fields:

| Field | What to set |
|---|---|
| **Graph** | Your Dialogue Graph asset |
| **Player Tag** | Must match your Player GameObject's tag (default: `"Player"`) |
| **Interact Key** | Key the player presses to start dialogue when in range (default: `E`). Ignored if **Start On Enter** is ticked. |
| **Start On Enter** | Tick to start dialogue automatically as soon as the player enters the zone, with no key press. |
| **Interact Prompt** | Optional GameObject shown while the player is in range (e.g. a "Press E to talk" label). |

Also:

- Add a **Collider** (3D: `BoxCollider` / 2D: `BoxCollider2D`, etc.) to the same GameObject and enable **Is Trigger**.
- A `Rigidbody` (or `Rigidbody2D`) must be present on **either** the player or the trigger object for `OnTriggerEnter` to fire. The player typically already has one.

> `DialogueTrigger` does not register a speaker name, so camera look-at is not available when using this component. Use `NPCDialogue` if you need the camera to face the speaker during dialogue.

---

## Step 5 — Set up the UI

1. Create (or select) a GameObject in the scene.
2. Add the **DialogueUI** component. Unity automatically adds a `UIDocument` component because `DialogueUI` requires it.
3. On the **UIDocument** component, assign `UI_Dialogue.uxml` to the **Source Asset** field.
4. On the **DialogueUI** component, assign `UI_ChoiceButton.uxml` to the **Choice Button Template** field. Without this, choice buttons are created as plain unstyled buttons.

`DialogueUI` subscribes automatically to `OnNPCLine`, `OnChoiceNode`, and `OnDialogueEnd` on the `DialogueManager` at startup — no additional event wiring is required.

> Want a fully custom UI instead? The [UI](ui.md) page walks through a minimum implementation from scratch using those same events.

---

## Done!

What you just built:

- A **Dialogue Graph** asset with nodes, connections, and an assigned Start node
- A **DialogueManager** singleton in the scene driving conversations and firing events
- An **NPCDialogue** or **DialogueTrigger** component on your NPC that starts the conversation
- A **DialogueUI** component wired to the built-in UI Toolkit display

Ready to go further?

- **[Variables](variables.md)** — gate choices on game state, no C# needed
- **[Conditions](conditions.md)** — write C# conditions for complex game logic
- **[Entry Points](entry-points.md)** — branch to different dialogue based on story state
- **[Graph Editor](graph-editor.md)** — shortcuts, node types, toolbar guide
- **[API Reference](api-reference.md)** — full list of methods, events, and interfaces
