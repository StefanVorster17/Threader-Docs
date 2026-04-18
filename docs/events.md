# Events

Events are named string keys fired at specific points in a conversation. They are the primary way dialogue drives the rest of your game — opening doors, spawning enemies, triggering cutscene beats, awarding items, updating UI — without writing any coupling code inside the dialogue system itself.

---

## Two ways to fire an event

### From an NPC Node

Every NPC node has an **Events** section (click **+ Add** to add a row). These events fire when the node is reached, **before** any lines are displayed.

| Field | Description |
|---|---|
| **Key** | The event string your listeners will receive. |
| **Global** | See [Local vs Global](#local-vs-global) below. |
| **✕** | Removes this row. |

Multiple events are fired in list order. Empty keys are skipped.

> Use NPC node events when you want something to happen at the same moment speaking begins — for instance, closing a shop UI, playing a sting, or setting a flag before the first line plays.

### From a Fire Event Node

The **Fire Event Node `[F2]`** fires events standalone without any dialogue being shown. It fires all its rows in list order, then advances silently.

This is the right choice when you need to trigger game systems **between** branches or **at a dead-end** in the flow without attaching the event to a speaking node.

> See [Node Reference — Fire Event Node](nodes.md#fire-event-node-f2) for the full field list.

---

## Local vs Global

Both authoring sources have a **Global** toggle on each event row.

| Setting | Event fires on | Use when |
|---|---|---|
| **Local** (off) | `DialogueManager.OnNodeEvent` only | The event is specific to this NPC — guard your handler with `CurrentActor` |
| **Global** (on) | `DialogueManager.OnNodeEvent` **and** `DialogueManager.OnGlobalNodeEvent` | The event needs to be received by any listener in the scene, regardless of which NPC is speaking |

---

## Responding without C# — the Inspector list

If you are using **NPCDialogue**, you can handle events entirely in the Inspector with no code.

### Setup

1. Select the NPC GameObject that has **NPCDialogue**.
2. In the Inspector, find the **Node Events** list.
3. Click **+** to add a row.
4. Set the **Key** field to match the event key you put in the graph.
5. Wire up the **Response** UnityEvent — drag a target object and choose a method, exactly like a Button `onClick`.

When the dialogue system fires an event whose key matches an entry in this list, the corresponding UnityEvent is invoked automatically.

You can add as many rows as you need — one per key.

> **This only works for Local events** routed through `OnNodeEvent`. The `NPCDialogue` component subscribes to `OnNodeEvent` and checks whether the currently running graph matches this NPC's assigned graph, so the response only fires during this NPC's own conversation.

---

## Responding from C# — event subscriptions

Subscribe in `OnEnable`, unsubscribe in `OnDisable`.

### Local events — `OnNodeEvent`

```csharp
void OnEnable()
{
    DialogueManager.Instance.OnNodeEvent += HandleNodeEvent;
}

void OnDisable()
{
    DialogueManager.Instance.OnNodeEvent -= HandleNodeEvent;
}

void HandleNodeEvent(string key)
{
    // Guard: only respond when this NPC is the one speaking
    if (DialogueManager.Instance.CurrentActor != myNpc) return;

    switch (key)
    {
        case "open_shop":   shopUI.Show();    break;
        case "give_reward": Inventory.Add(rewardItem); break;
    }
}
```

The `CurrentActor` guard prevents this handler from firing when a *different* NPC's dialogue triggers the same key name.

### Global events — `OnGlobalNodeEvent`

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
    // No actor guard needed — global events are intentionally broadcast
    if (key == "play_fanfare") AudioManager.Play("fanfare");
}
```

---

## Inspector UnityEvents on DialogueManager

For events that fire when dialogue **starts** or **ends** (not keyed events), `DialogueManager` exposes two `UnityEvent` fields visible in the Inspector:

| Field | Fires when |
|---|---|
| **On Dialogue Started** | Any conversation begins (equivalent to the C# `OnDialogueStartedEvent`) |
| **On Dialogue Ended** | Any conversation ends (equivalent to the C# `OnDialogueEnd`) |

Wire these up directly in the Inspector — no code needed. Useful for: hiding/showing a HUD, playing a background music transition, locking or unlocking a game system.

---

## Practical patterns

### Trigger a one-time effect mid-conversation

Add a **Fire Event Node** between two NPC nodes. Wire the key to your effect handler. The player never sees it — execution passes through silently.

```
[NPC: "Brace yourself..."]
    → [Fire Event: key="explosion" Global=true]
        → [NPC: "What was that?!"]
```

### Open a shop UI when the NPC starts speaking

On the NPC node that represents the shop greeting, add an Events row with key `open_shop` (Local). On your shop UI component:

```csharp
void OnEnable()
{
    DialogueManager.Instance.OnNodeEvent += key =>
    {
        if (DialogueManager.Instance.CurrentActor != shopkeeper) return;
        if (key == "open_shop") shopUI.SetActive(true);
    };
}
```

Or, without code: add a **Node Events** row to the **NPCDialogue** component on the shopkeeper with key `open_shop` and target `shopUI.SetActive(true)`.

### Multiple NPCs responding to the same global event

Mark the event **Global** on the node. Any component subscribed to `OnGlobalNodeEvent` receives it — no guard needed, no filtering by actor.

```csharp
// On a crowd controller, not tied to any one NPC
DialogueManager.Instance.OnGlobalNodeEvent += key =>
{
    if (key == "crowd_cheer") crowdAnimator.SetTrigger("Cheer");
};
```

---

## Key naming conventions

Events have no enforced format, but these patterns help avoid naming collisions:

| Pattern | Example |
|---|---|
| Verb | `open_shop`, `give_reward`, `play_fanfare` |
| Scope prefix | `villager_quest_start`, `boss_phase2_begin` |
| Dot notation | `quest.cat.accepted`, `audio.sting.dramatic` |

Keys are **case-sensitive**. `open_shop` ≠ `Open_Shop`.
