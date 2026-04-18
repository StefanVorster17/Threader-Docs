# Line Sheet

A **Dialogue Line Sheet** is a companion asset for a `DialogueGraph` that stores per-speaker audio clips and animator actions for every NPC line in that graph. Multiple speakers can share the same graph — each has their own clip and animations per line, looked up at runtime by speaker name.

---

## Overview

Line sheets separate audio and animation data from the dialogue graph itself. This means:

- Writers and voice directors can work in the same graph without touching audio assets
- Multiple NPCs (e.g. a merchant in two different towns) can reuse the same graph but play different voice lines
- Swapping VO or adding a new language doesn't require editing the graph

---

## Setup

### 1. Open a graph in the graph editor

Line sheets are always created relative to a graph and are never created manually. There is no **Create → Threader → Dialogue Line Sheet** entry in the Project window — use the graph editor instead.

### 2. Open the Line Sheet Editor

From the sidebar **PROJECT** section, click **Line Sheet Editor**. This opens an overview of every NPC node in the graph, with its lines and speaker entries.

Alternatively, click **Line Data** on any individual NPC node in the graph canvas to open a focused editor for just that node.

### 3. Add speaker entries

For each line, add one entry per speaker who delivers that line. Set:

| Field | Description |
|---|---|
| **Speaker** | Name from your `SpeakerRoster`. Must match the speaker registered with `DialogueManager`. |
| **Clip** | The `AudioClip` played when this speaker delivers this line. |
| **Animator Actions** | Optional list of animator parameters to fire when the line starts (parameter name, type, and value). |

---

## Batch creation

To create and sync sheets for every graph in the project at once, use:

**Threader → Create & Sync All Line Sheets**

This scans all `DialogueGraph` assets, creates a sheet next to any graph that doesn't have one, and syncs all NPC node rows. Safe to re-run — existing data is never overwritten.

---

## Runtime resolution

At runtime, `DialogueManager` resolves the clip and animator actions for each line via:

```csharp
sheet.Lookup(nodeGuid, lineIndex, speakerName)
```

The lookup returns a `LineSheetSpeakerEntry` containing the `Clip` and `AnimatorActions` for that exact node / line / speaker combination. If the graph has no sheet, or the sheet has no matching entry, the line plays silently — no crash.

Sheets are created automatically next to their graph with the name `{graphName}_Sheet.asset`.

### Text localization via PreviewText

Each `LineSheetRow` has a **PreviewText** field. When the active line sheet contains a non-empty `PreviewText` for a given node/line, that text is used **instead of** the original text on the NPC node. The fallback chain is:

1. `LineSheetRow.PreviewText` from the active language's sheet (if non-empty)
2. `NPCLine.Text` on the node itself (the original/source language)

This is the primary mechanism for localising NPC dialogue text without editing the graph.

### Choice text localization

Each line sheet also stores **ChoiceSheetRow** entries for Player Choice nodes. A `ChoiceSheetRow` has:

| Field | Description |
|---|---|
| `NodeGuid` | The Player Choice node this row belongs to |
| `ChoiceIndex` | Which choice in the node (0-based) |
| `PreviewText` | The localized choice label for this language |

Choice rows are looked up via `sheet.LookupChoiceRow(nodeGuid, choiceIndex)`.

---

## Multi-language support

Threader supports multiple languages per graph using the **LineSheets** list on `DialogueGraph`. Instead of a single sheet, a graph can have one sheet per language — each sheet stores its own audio clips, animator actions, and localized text.

### How it works

Each `DialogueGraph` has a `LineSheets` field: a `List<NamedLineSheet>`. Each entry pairs a **Language** string (e.g. `"English"`, `"French"`, `"Japanese"`) with a `DialogueLineSheet` asset.

At runtime, `DialogueManager` has an **active language** that determines which sheet is used:

```csharp
// Set the active language (call once, e.g. from a settings menu)
DialogueManager.Instance.SetActiveLanguage("French");

// Read the current language
string lang = DialogueManager.Instance.ActiveLanguage;
```

When an NPC node plays, the manager calls `graph.GetSheet(activeLanguage)` to retrieve the correct sheet. The sheet's `PreviewText` fields provide localised line text, and the sheet's `Clip` fields provide language-specific audio.

### Setup

1. Create one `DialogueLineSheet` per language for each graph (e.g. `VillagerGraph_EN.asset`, `VillagerGraph_FR.asset`)
2. Open the graph in the Graph Editor, or edit the `DialogueGraph` asset directly in the Inspector
3. In the **LineSheets** list, add one entry per language:

| Language | Sheet |
|---|---|
| `English` | `VillagerGraph_EN` |
| `French` | `VillagerGraph_FR` |

4. Fill in each sheet with the appropriate audio clips, animator actions, and `PreviewText` translations
5. At runtime, call `DialogueManager.Instance.SetActiveLanguage("French")` to switch languages

### Fallback behaviour

- If the active language has no matching sheet, or the sheet has no matching row, the system falls back to the original node text and plays silently (no audio)
- If `ActiveLanguage` is null or empty, `GetSheet` returns `null` and the original node text is used
- The language string comparison is exact (case-sensitive)

### Legacy single-sheet field

For backwards compatibility, `DialogueGraph` retains a legacy `LineSheet` field (hidden in the Inspector). Graphs created before multi-language support will continue to work. When migrating, move the existing sheet into the `LineSheets` list with an appropriate language label.

---

## Speaker name fallback

The speaker name used for line sheet lookup follows the same resolution chain as speaker registration:

| Priority | Source |
|---|---|
| 1 | NPC node's own **Speaker Name** field |
| 2 | Graph's **Default Speaker** |
| 3 | `BarkSource.speakerName` *(bark graphs only)* |

This means a shared bark graph with no speaker set on nodes will automatically pick the correct speaker's clips based on which `BarkSource` triggered it.

---

## Inspector

Selecting a `DialogueLineSheet` asset shows:

- **Stats** — number of lines, speakers, assigned clips, and any orphaned rows
- **Source Graph** — the graph this sheet belongs to
- **Open Line Sheet Editor** button — opens the full overview window
- A help note directing you to the graph editor

Orphaned rows appear when a node or line has been deleted from the graph but the sheet row was not cleaned up. Re-running **Create & Sync All Line Sheets** removes orphans.

---

## Checklist

- Each graph that needs audio has a `_Sheet.asset` next to it (created automatically on first open or via batch sync)
- Speaker names in the sheet match the **Speaker Name** fields in your `SpeakerRoster`
- The `LineSheets` list on the `DialogueGraph` asset has at least one entry pointing to the correct sheet
- For multi-language setups: one sheet per language is listed in `LineSheets`, and `SetActiveLanguage()` is called at runtime before dialogue starts
- `BarkSource` components have their **Speaker Name** field set when using shared bark graphs
