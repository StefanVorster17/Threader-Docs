# Tutorial: A Quest-Driven NPC from Scratch

This tutorial builds a complete, quest-aware NPC end-to-end. When you finish, you'll have:

- A villager who greets the player on first meeting
- A quest to retrieve a lost cat
- Dialogue that changes once the cat is returned
- A variable tracking the quest state
- Player choices gated behind conditions
- Events wired to your game code
- Entry points that auto-switch as the quest progresses

**Time:** ~30 minutes  
**Prerequisites:** `DialogueManager` in your scene, a basic UI that handles `OnNPCLine` and `OnChoiceNode`. See [Quick Start](quick-start.md) if you haven't done this yet.

---

## 1. Plan the conversation tree

Before touching the editor, sketch the branches:

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

Three branches, two entry points, one variable. Keep this sketch open while building.

---

## 2. Create the DialogueVariables asset

**Right-click** in Project → **Create → Threader → Variables Store**.  
Name it `VillagerVars`.

In the Inspector, add one variable:

| Name | Type | Default |
|---|---|---|
| `hasCat` | Bool | false |

Assign `VillagerVars` to **DialogueManager → Variables List**.

---

## 3. Create the graph

**Right-click** in Project → **Create → Threader → Dialogue Graph**.  
Name it `VillagerDialogue`. Double-click to open.

---

## 4. Build Branch 1 — First meeting

### NPC greeting node

Press **N** to create an NPC node. Click it to select, then in the inspector sidebar:

- **Speaker:** `Villager` (type the name, or pick from a roster if you have one set up)
- Click **+ Add Line** and enter: `Oh, a traveller! You haven't seen a small orange cat around here, have you?`

### Player choice node

Press **C** to create a Player Choice node. Connect the NPC node's output port to its input.

Add two choices (click **+ Add Choice** twice):

**Choice 0:**
- Text: `I'll keep an eye out for you.`

**Choice 1:**
- Text: `Sounds like a personal problem.`

### Two end nodes

Press **E** twice to create two End nodes — one for each choice.

Connect **Choice 0 output → End 0**, **Choice 1 output → End 1**.

On **both** End nodes, set **Set Entry Point On Complete** to `InProgress`.

> This means no matter what the player says, the next conversation will start from the `InProgress` branch.

---

## 5. Build Branch 2 — Quest in progress

### Mark an entry point

Press **N** to create a new NPC node somewhere below. Right-click it → **Set as Entry Point**. Type `InProgress` as the key. The node gets a yellow badge.

Set its line to: `Any luck finding my little {varName}?`

> Wait — we don't have a variable called `varName`. Let's use a plain name for now: `Any luck finding Mr Whiskers?` You could also add a `catName` string variable and use `{catName}` here if you want it configurable.

### Player choice node with a condition

Press **C** to create a Player Choice node below the InProgress NPC node. Connect them.

Add two choices:

**Choice 0:**
- Text: `I found him! Here you go.`
- Click **+ Add** under Conditions
  - Variable: `hasCat`
  - Operator: `Equal`
  - Value: `true`
  - Hide: ✓ (hide until the player actually has the cat)

**Choice 1:**
- Text: `Still looking, sorry.`

### Two end nodes

Create two End nodes. Connect **Choice 0 output → End A**, **Choice 1 output → End B**.

- **End A** (cat returned): set **Set Entry Point On Complete** to `QuestDone`
- **End B** (still looking): set **Set Entry Point On Complete** to `InProgress` (stays in this branch)

---

## 6. Build Branch 3 — Quest complete

Press **N** for a new NPC node. Right-click → **Set as Entry Point**, key: `QuestDone`.

Add two lines:
1. `You found Mr Whiskers! I can't thank you enough.`
2. `He's been with me for twelve years. I was so worried.`

Press **E** for an End node. Connect the NPC node to it. Leave **Set Entry Point On Complete** empty — the quest is over.

---

## 7. Add a fire-event node for the quest reward

Between the "cat returned" choice and End A, insert a **Fire Event** node (press **V**).

- **Event Key:** `QuestReward`
- **Global:** ✓ (so any listener in the scene can pick it up)

Reconnect: **Choice 0 output → Fire Event Node → End A**.

In your game code, subscribe once:

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
        GivePlayerGold(50);
}
```

---

## 8. Set hasCat from code

When the player finds the cat (however your game handles that), set the variable:

```csharp
// Find the asset and set the variable
var vars = DialogueManager.Instance.VariablesList[0]; // or find by name
vars.SetBool("hasCat", true);
```

The choice `I found him!` will now unhide automatically the next time the InProgress dialogue runs.

---

## 9. Add the NPC to the scene

Add an NPC GameObject with a `Collider` (set **Is Trigger**) and an `NPCDialogue` or `DialogueTrigger` component.

- Assign `VillagerDialogue` to the **Graph** field
- For `NPCDialogue`, set **Speaker Name** to `Villager` (must match the graph nodes exactly)
- Leave **Active Entry Point Key** empty — the first conversation starts from the default start node

---

## 10. Hook up the UI

If you're using the built-in `DialogueUI`, make sure it's in the scene and the Canvas is set up. Otherwise wire `DialogueManager.OnNPCLine` and `DialogueManager.OnChoiceNode` to your own UI from `OnEnable`:

```csharp
void OnEnable()
{
    DialogueManager.Instance.OnNPCLine    += ShowLine;
    DialogueManager.Instance.OnChoiceNode += ShowChoices;
    DialogueManager.Instance.OnDialogueEnd += HideUI;
}

void OnDisable()
{
    DialogueManager.Instance.OnNPCLine    -= ShowLine;
    DialogueManager.Instance.OnChoiceNode -= ShowChoices;
    DialogueManager.Instance.OnDialogueEnd -= HideUI;
}

void ShowLine(NPCLine line) => dialogueText.text = line.Text;

void ShowChoices(List<ChoiceData> choices)
{
    for (int i = 0; i < choices.Count; i++)
    {
        int index = i; // capture for the lambda
        var btn = Instantiate(choiceButtonPrefab, choiceContainer);
        btn.GetComponentInChildren<TMP_Text>().text = choices[i].Text;
        btn.interactable = !choices[i].IsLocked;
        btn.onClick.AddListener(() => DialogueManager.Instance.SelectChoice(index));
    }
}
```

---

## 11. Test the full loop

Enter Play mode and walk into the trigger zone (or call `StartDialogue` manually).

**First talk:**
- "I'll keep an eye out" or "Not my problem" — both should End and silently switch to `InProgress`

**Second talk:**
- "Any luck finding Mr Whiskers?" — only "Still looking" should be visible (hasCat is false)

**Set hasCat = true** — in a quick test you can do this in the Console with a one-shot script, or temporarily check a Update key press:
```csharp
if (Input.GetKeyDown(KeyCode.F5))
    DialogueManager.Instance.VariablesList[0].SetBool("hasCat", true);
```

**Third talk:**
- Both choices now visible — pick "I found him!"
- The `QuestReward` event fires → player gets 50 gold
- End node switches entry point to `QuestDone`

**Fourth talk:**
- The tearful thanks plays, then dialogue ends permanently at the default end node

---

## What to try next

| Extension | How |
|---|---|
| Add voice clips | Drag AudioClips onto each NPC line in the node inspector |
| Use a second NPC speaker | Add a second Speaker Name, register with `DialogueManager.RegisterSpeaker`, add NPC nodes with that speaker — the camera will cut between them |
| Track whether the player was rude | Add a `bool wasRude` variable, use a **Set Variable Node** after the "Not my problem" choice, add a condition hiding the reward choice if `wasRude = true` |
| Persist quest state on save | See [Saving](saving.md) — save `hasCat` and `ActiveEntryPointKey` |
| Test without entering Play mode | Open the **Dialogue Preview Window** (`Shift+Alt+P`), seed `hasCat = false`, and step through every branch |
