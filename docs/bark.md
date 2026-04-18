# Bark System

Barks are fire-and-forget ambient lines that play without entering a full conversation and without blocking the player. An NPC can mutter as the player walks past, a guard can react to a sound, or a merchant can call out — all without triggering `OnDialogueStarted` or touching the main dialogue panel.

---

## Overview

A bark graph is a normal `DialogueGraph` asset with **Graph Type** set to **Bark** in the GRAPH sidebar. The runtime runs it on a second, non-blocking runner. Each line fires `OnBark` instead of `OnNPCLine`, leaving the main dialogue system completely unaffected.

Bark graphs support:

- **NPC Node** — each line fires `OnBark`
- **Random Node** — randomise which line path is taken
- **Weighted Random Node** — weighted random path selection
- **Switch Node** — multi-case condition routing
- **Branch Node** — condition-driven bark selection
- **Set Variable Node** — write variables mid-bark (e.g. mark a bark as heard)
- **Jump Node** — redirect to a tagged node
- **Fire Event Node** — broadcast named events during bark playback
- **Play Audio Node** — play audio clips
- **Animator Trigger Node** — set Animator parameters on registered speakers
- **Debug Node** — log messages to the Console
- **End Node** — terminates the bark, including the optional "On end ->" sub-graph slot

Bark graphs do **not** support Player Choice Node, Wait Node, or Sub Graph Node. These are hidden from the editor automatically. Wait nodes encountered in a bark graph auto-advance immediately.

---

## Setup

### 1. Create a bark graph

Right-click in the Project window -> **Create -> Threader -> Dialogue Graph**. Open it in the graph editor. In the **GRAPH** sidebar section, change **Graph Type** to **Bark**.

Build the graph normally — NPC nodes, branches, random nodes. Leave speaker fields blank if you want caller speaker fallback (see below).

### 2. Add a BarkSource

Add the `BarkSource` component to the NPC's GameObject (alongside `NPCDialogue`).

| Field | Description |
|---|---|
| **Bark Graph** | Assign the bark graph asset |
| **Trigger Mode** | `OnEnter` fires when the player enters the trigger collider. `OnTimer` fires on an interval. `Manual` — call `Bark()` from script. |
| **Player Tag** | The tag used to identify the player for `OnEnter` trigger detection. Default is `"Player"`. |
| **Cooldown** | Minimum seconds between barks |
| **Speaker Name** | The speaker name this NPC is registered under — must match their `NPCDialogue` **Speaker Name**. Used to resolve audio clips and animator actions from the bark graph's Line Sheet when the NPC node and graph have no speaker set. Populated from your SpeakerRoster assets as a dropdown. |
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

## Speaker resolution in bark graphs

NPC nodes inside a bark graph resolve their speaker name through a three-level fallback chain:

1. **Node Speaker** — set directly on that NPC node
2. **Graph Default Speaker** — `DefaultSpeakerName` on the bark graph itself
3. **BarkSource Speaker Name** — the **Speaker Name** field on the `BarkSource` component (or the `speakerName` parameter passed to `PlayBark()` from code)

A shared bark graph with blank speaker fields automatically voices, positions, and resolves line sheet data for whichever NPC triggered it.

---

## NPC-to-NPC / overheard dialogue

Because the bark runner waits for each audio clip to finish before advancing, you can script a sequential back-and-forth exchange between two NPCs as a single bark graph — no extra code needed.

**Example:** Two guards talking as the player walks past.

1. Create a bark graph with alternating NPC nodes: Guard A -> Guard B -> Guard A -> End
2. Assign audio clips to each line
3. In your `OnBark` handler, read `line.SpeakerName` to route each line to the correct speech bubble

---

## Checklist

- Graph Type is set to **Bark** in the GRAPH sidebar
- No Player Choice nodes in the bark graph (validator warns if found)
- `BarkSource` component is on the NPC GameObject
- `OnBark` is subscribed in a MonoBehaviour
- Speaker registered with `DialogueManager` if you need 3D audio positioning