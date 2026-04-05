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

Drag from the **output port** of one node to the **input port** of the next to connect them.  
Right-click the first node → **Set as Start Node**.  
Press **Ctrl+S** to save.

> See the [Graph Editor](graph-editor.md) page for a full shortcut list, sidebar guide, and node details.

---

## Step 3 — Add a DialogueManager to the scene

Create an empty GameObject in the scene and add the **DialogueManager** component.  
This is a singleton — **one per scene is enough**. It drives conversations and fires events that your UI listens to.

Optionally assign a **Speaker Roster** asset to the **Speaker Rosters** slot — this populates all speaker dropdowns in the graph editor. See [Graph Editor — Speaker Roster assets](graph-editor.md#speaker-roster-assets) for details.

---

## Step 4 — Start dialogue from your own code

Call `StartDialogue` from wherever your game detects an interaction — trigger zone, button press, proximity check, etc.:

```csharp
public DialogueGraph graph; // drag your graph asset here in the Inspector

void OnInteract()
{
    DialogueManager.Instance.StartDialogue(graph);
}
```

---

## Step 5 — Set up the UI

Add the built-in **DialogueUI** component to a GameObject and wire up the UIDocument and field references in the Inspector — it handles everything automatically.

> Want a fully custom UI? The [API Reference](api-reference.md) lists all events (`OnNPCLine`, `OnChoiceNode`, `OnDialogueEnd`) you can subscribe to instead.

---

## Done!

What you just built:

- A graph asset with nodes and branching
- A **DialogueManager** singleton in the scene
- An **NPCDialogue** component on the NPC (call `StartDialogue()` from your interactor), or a **DialogueTrigger** component for automatic trigger-zone interaction
- UI events wired to display lines and choices

Ready to go further?

- **[Variables](variables.md)** — gate choices on game state, no C# needed
- **[Conditions](conditions.md)** — write C# conditions for complex game logic
- **[Entry Points](entry-points.md)** — branch to different dialogue based on story state
- **[Graph Editor](graph-editor.md)** — shortcuts, node types, toolbar guide
- **[API Reference](api-reference.md)** — full list of methods, events, and interfaces
