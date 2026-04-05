# Troubleshooting

Answers to the most common problems. Check the Unity **Console** window first — Threader logs specific warnings and errors for most failure cases.

---

## Dialogue won't start

### "No DialogueManager found in the scene"
You called `StartDialogue()` but there is no `DialogueManager` component in the scene.

**Fix:** Add an empty GameObject, attach the `DialogueManager` component, and assign your Variables List and Speaker Rosters.

---

### "No DialogueGraph assigned"
`NPCDialogue` or `DialogueTrigger` has no graph in its **Graph** field.

**Fix:** Assign a `DialogueGraph` asset to the component in the Inspector.

---

### "DialogueGraph has no start node set"
The graph exists but has no **Start** node, or the Start node has been deleted.

**Fix:** Open the graph in the Graph Editor. If no Start node is visible, create one from the **CREATE** sidebar panel and connect it to your first NPC node.

---

### DialogueTrigger does nothing when the player walks in

Three things to check in order:

1. **Is Trigger** — the Collider on the trigger object must have **Is Trigger** ticked.
2. **Rigidbody** — either the trigger object or the player must have a `Rigidbody` (or `Rigidbody2D`) component. Without one, `OnTriggerEnter` never fires.
3. **Player Tag** — the `DialogueTrigger` **Player Tag** field (default `Player`) must exactly match the Tag on your player GameObject.

Threader logs warnings for missing colliders and Rigidbodies at startup — check the Console.

---

### `startOnEnter` is ticked but dialogue starts repeatedly

Dialogue is firing every time the trigger is re-entered.

**Fix:** Use an Entry Point to move the actor to a different branch after the first conversation, so subsequent enters play a different (or shorter) line. See [Entry Points](entry-points.md).

---

## Speaker issues

### Speaker name shows blank or "(Graph Default: )"

An NPC node's **Speaker** dropdown is set to a name that doesn't exist in any **SpeakerRoster** asset assigned to `DialogueManager`.

**Fix:**
1. Open **DialogueManager → Speaker Rosters** and confirm the correct roster is assigned.
2. Open the roster asset and confirm the speaker name matches exactly (case-sensitive) what's set on the NPC node.

---

### Animator not found / no animation triggers firing

`DialogueManager` logs `"Animator not found for speaker X"`.

**Fix:** Call `DialogueManager.Instance.RegisterSpeaker("SpeakerName", transform)` from the speaker's `Awake` or `Start`. `NPCDialogue` does this automatically — if you're using a custom actor you must register manually.

---

### Audio clips not playing from the speaker's position

The clip plays in 2D (centred) instead of from the speaker's world position.

**Fix:** Same as above — the speaker's `Transform` is only known to `DialogueManager` after `RegisterSpeaker` is called. Without a registered transform, audio falls back to the 2D `AudioSource` on the manager.

---

## Variables

### A condition has no effect — the choice always appears

- **Typo in the variable name.** Names are case-sensitive. `foundCat` ≠ `FoundCat`. Open the `DialogueVariables` asset and compare carefully.
- **Asset not assigned.** The asset containing that variable must be in **DialogueManager → Variables List**. If it's missing, the condition row is silently skipped.
- **Wrong asset type.** Checking a `Bool` variable with `GreaterThan 0` will always pass — use `Equal true` for booleans.

---

### Variable changes in the graph are lost after Play mode ends

This is correct behaviour. `DialogueVariables` resets to its serialized defaults when Play mode ends (via `OnDisable`). The on-disk asset is never modified during play.

**Fix:** If you need variables to survive sessions, save and restore them yourself — see [Saving](saving.md).

---

### `{varName}` token shows as literal text in dialogue

- The variable name inside `{}` must exactly match the **Name** field in the `DialogueVariables` asset.
- The asset must be assigned to **DialogueManager → Variables List**.
- Use `{varName:name}` to substitute the **Display Name** field instead.

---

## Conditions

### Custom condition always passes / always blocks unexpectedly

Check the **When Missing** setting on the `ConditionDefinition` asset:

| When Missing | Behaviour when no handler is registered |
|---|---|
| **Allow** | Condition passes (choice is always shown) |
| **Block** | Condition fails (choice is always hidden/locked) + Console warning |

If your handler isn't registered yet (e.g. the MonoBehaviour hasn't run `Awake`), the fallback fires instead.

---

### Condition handler stops working after a scene reload

You registered a delegate in `Awake` but forgot to unregister it in `OnDestroy`. The old delegate kept a reference to the destroyed object and throws a null reference on the next evaluation.

**Fix:**
```csharp
void Awake()   => ConditionService.Register("MyKey", param => Check(param));
void OnDestroy() => ConditionService.Unregister("MyKey");
```

---

## Graph editor

### Unsaved changes keep reappearing after save

If you press **Save** but the orange **● Unsaved** dot returns immediately, a script recompile is triggering a graph reload. This is normal — recompiles force a full editor refresh. Save again after the compile finishes.

---

### Graph editor is blank / won't load

- If you just recompiled, wait for the compile to finish and reopen the window.
- If the graph asset is missing from disk, the editor will open but show nothing. Re-assign the graph via **PROJECT → Load**.

---

### Node connections disappear after duplicating a node

Duplicating a node in Unity's Project window creates a copy with the same GUIDs, which causes duplicate GUID warnings. Always create new nodes via the **CREATE** panel in the Graph Editor — don't duplicate graph assets directly.

---

## Entry points

### Entry point key does nothing — dialogue always starts from the beginning

- The entry point key must be defined in the graph (a Start-type node with that key set, or a node tagged with that key).
- The **Active Entry Point Key** on `NPCDialogue` or `DialogueTrigger` must be set to the same key — it's case-sensitive.
- If the key doesn't match any node in the graph, the runner falls back to the default Start node.

---

### "Next entry" dropdown on End nodes is empty

No entry point keys have been set in this graph yet.

**Fix:** Open the graph. In the node you want to act as an entry point, expand the **Tag** field and assign a key, or use the **NAVIGATE → Entry Points** sidebar panel to see all defined keys.

---

## General

### Everything was working and suddenly nothing runs

1. Check the Console for red errors — a compile error will prevent all scripts from running.
2. Check that **DialogueManager** still exists in the scene (it's not part of a prefab that got reverted).
3. Check that Play mode is active — dialogue cannot start in edit mode via `StartDialogue`, only via the Preview Window.

---

### Console is spamming warnings every frame

A common cause is calling `StartDialogue` in `Update` without checking `DialogueManager.Instance.IsDialogueActive` first.

**Fix:**
```csharp
void Update()
{
    if (Input.GetKeyDown(KeyCode.E) && !DialogueManager.Instance.IsDialogueActive)
        npcDialogue.StartDialogue();
}
```
