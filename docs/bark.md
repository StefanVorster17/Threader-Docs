# Bark System

Barks are fire-and-forget ambient lines that play without entering a full conversation and without blocking the player. An NPC can mutter as the player walks past, a guard can react to a sound, or a merchant can call out — all without triggering `OnDialogueStarted` or touching the main dialogue panel.

---

## Overview

A bark graph is a normal `DialogueGraph` asset with **Graph Type** set to **Bark** in the `DialogueGraph` asset's Inspector. The runtime runs it on a second, non-blocking runner. Each line fires `OnBark` instead of `OnNPCLine`, leaving the main dialogue system completely unaffected.

Bark graphs support:

- **Bark NPC Node** — bark-native line node with built-in [multi-speaker support](#multi-speaker-barks-with-bark-npc-node); each line fires `OnBark`. This is what the CREATE sidebar offers in place of NPC Node once a graph's type is Bark.
- **NPC Node** — legacy single-speaker line node. No longer offered in the sidebar for new bark graphs, but any NPC nodes already in a bark graph keep working via the same speaker-fallback path described below.
- **Random Node** — randomise which line path is taken
- **Weighted Random Node** — weighted random path selection
- **Switch Node** — multi-case condition routing
- **Branch Node** — condition-driven bark selection
- **Set Variable Node** — write [variables](variables.md) mid-bark (e.g. mark a bark as heard)
- **Jump Node** — redirect to a tagged node
- **Fire Event Node** — broadcast named [events](events.md) during bark playback
- **Play Audio Node** — play audio clips
- **Animator Trigger Node** — set Animator parameters on registered speakers
- **Debug Node** — log messages to the Console
- **End Node** — terminates the bark, including the optional "On end ->" [sub-graph](sub-graph.md) slot

Bark graphs do **not** support Player Node, Player Choice Node, Wait Node, or Sub Graph Node. These are hidden from the editor automatically. Wait nodes encountered in a bark graph auto-advance immediately.

---

## Setup

### 1. Create a bark graph

Right-click in the Project window -> **Create -> Threader -> Graph -> Dialogue Graph**. Select the new asset and, at the top of its **Inspector**, change **Graph Type** from `Dialogue` to `Bark`. If the graph already contains nodes, a confirmation dialog warns that switching type may break existing nodes, line sheets, and connections — confirm to proceed.

Open the graph in the [graph editor](graph-editor.md). The top bar now shows a **`· Bark`** badge next to the graph name, the sidebar's GRAPH section becomes [**SPEAKERS**](graph-editor.md#speakers) (Bark Speakers picker instead of Default Speaker / Look At Speaker), and the CREATE sidebar swaps **NPC Node** for **Bark NPC Node**.

Build the graph normally — Bark NPC nodes, branches, random nodes. Leave speaker fields blank on legacy NPC nodes if you want caller speaker fallback (see below); for `BarkNPCNode`, add one entry per speaker instead (see [multi-speaker barks](#multi-speaker-barks-with-bark-npc-node)).

### 2. Add a BarkSource

Add the `BarkSource` component to the NPC's GameObject (alongside `NPCDialogue`).

| Field | Description |
|---|---|
| **Bark Graph** | Assign the bark graph asset |
| **Trigger Mode** | `OnEnter` fires when the player enters the trigger collider. `OnTimer` fires on an interval. `Manual` — call `Bark()` from script. |
| **Player Tag** | The tag used to identify the player for `OnEnter` trigger detection. Default is `"Player"`. |
| **Cooldown** | Minimum seconds between barks |
| **Speaker Name** | Dropdown scoped to the assigned bark graph's **Bark Speakers** list (or free-text if no graph/speakers are configured yet). Selects which `BarkNPCNode` speaker entry this NPC plays, and doubles as the third-level fallback for legacy NPC-node speaker/Line Sheet resolution (see below). |
| **Randomise Speaker** | When enabled, `Start()` picks a random name from the bark graph's **Bark Speakers** list and uses it as this NPC's Speaker Name instead of the fixed value above. Useful for crowds of interchangeable NPCs sharing one `BarkSource` prefab. |
| **Suppress During Dialogue** | When true (default), silently skips the bark while a full conversation is active |

### 3. Wire the output

Subscribe to `DialogueManager.OnBark` in any MonoBehaviour:

```csharp
void OnEnable()
{
    DialogueManager.Instance.OnBark += HandleBark;
}

void OnDisable()
{
    DialogueManager.Instance.OnBark -= HandleBark;
}

void HandleBark(NPCLine line)
{
    // Show line.Text in a world-space speech bubble, HUD ticker, etc.
    Debug.Log($"[Bark] {line.SpeakerName}: {line.Text}");
}
```

---

## Playing barks from code

```csharp
DialogueManager.Instance.PlayBark(barkGraph, speakerTransform, speakerName);
```

- `barkGraph` — must have `IsBark = true` (Graph Type set to Bark)
- `speakerTransform` — used for 3D audio positioning; may be `null` for 2D audio
- `speakerName` — optional; used as the third-level speaker fallback for line sheet lookup (see below). `BarkSource` passes its **Speaker Name** field here automatically.

---

## Multi-speaker barks with Bark NPC Node

**Problem solved:** several NPC "types" (e.g. Male Villager, Female Guard) can share one bark graph while each playing their own voice, animations, and audio — without duplicating the graph per type. This is what [Bark NPC Node](nodes.md#bark-npc-node) is for. It carries one shared text line plus a list of per-speaker entries; at runtime the entry whose **Speaker Name** matches the calling NPC's resolved speaker name is selected.

### Setup

1. **Create a `BarkSpeakerRoster`** — right-click in the Project window → **Create → Threader → Speakers → Bark Speaker Roster**. Add every speaker type you want available across your bark graphs (e.g. `MaleVillager`, `FemaleGuard`).
2. **Assign the roster** — select your `DialogueManager` and drag the roster asset into the **Bark Speaker Rosters** list. You can assign multiple rosters; they merge into one flat list.
3. **Pick speakers for this graph** — open the bark graph, go to the sidebar's [**SPEAKERS**](graph-editor.md#speakers) section → **Bark Speakers**, and move the names this graph should use from **Available from roster** into **In this graph**.
4. **Place `BarkNPCNode`s** — drag the **Bark NPC Node** pill from CREATE onto the canvas. Its per-speaker dropdowns are populated from the graph's local Bark Speakers list (step 3), not the full roster — so unrelated speakers never clutter the picker.
5. **Set each `BarkSource`'s Speaker Name** — on every NPC's `BarkSource` component, the **Speaker Name** dropdown is scoped to the same graph-local list. This is the string matched against each `BarkNPCNode`'s speaker entries at runtime. Enable **Randomise Speaker** instead if you want a NPC to pick a random voice from the list each time it starts.

### Backwards compatibility

Legacy `NPCNode`s already placed in a bark graph continue to work exactly as before, via the three-level speaker fallback chain below — you don't have to migrate existing bark graphs to adopt `BarkNPCNode` for new content.

---

## Speaker resolution in bark graphs

### Legacy NPC Node

NPC nodes inside a bark graph resolve their speaker name through a three-level fallback chain:

1. **Node Speaker** — set directly on that NPC node
2. **Graph Default Speaker** — `DefaultSpeakerName` on the bark graph itself
3. **BarkSource Speaker Name** — the **Speaker Name** field on the `BarkSource` component (or the `speakerName` parameter passed to `PlayBark()` from code)

A shared bark graph with blank speaker fields automatically voices, positions, and resolves line sheet data for whichever NPC triggered it.

### Bark NPC Node

`BarkNPCNode` does not use the fallback chain above — it looks up its `Speakers` list directly for an entry whose **Speaker Name** equals the calling NPC's resolved speaker name (the `BarkSource.speakerName` field, or the `speakerName` argument passed to `PlayBark()`). If no entry matches, the node's shared **Text** still displays but no per-speaker audio or animation plays.

`BarkSource.speakerName` is required at runtime for `BarkNPCNode` to resolve correctly — it identifies which entry inside the node to use.

---

## Line sheets in bark graphs

`BarkNPCNode` has its own multi-language [Line Sheet](line-sheet.md) support, matching how NPC Node and Player Node work:

- The inline **Clip**, **Line Key**, and **Animator Actions** fields on each speaker entry remain as a **fallback** — a `BarkNPCNode` works with no line sheet at all.
- When a `LineSheetRow` exists for the node and the active language, its data takes priority: `PreviewText` overrides the shared **Text**, and each `LineSheetSpeakerEntry`'s `Clip` / `LineKey` / `AnimatorActions` override the matching inline speaker entry.
- Per-speaker audio in the sheet uses `LineSheetSpeakerEntry.LineKey` — a field distinct from `LineSheetRow.LineKey` (which NPC Node and Player Node use, shared across all speakers on that row).
- Each `BarkNPCNode` has exactly one `LineSheetRow` (`LineIndex = 0`); its speaker entries are seeded automatically from the node's own `Speakers` list.

Click **Line Data** on a `BarkNPCNode` to open its per-node editor — the same popup style used by NPC Node and Player Node, scoped to this node's speakers with no add/remove-line controls (a `BarkNPCNode` only ever has one line).

---

## NPC-to-NPC / overheard dialogue

Because the bark runner waits for each audio clip to finish before advancing, you can script a sequential back-and-forth exchange between two NPCs as a single bark graph — no extra code needed.

**Example:** Two guards talking as the player walks past.

1. Create a bark graph with alternating Bark NPC nodes (or legacy NPC nodes): Guard A -> Guard B -> Guard A -> End
2. Assign audio clips to each line (per-speaker entries if using `BarkNPCNode`)
3. In your `OnBark` handler, read `line.SpeakerName` to route each line to the correct speech bubble

---

## Checklist

- Graph Type is set to **Bark** in the `DialogueGraph` Inspector
- No Player Choice or Player nodes in the bark graph (validator warns on Player Choice nodes)
- `BarkSource` component is on the NPC GameObject
- For multi-speaker barks: a `BarkSpeakerRoster` is assigned to `DialogueManager`, the graph's **Bark Speakers** list is populated, and each `BarkSource.speakerName` is set (or **Randomise Speaker** is enabled)
- `OnBark` is subscribed in a MonoBehaviour
- Speaker registered with `DialogueManager` if you need 3D audio positioning