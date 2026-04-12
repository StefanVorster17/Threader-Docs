# Speaker Roster

A **Speaker Roster** is a ScriptableObject asset that declares the valid speaker names in your project. It populates the Speaker dropdowns in graph nodes and on `NPCDialogue` components, so you select a name rather than type one.

---

## Why it exists

Every NPC node and player choice node references a speaker by name — a plain string like `"Villager"` or `"BlacksmithMira"`. At runtime, `DialogueManager` uses that string to find the matching `NPCDialogue` component in the scene, which provides the transform for camera look-at, spatial audio source, and Animator targeting.

Without a roster, the editor's Speaker dropdowns fall back to scanning every `SpeakerRoster` asset found anywhere in the project, which is slower and may include unrelated names from other scenes or packages. With a roster assigned, dropdowns are instant and scoped to exactly the names you defined.

---

## Creating a roster

1. Right-click in the Project window → **Create → Threader → Speaker Roster**
2. Name the asset something descriptive, e.g. `VillageRoster` or `Chapter1Speakers`
3. In the Inspector, click **+** and type each speaker name you want to use — these become the options in every Speaker dropdown across the graph editor and `NPCDialogue` components

Speaker names are **case-sensitive**. `"Villager"` and `"villager"` are treated as different speakers.

---

## Assigning to DialogueManager

Select your `DialogueManager` GameObject in the scene and drag the roster asset into the **Speaker Rosters** list in the Inspector.

You can assign **multiple** roster assets — all are merged into one flat list for the dropdowns. This is useful when a scene contains speakers from different chapters or systems, or when you split rosters by NPC group for easier maintenance.

---

## The ⚠ indicator in dropdowns

If a Speaker dropdown entry shows a **⚠** prefix, it means the stored name does not match any name in the currently assigned rosters. This happens most often when:

- A speaker was renamed in the roster but not updated on affected nodes
- A roster asset was unassigned from `DialogueManager`
- The value was typed manually and contains a typo

The original string is **never silently overwritten** — Threader preserves whatever is stored so data is not lost. Fix the mismatch by either correcting the name in the roster or updating the node's Speaker field to match the current roster.

---

## Setting up NPCDialogue components

Every speaker name used in the graph must have a matching `NPCDialogue` component in the scene with the identical **Speaker Name** field. This is how the runtime resolves the name to a real scene object.

### Primary NPC

The character the player physically interacts with to start the conversation.

- Add `NPCDialogue` to the NPC's root GameObject
- **Graph** — drag the `DialogueGraph` asset here
- **Speaker Name** — select from the dropdown populated by your assigned Speaker Roster
- **Is Interactable** — leave checked

### Secondary speaker

A character who appears in the same graph but is not the one the player clicks to start dialogue.

- Add `NPCDialogue` to that character's GameObject
- **Graph** — leave empty (secondary speakers don't own the graph)
- **Speaker Name** — select from the dropdown; must be the same name used in the graph node
- **Is Interactable** — uncheck

Without an `NPCDialogue` component on a secondary speaker, the runtime has no transform to target — look-at and Animator actions for that name silently do nothing and a warning is logged to the Console.

---

## Speaker name resolution order

When an NPC node fires, the runtime resolves the speaker through this chain:

1. **Node Speaker field** — the name set directly on that node
2. **Graph Default Speaker** — the fallback name set at the top of the GRAPH sidebar in the editor; used when the node's own Speaker field is blank
3. **Calling actor's Speaker Name** — used when the graph was invoked as a sub-graph; the speaker name of whoever started the top-level conversation is inherited automatically

This means a shared graph (e.g. a generic idle sequence referenced by many NPCs) works correctly without setting any Speaker field on its nodes — each NPC supplies their own name as the final fallback.

---

## Tips

- Keep one roster per scene or chapter rather than one giant roster for the whole project — dropdowns stay short and relevant, and it's obvious when a speaker is missing.
- Use the **Validate** button in the graph editor's PROJECT sidebar to catch ⚠ mismatches across all nodes in the current graph before release.
- Speaker Name matching is case-sensitive. If look-at or Animator actions are silently not firing, a name mismatch is the first thing to check.
