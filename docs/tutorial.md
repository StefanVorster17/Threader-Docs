# Tutorial: A Quest-Driven NPC from Scratch

This tutorial builds a complete, quest-aware NPC end-to-end. When you finish, you will have:

- A villager who greets the player on first meeting
- A quest to retrieve a lost cat
- Dialogue that changes once the cat is returned
- A variable tracking the quest state
- Player choices gated behind conditions
- Events wired to your game code
- Entry points that auto-switch as the quest progresses

**Time:** ~30 minutes  
**Prerequisites:** A scene with a `DialogueManager`, a `DialogueUI` (or custom UI), and basic understanding of the [Quick Start](quick-start.md) setup. If you have not done the Quick Start yet, complete that first — this tutorial assumes those components are already in your scene.

---

## 1. Plan the conversation tree

Before touching the editor, sketch the branches on paper or in a text file. This saves a lot of time rearranging nodes later:

```
[Start] Greet player
    → First meeting: "Have you seen my cat?"
        Choice: "I'll look for it"   → quest accepted  → [End: switch to InProgress]
        Choice: "Not my problem"     → [End: switch to InProgress]
    → [Entry: InProgress] "Any luck finding my cat?"
        Choice: "I found it!" (requires hasCat=true)  → reward  → [End: switch to QuestDone]
        Choice: "Still looking..."                     → [End: stay at InProgress]
    → [Entry: QuestDone] "Thank you so much!"
        → [End]
```

Three branches, two entry points, one variable. Keep this sketch open while building — you will refer back to it throughout.

---

## 2. Create the DialogueVariables asset

You need one variable to track whether the player has found the cat.

1. In the Project window, right-click → **Create → Threader → Variables Store**.
2. Name it `VillagerVars`.
3. Select the asset. In the Inspector, click **+ Add Variable**.
4. Fill in the fields:

| Name | Type | Default Value |
|---|---|---|
| `hasCat` | Bool | `false` (unchecked) |

5. Now assign this asset to your DialogueManager: select the `DialogueManager` GameObject in the scene, find the **Variables List** in the Inspector, click **+**, and drag `VillagerVars` into the new slot.

> **Why create the asset first?** Variable dropdowns inside the graph editor are populated from assets assigned to the DialogueManager. If you create the graph first, the dropdowns will be empty until you assign the variables asset. Setting this up now means everything is ready when you start building nodes.

---

## 3. Create the graph

1. In the Project window, right-click → **Create → Threader → Dialogue Graph**.
2. Name it `VillagerDialogue`.
3. Double-click it to open the [Graph Editor](graph-editor.md).

The graph editor opens with a dark canvas and a left sidebar. Before adding any nodes, set up the graph:

1. In the **GRAPH** sidebar section, set **Default Speaker** to `Villager` (from your [Speaker Roster](speaker-roster.md) dropdown). This means every NPC node in this graph will use "Villager" as the speaker unless you override it on individual nodes.
2. Leave **Graph Type** as **Dialogue**.
3. Leave **Look At Speaker** ticked.

---

## 4. Build Branch 1 — First meeting

This is the branch the player sees the very first time they talk to the villager.

### 4a — NPC greeting node

1. Press **`N`** to create an NPC node. It appears on the canvas — click it to select it.
2. The node has a **Lines** section. The first line is already created for you. Type the greeting text:

    `Oh, a traveller! You haven't seen a small orange cat around here, have you?`

3. The **Speaker** field should already show `(Graph Default: Villager)` since we set the Default Speaker in Step 3. Leave it as-is.

4. **Right-click** this node → **Set as Start Node**. A green **▶ START** badge appears. This is where the conversation begins on the player's first visit.

### 4b — Player Choice node

1. Press **`C`** to create a Player Choice node. Position it to the right of the NPC node.
2. **Connect them:** drag from the NPC node's **output port** (small circle on the right) to the Player Choice node's **input port** (small circle on the left). A wire appears connecting the two nodes.
3. In the Player Choice node, click **+ Add Choice** twice to create two choices.
4. Type the text for each:

    - **Choice 0:** `I'll keep an eye out for you.`
    - **Choice 1:** `Sounds like a personal problem.`

### 4c — Two End nodes

1. Press **`E`** twice to create two End nodes. Position them to the right of the Player Choice node.
2. Connect **Choice 0's output port → End node A's input**.
3. Connect **Choice 1's output port → End node B's input**.
4. On **both** End nodes, find the **Next entry** dropdown and select `InProgress`.

> **What does "Next entry" do?** When the player reaches this End node, the system automatically calls `SetEntryPoint("InProgress")` on the NPC's actor component. The next time the player talks to this NPC, the conversation will start from the node marked with the `InProgress` entry point instead of the Start node. See [Entry Points](entry-points.md) for the full explanation.

> **But "InProgress" doesn't exist yet!** That's fine — you will create it in the next step. The End node dropdown lets you type any key string. It will resolve at runtime as long as the entry point exists by then.

Your graph so far:

```
[▶ START] NPC: "Oh, a traveller! ..."
    └─→ Player Choice
          ├─ "I'll keep an eye out..." ─→ End (Next entry: InProgress)
          └─ "Sounds like a personal problem." ─→ End (Next entry: InProgress)
```

---

## 5. Build Branch 2 — Quest in progress

This branch plays on all subsequent visits while the player is still looking for the cat.

### 5a — Create the entry point NPC node

1. Press **`N`** to create a new NPC node. Position it below Branch 1 so the graph stays readable.
2. **Right-click the node → Set as Entry Point**. A popup appears asking for a key. Type `InProgress` and press Enter. The node gets a **yellow ⚑ badge** with the label `InProgress`.
3. Type the line text: `Any luck finding Mr Whiskers?`

### 5b — Player Choice node with a condition

1. Press **`C`** to create a Player Choice node below the InProgress NPC node.
2. Connect the InProgress NPC node's output → this Player Choice node's input.
3. Add two choices:

**Choice 0 — "I found him!"**

1. Type the text: `I found him! Here you go.`
2. This choice should only appear if the player actually has the cat. In the choice card, find the **Conditions** section and click **+ Add**.
3. Fill in the condition row:

    | Variable | Operator | Value | Hide | NOT |
    |---|---|---|---|---|
    | `hasCat` | `Equal` | `true` (checked) | ✓ | unchecked |

    - **Variable:** Select `hasCat` from the dropdown (populated from the `VillagerVars` asset you assigned to DialogueManager).
    - **Operator:** `Equal` — the condition passes when the variable equals the value.
    - **Value:** The checkbox should be **checked** (meaning `true`).
    - **Hide:** **Tick this.** When the condition fails (hasCat is false), this choice will be completely hidden from the player rather than shown as greyed out.
    - **NOT:** Leave unchecked — we want the condition as-is, not inverted.

**Choice 1 — "Still looking"**

1. Type the text: `Still looking, sorry.`
2. No conditions needed — this choice is always available.

### 5c — Two End nodes

1. Press **`E`** twice for two more End nodes.
2. Connect **Choice 0 → End node C** and **Choice 1 → End node D**.
3. On **End node C** (cat returned), set **Next entry** to `QuestDone`.
4. On **End node D** (still looking), set **Next entry** to `InProgress` — this keeps the NPC on this same branch until the player brings the cat.

Your graph so far:

```
[▶ START] NPC: "Oh, a traveller! ..."
    └─→ Player Choice → two Ends (both → InProgress)

[⚑ InProgress] NPC: "Any luck finding Mr Whiskers?"
    └─→ Player Choice
          ├─ "I found him!" [hasCat=true, hidden] ─→ End (→ QuestDone)
          └─ "Still looking, sorry." ─→ End (→ InProgress)
```

---

## 6. Build Branch 3 — Quest complete

This is the final branch, played once the cat has been returned.

1. Press **`N`** for a new NPC node. Position it below the InProgress branch.
2. **Right-click → Set as Entry Point**, key: `QuestDone`. Yellow badge appears.
3. Add two lines (click **+ Add Line** to add the second one):
    - Line 1: `You found Mr Whiskers! I can't thank you enough.`
    - Line 2: `He's been with me for twelve years. I was so worried.`
4. Press **`E`** for an End node. Connect the NPC node to it.
5. Leave **Next entry** on the End node as **(keep current)** — the quest is over, no further switching needed.

---

## 7. Add a Fire Event node for the quest reward

When the player returns the cat, you want your game to give a reward. Use a [Fire Event node](nodes.md#fire-event-node-f2) to broadcast a named event that your game code can listen for.

1. Press **`F2`** to create a Fire Event node.
2. Position it **between** the "I found him!" choice output and End node C. You will need to disconnect the existing wire first — click the wire and press **Delete**, or drag a new connection.
3. Wire it: **Choice 0 output → Fire Event node → End node C**.
4. In the Fire Event node, fill in the event row:
    - **Key:** `QuestReward`
    - **Scope** (dropdown): Select **Global** — this makes the event fire on both `OnNodeEvent` and `OnGlobalNodeEvent`, so any listener in the scene can pick it up regardless of which NPC is speaking.

> **Why Global?** Local events only fire on `OnNodeEvent` and are typically scoped to the current actor via `CurrentActor`. Global events additionally fire on `OnGlobalNodeEvent`, which any script anywhere in the scene can subscribe to without needing to know which NPC triggered it. For a reward system, Global is the right choice. See [Events](events.md) for the full explanation.

### Listen for the event in code

In any MonoBehaviour in your scene, subscribe to the global event:

```csharp
void OnEnable()
{
    DialogueManager.Instance.OnGlobalNodeEvent += HandleGlobalEvent;
}

void OnDisable()
{
    DialogueManager.Instance.OnGlobalNodeEvent -= HandleGlobalEvent;
}

void HandleGlobalEvent(string key)
{
    if (key == "QuestReward")
    {
        // Your game logic here — give gold, play a sound, update the quest log, etc.
        GivePlayerGold(50);
        Debug.Log("Player received 50 gold for finding Mr Whiskers!");
    }
}
```

> **Always unsubscribe in `OnDisable`.** If you only subscribe in `OnEnable` and forget to unsubscribe, the delegate persists even after the object is destroyed, which can cause null reference errors on the next scene load.

---

## 8. Set hasCat from code

When the player finds the cat (however your game handles that — picking up an object, completing a mini-game, interacting with the cat), set the variable:

```csharp
// Find the variables asset and set hasCat to true
DialogueVariables vars = DialogueManager.Instance.VariablesList[0];
vars.SetBool("hasCat", true);
```

The choice `I found him!` will now unhide automatically the next time the InProgress dialogue runs — no further wiring is needed. The condition check happens live every time the Player Choice node is reached.

> **Multiple variable assets?** If you have more than one asset in the `VariablesList`, find the right one by name: `DialogueManager.Instance.VariablesList.Find(v => v.name == "VillagerVars")`.

---

## 9. Add the NPC to the scene

You need a GameObject in the scene that the player interacts with to start the dialogue.

### Using NPCDialogue (recommended)

1. Create or select the NPC's GameObject.
2. **Add Component → NPCDialogue**.
3. Set the fields:
    - **Graph:** Drag `VillagerDialogue` here.
    - **Speaker Name:** Select `Villager` from the dropdown (must match the speaker name in the graph exactly).
    - **Is Interactable:** Leave ticked.
4. Leave **Active Entry Point Key** empty — the first conversation starts from the Start node. The entry point will be updated automatically by the End nodes as the quest progresses.

### Starting the dialogue

Call `StartDialogue()` from your player interaction system:

```csharp
// Example: from a raycast-based interaction script
void Update()
{
    if (Input.GetKeyDown(KeyCode.E))
    {
        Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        if (Physics.Raycast(ray, out RaycastHit hit, 3f))
        {
            NPCDialogue npc = hit.collider.GetComponent<NPCDialogue>();
            if (npc != null && npc.IsInteractable)
                npc.StartDialogue();
        }
    }
}
```

Alternatively, if you prefer a no-code approach, use **DialogueTrigger** instead — see [Quick Start — Option B](quick-start.md#option-b--dialoguetrigger-no-code-required) for the setup.

---

## 10. Make sure the UI is ready

If you followed the [Quick Start](quick-start.md), you should already have a `DialogueUI` in the scene. Verify:

- A `DialogueUI` component exists in the scene
- Its `UIDocument` has `UI_Dialogue.uxml` assigned as **Source Asset**
- The **Choice Button Template** is set to `UI_ChoiceButton.uxml`

If you are using a custom UI instead, make sure it subscribes to `DialogueManager.Instance.OnNPCLine`, `OnChoiceNode`, and `OnDialogueEnd`. See the [UI](ui.md) page for full custom UI examples with both Canvas/uGUI and UI Toolkit.

---

## 11. Test the full loop

Enter Play mode and interact with the NPC.

### First talk

- The greeting plays: "Oh, a traveller! You haven't seen a small orange cat around here, have you?"
- Two choices appear: "I'll keep an eye out" and "Sounds like a personal problem"
- Pick either one — both End nodes set the next entry to `InProgress`

### Second talk

- The InProgress branch plays: "Any luck finding Mr Whiskers?"
- Only **one** choice is visible: "Still looking, sorry." (because `hasCat` is still `false`, and the "I found him!" choice is hidden)
- Pick "Still looking" — the End node keeps the entry at `InProgress`

### Set hasCat = true

For testing, you can temporarily add a key press to set the variable without building the full cat-finding gameplay:

```csharp
void Update()
{
    if (Input.GetKeyDown(KeyCode.F9))
    {
        DialogueManager.Instance.VariablesList[0].SetBool("hasCat", true);
        Debug.Log("hasCat set to true!");
    }
}
```

Press **F9** during play, then talk to the NPC again.

### Third talk

- "Any luck finding Mr Whiskers?" — now **both** choices appear
- Pick "I found him!"
- The Fire Event node fires `QuestReward` → your `HandleGlobalEvent` runs → player gets 50 gold
- The End node switches the entry point to `QuestDone`

### Fourth talk

- The QuestDone branch plays: "You found Mr Whiskers! I can't thank you enough." followed by "He's been with me for twelve years. I was so worried."
- The End node has no Next entry switch, so the NPC stays at `QuestDone` permanently

### Testing without Play mode

You can also step through the graph in the editor without entering Play mode using the **Dialogue Preview Window**:

1. Open it via **Threader → Dialogue Preview** (or `Ctrl+Shift+P`).
2. Select the `VillagerDialogue` graph.
3. In the preview sidebar, find the **Variables** section and set `hasCat` to your desired test value.
4. Click **Start** and step through each node to verify all branches, conditions, and entry points behave as expected.

---

## 12. Save the graph

Press **Ctrl+S** to save the graph. Everything is stored in the `VillagerDialogue.asset` file.

---

## The complete graph layout

Your finished graph should look something like this:

```
[▶ START] NPC: "Oh, a traveller! ..."
    └─→ Player Choice
          ├─ "I'll keep an eye out..." ──────────────→ End (→ InProgress)
          └─ "Sounds like a personal problem." ──────→ End (→ InProgress)

[⚑ InProgress] NPC: "Any luck finding Mr Whiskers?"
    └─→ Player Choice
          ├─ "I found him!" [hasCat=true, HIDDEN] ─→ Fire Event (QuestReward, Global) ─→ End (→ QuestDone)
          └─ "Still looking, sorry." ──────────────→ End (→ InProgress)

[⚑ QuestDone] NPC: "You found Mr Whiskers! ..." → "He's been with me for twelve years. ..."
    └─→ End (keep current)
```

---

## What to try next

Now that you have a working quest NPC, here are ways to extend it:

| Extension | How |
|---|---|
| Add voice-over audio | Create a [Line Sheet](line-sheet.md) for the graph, add speaker entries for "Villager", and assign AudioClips to each line row. The audio plays automatically during dialogue. |
| Use a second NPC speaker in the same conversation | Add a second speaker name to your [Speaker Roster](speaker-roster.md). Add an `NPCDialogue` component on that character's GameObject with the matching Speaker Name. In the graph, set specific NPC nodes to use the second speaker — the camera will cut between them automatically. |
| Track whether the player was rude | Add a `bool wasRude` variable to `VillagerVars`. After the "Sounds like a personal problem" choice, insert a **[Set Variable Node](nodes.md#set-variable-node-v)** (`V`) that sets `wasRude = true`. Then add a condition on the reward choice hiding it if `wasRude` is `true`. |
| Persist quest state across game sessions | See [Saving](saving.md) — save the `hasCat` variable value and the NPC's `ActiveEntryPointKey` to your save system, then restore them on load. |
| Test branches without Play mode | Open the **[Dialogue Preview Window](preview-window.md)** (`Ctrl+Shift+P`), seed `hasCat` to the value you want, and step through every branch. |
| Add localized dialogue | Create one [Line Sheet](line-sheet.md) per language, assign them to the graph, and call `SetActiveLanguage("French")` at runtime. See [Translation](translation.md). |
| Reuse the greeting across multiple NPCs | Extract the greeting branch into a separate graph and call it via a **[Sub Graph Node](sub-graph.md)** from each NPC's personal graph. |
