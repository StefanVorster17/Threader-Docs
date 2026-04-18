# Troubleshooting

Answers to the most common problems. Check the Unity **Console** window first — Threader logs specific warnings and errors for most failure cases.

If your issue isn't covered here, or you've found a bug, open an issue on the [GitHub repository](https://github.com/StefanVorster17/Threader-Docs/issues) — include your Unity version, a description of what you expected, and what actually happened.

You can also join the [Threader Discord](https://discord.gg/nb8HCuX2h5) for community support and direct help.

---

**Jump to section:**

- [Dialogue won't start](#dialogue-wont-start)
- [Speaker issues](#speaker-issues)
- [UI and display](#ui-and-display)
- [Variables](#variables)
- [Conditions](#conditions)
- [Bark system](#bark-system)
- [Sub-graphs](#sub-graphs)
- [Line sheet and audio](#line-sheet-and-audio)
- [Saving and persistence](#saving-and-persistence)
- [Entry points](#entry-points)
- [Graph editor](#graph-editor)
- [General](#general)

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

## UI and display

### The dialogue box never appears

- Confirm `DialogueUI` (or your custom UI component) exists in the scene and is active.
- If using the built-in `DialogueUI`, confirm the `UIDocument` component on the same GameObject has a **Panel Settings** asset assigned and a source UXML asset (`UI_Dialogue.uxml`) set.
- Confirm `DialogueManager` has started — subscribe to `OnDialogueStarted` temporarily to verify the event fires.

See [UI](ui.md) for setup details.

---

### Speaker name is blank in the UI

- The NPC node's **Speaker** dropdown is set to a name not present in any assigned **Speaker Roster**. See [Speaker Roster](speaker-roster.md).
- If using the built-in `DialogueUI`, confirm it is reading from `OnNPCLine` and passing `line.SpeakerName` to your speaker name label.

---

### Choices are not appearing

- Confirm the `PlayerChoiceNode` has at least one output choice connected to a downstream node. Unconnected choices are filtered out at runtime.
- If using a custom UI, confirm you are subscribing to `OnChoiceNode` and calling `DisplayChoices(choiceNode.Choices)`.
- Locked choices (condition fails + **Behaviour** set to **Hide**) are removed from the list before `OnChoiceNode` fires — they will not appear even if you expect them.

---

### Choices appear but clicking does nothing

- Confirm you are calling `DialogueManager.Instance.SelectChoice(index)` with the correct index from the displayed list — not the original list index if hidden choices were removed.
- If using the built-in `DialogueUI`, confirm the choice button template UXML contains a `Button` element with the name `choice-button`.

---

### Typewriter effect shows all text at once

The **Chars Per Second** field on `DialogueUI` may be set to `0` or a very high value. Set it to something like `40`–`60` for a natural speed. See [UI](ui.md).

---

### Dialogue box stays visible after dialogue ends

- Confirm you are hiding the UI in response to the `OnDialogueEnded` event.
- If using the built-in `DialogueUI`, confirm the `UIDocument` root element visibility is being toggled correctly and that `OnDialogueEnded` is being received.

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

## Bark system

### Bark fires but nothing plays in the UI

Barks are fire-and-forget — they fire `OnBark`, not `OnNPCLine`. The built-in `DialogueUI` does not display barks by default. Subscribe to `OnBark` separately and display the line however your project requires (world-space text, subtitle, etc.). See [Bark System](bark.md).

---

### Bark graph runs a full conversation instead

The `DialogueGraph` asset's **Graph Type** is still set to **Dialogue** instead of **Bark**. Open the graph in the Graph Editor, expand the **GRAPH** sidebar panel, and change **Graph Type** to **Bark**.

---

### `TriggerBark` does nothing

- Confirm the `BarkTrigger` component has a **Bark Graph** assigned.
- Confirm the speaker name on the `BarkTrigger` matches a registered speaker (or that `RegisterSpeaker` has been called for that speaker).
- Bark graphs will not run if a full blocking dialogue is already active on the same speaker. Use a different speaker slot or wait for dialogue to end.

---

### Bark plays Choice Node / Wait Node unexpectedly

Bark graphs do not support `PlayerChoiceNode` or `WaitNode`. If either is present in the graph, remove them and reconnect the path. The runner will skip Wait nodes encountered in a bark graph, but a `PlayerChoiceNode` will stall execution.

---

## Sub-graphs

### Sub-graph runs but speaker name is wrong

Speaker resolution in sub-graphs follows a three-step chain: node speaker → graph default speaker → calling actor's speaker name. If the NPC nodes inside the sub-graph have blank speaker fields and the sub-graph's **Default Speaker** is also blank, the speaker resolves to whoever triggered the top-level conversation. Set the correct speaker explicitly on each NPC node, or set a **Graph Default Speaker** on the sub-graph asset. See [Sub-Graph](sub-graph.md).

---

### Sub-graph ends and dialogue stops instead of returning

The **Sub Graph Node** output port must be connected to the next node in the calling graph. If the output has no connection, the runner treats the sub-graph return as an End node and closes dialogue normally.

---

### Recursive sub-graph causes the game to freeze

Threader does not guard against cycles in the sub-graph call stack. If Graph A calls Graph B and Graph B calls Graph A, the runner will recurse indefinitely. Ensure your sub-graph references form a directed acyclic graph — no graph should call itself or any of its ancestors.

---

### Sub-graph audio / Line Sheet not found

Each sub-graph resolves audio clips and animator actions from its own **Line Sheet** list, not the calling graph's. If a sub-graph's NPC nodes produce no audio, confirm a Line Sheet is attached to the sub-graph asset with entries for the relevant speaker. See [Line Sheet](line-sheet.md).

---

## Line sheet and audio

### No audio plays during dialogue

1. Confirm a **Line Sheet** is assigned to the graph — in the Graph Editor GRAPH sidebar, the **Line Sheets** list should contain at least one entry.
2. Confirm the Line Sheet has an entry for the correct speaker name (case-sensitive).
3. Confirm the entry has an audio clip assigned for that NPC node's line index.
4. Confirm the speaker's `Transform` is registered so `DialogueManager` knows which `AudioSource` to use. See [Speaker Roster](speaker-roster.md).

---

### Line Sheet audio plays for one language but not another

Each language requires its own Line Sheet assigned in the graph's **Line Sheets** list. If a language slot exists but the Line Sheet for it has no clips, silence is the expected result. Confirm clips are assigned in the correct language's Line Sheet and that `SetActiveLanguage()` was called before dialogue starts. See [Translation](translation.md).

---

### Animator action doesn't fire

- The **Animator Parameter** name in the Line Sheet entry must exactly match the parameter name in the Animator Controller (case-sensitive).
- The speaker must be registered with `RegisterSpeaker` before the node plays, so `DialogueManager` knows which Animator component to target.
- Confirm the Animator Controller on the speaker's GameObject contains the parameter.

---

### Wrong audio clip plays for a line

Line Sheet entries are matched by line index — the order of NPC nodes in the graph matters. If you reorder or insert nodes, the indices shift and clips may misalign. Use the **Line Sheet Editor** (Graph Editor toolbar → **Line Sheet Editor**) to review and reassign clips after structural changes to the graph.

---

## Saving and persistence

### Variables reset every time the game starts

This is correct behaviour. `DialogueVariables` stores design-time defaults in the asset. At runtime, those defaults are copied into in-memory dictionaries. When Play mode ends (or the game exits), those dictionaries are discarded and the asset is untouched.

**Fix:** Save and restore variable values via your own persistence system. Read values with `GetBool`/`GetInt`/`GetString` and restore them with `SetBool`/`SetInt`/`SetString` before any dialogue runs. See [Saving](saving.md).

---

### Entry point key is not persisting between sessions

`IDialogueActor.ActiveEntryPointKey` is an in-memory string on the component. When the scene reloads, it resets to whatever the Inspector default is.

**Fix:** Save the key string as part of your save data and call `SetEntryPoint(key)` on the actor after loading. See [Saving](saving.md) and [Entry Points](entry-points.md).

---

### Choice history is gone after reloading the game

`DialogueChoiceHistory` is static and in-memory only. Call `DialogueChoiceHistory.GetSaveData()` when saving and `DialogueChoiceHistory.LoadSaveData(data)` on load before any dialogue runs. See [Saving](saving.md).

---

## Graph editor

### Reading console errors from Threader

All `LogError` and `LogWarning` messages from Threader include three pieces of identifying information:

- **Graph name** — the asset that was running when the error occurred
- **Node type** — e.g. `JumpNode`, `NPCNode`, `EndNode`
- **Short node ID** — the first 8 characters of the node's GUID

Example: `[Threader] JumpNode 'a3f9c812' in graph 'VillagerGraph': tag 'shop_loop' not found on any node.`

To locate the node: open the graph in the Graph Editor, paste the 8-char ID into the **Search GUID** field in the NAVIGATE sidebar, and press **Go**.

**Clicking the console entry** pings the source graph asset in the Project window. Unity does not support opening a custom editor window from a console double-click (that callback is reserved for `.cs` files) — the asset ping is the closest supported equivalent.

---

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
