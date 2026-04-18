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

### 1. Create a line sheet

There are several ways to create a line sheet:

- **From the [graph editor](graph-editor.md)** — open the sidebar **PROJECT** section and click **Line Sheet Editor**. If no sheet exists yet, one is created automatically next to the graph asset.
- **Batch creation** — use **Threader → Create & Sync All Line Sheets** in the Unity menu bar to create and sync sheets for every graph in the project at once.
- **Manually** — right-click in the Project window → **Create → Threader → Line Sheet**. Then assign the graph to the sheet's **Source Graph** field and click **Sync** to populate its rows.

Sheets created automatically are named `{GraphName}_Sheet.asset` and placed next to their graph. For [multi-language setups](translation.md), create one sheet per language (e.g. `VillagerGraph_EN.asset`, `VillagerGraph_FR.asset`).

### 2. Open the Line Sheet Editor

From the sidebar **PROJECT** section, click **Line Sheet Editor**. This opens an overview of every NPC node in the graph, with its lines and speaker entries.

Alternatively, click **Line Data** on any individual NPC node in the graph canvas to open a focused editor for just that node.

### 3. Add speaker entries

For each line, add one entry per speaker who delivers that line. Set:

| Field | Description |
|---|---|
| **Speaker** | Name from your [SpeakerRoster](speaker-roster.md). Must match the speaker registered with `DialogueManager`. |
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

### Choice text localization (editor preview only)

Each line sheet also stores **ChoiceSheetRow** entries for Player Choice nodes. A `ChoiceSheetRow` has:

| Field | Description |
|---|---|
| `NodeGuid` | The Player Choice node this row belongs to |
| `ChoiceIndex` | Which choice in the node (0-based) |
| `PreviewText` | The localized choice label for this language |

Choice rows are looked up via `sheet.LookupChoiceRow(nodeGuid, choiceIndex)`.

> **Important:** Choice text localization via the line sheet is currently used for **editor preview only**. The graph editor reads `ChoiceSheetRow.PreviewText` so you can see translated choice text while authoring, but at runtime `DialogueManager.HandleChoices` takes choice text from `ChoiceData.Text` on the node (with `{variable}` token substitution). If you need localized choice text at runtime, override the text in your `OnChoiceNode` handler by looking up the active sheet yourself, or manage choice translations through your own localisation pipeline.

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

> **Tip:** Create a [Language Library](translation.md#language-library) asset and assign it to `DialogueManager`. The library pre-fills language slots on every graph so you only need to drag in sheet assets — no manual text entry. See [Translation — Language Library](translation.md#language-library).

### Fallback behaviour

`GetSheet(language)` resolves the sheet through this chain:

1. If the graph has **no sheets at all** (`LineSheets` is empty) → returns the legacy single `LineSheet` field (or `null` if that is also unset)
2. If `language` is **non-empty** and a matching `NamedLineSheet` entry exists → returns that sheet
3. If `language` is **non-empty** but no match is found → falls back to the **first sheet** in the list (the primary/default language)
4. If `language` is **null or empty** → returns the **first sheet** in the list

This means the first entry in the `LineSheets` list acts as the default language. A graph with at least one sheet always returns a sheet — it never returns `null` unless the first entry's Sheet field is unassigned.

For individual lines and rows:

- If the resolved sheet has no matching row for a given node/line, or the row's `PreviewText` is empty, the system falls back to the original text on the NPC node
- If the resolved sheet has no matching `LineSheetSpeakerEntry` for the current speaker, the line plays silently (no audio clip)
- The language string comparison is **exact** (case-sensitive) — `"French"` ≠ `"french"`

### Legacy single-sheet field

For backwards compatibility, `DialogueGraph` retains a legacy `LineSheet` field (hidden in the Inspector). Graphs created before multi-language support will continue to work — `GetSheet` falls back to the legacy field when the `LineSheets` list is empty.

When you open a graph with a legacy sheet in the Inspector, a warning box with a **Migrate to Multi-Language Sheets** button appears. Clicking it moves the existing sheet into the `LineSheets` list as the `"Default"` language entry and clears the legacy field.

---

## Speaker name fallback

The speaker name used for line sheet lookup follows the same resolution chain as speaker registration:

| Priority | Source |
|---|---|
| 1 | NPC node's own **Speaker Name** field |
| 2 | Graph's **Default Speaker** |
| 3 | Current actor's **Speaker Name** *(main dialogue)* or `BarkSource.speakerName` *(bark graphs)* |

This means a shared graph with no speaker set on nodes will automatically pick the correct speaker's clips based on which actor triggered the dialogue.

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
