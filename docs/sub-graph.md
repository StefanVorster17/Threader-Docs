# Sub-Graph System

The Sub Graph Node lets one `DialogueGraph` call another as a sub-routine, then return and continue — exactly like a function call in code. Use it to share common dialogue sequences (generic greetings, shop flows, idle lines) across many different NPC graphs without duplicating content.

---

## How it works

Place a **Sub Graph Node** in your graph, assign a target `DialogueGraph` asset, and wire it like any other node. When the runner reaches the Sub Graph Node:

1. The current graph is pushed onto the call stack
2. The target graph starts at its default start node (or at a named entry point if one is set)
3. When the target graph's End node is reached, the call stack pops back to the calling graph
4. Execution resumes on the output connected to the Sub Graph Node

The Sub Graph Node has a single output port. When the sub-graph ends, execution always continues from that output. If the output has no connection, the conversation ends normally.

---

## Speaker resolution

NPC nodes inside a sub-graph resolve their speaker name through a three-level chain:

| Priority | Source |
|---|---|
| 1 | **Node Speaker** — the speaker set directly on that NPC node |
| 2 | **Graph Default Speaker** — the `DefaultSpeakerName` on the sub-graph asset |
| 3 | **Calling actor's speaker name** — the `SpeakerName` from the `IDialogueActor` that started the top-level conversation |

This means a shared graph with blank speaker fields automatically resolves the display name and camera look-at to whoever triggered the dialogue. For audio clips and animator actions, the same resolved speaker name is used to look them up in the sub-graph's own Line Sheet — so if you need per-speaker audio, add entries for each speaker in that Line Sheet.

---

## End Node "On end →" sub-graph slot

The **End Node** has an optional **On end →** section with a `DialogueGraph` field and entry point key. When a graph is assigned:

1. The `Next entry` switch fires first — the actor's entry point is updated before the sub-graph runs
2. The sub-graph runs as a sub-routine
3. When the sub-graph's End node is reached, the conversation truly closes

This is useful for routing quest NPCs back to generic idle lines automatically after a unique conversation ends.

---

## Depth limit

Sub-graphs can be nested up to **16 levels deep**. Exceeding this limit logs an error and ends the dialogue cleanly to prevent infinite loops caused by circular references.

---

## Patterns

### Unimportant NPC — no unique dialogue

Assign a shared generic graph directly to the NPC's `NPCDialogue` component. Leave speaker fields blank in the shared graph. The NPC's own `Speaker Name` field serves as the caller fallback so the UI and camera resolve correctly for each individual NPC automatically. If these NPCs need audio clips, open the shared graph's Line Sheet and add a speaker entry for each NPC's speaker name with the appropriate clips.

### Important NPC — quest-driven routing

Give the NPC a personal "router" graph that contains only Branch or Entry Point logic — no dialogue lines. Each branch leads to a Sub Graph Node pointing at the relevant content graph (generic idle, quest intro, quest done, etc.).

```
Router graph:
  Start → Branch [questPhase == "done"]
      True  → Sub Graph Node (QuestDone.graph)       → End
      False → Sub Graph Node (GenericGreeting.graph) → End
```

Your quest system calls:
```csharp
npc.SetEntryPoint("QuestDone");
```
The router handles the rest — no graph duplication, no per-state NPC components.

### End-of-quest fallback

On the **End Node** of a quest-specific graph, set **On end →** to the generic idle graph. After the unique lines finish, the NPC delivers their normal ambient response before the conversation closes — automatically, with zero extra wiring.

### Shared shop or trade flow

Build one `ShopDialogue.graph` with all the shop interaction lines. Any merchant NPC points their Sub Graph Node at this asset. The speaker name resolves to each merchant automatically via the fallback chain. For audio clips and animator actions, open the shared graph's Line Sheet and add a speaker entry for each merchant — the runtime uses the resolved speaker name to look up the correct clips from that sheet.

---

## Editor notes

- The Sub Graph Node is **not available in bark graphs**. It is hidden from the sidebar and context menu when **Graph Type** is set to **Bark**.
- The validator checks for circular sub-graph references and warns if the depth limit could be exceeded.
- Sub-graphs can themselves contain Sub Graph Nodes (nested calls), up to the 16-level limit.
