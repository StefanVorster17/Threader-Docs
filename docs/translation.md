# Translation & Localization

Threader supports multiple languages per dialogue graph through the **Line Sheet** system. Each language gets its own `DialogueLineSheet` asset containing translated text, language-specific audio clips, and animator actions. Switching languages at runtime is a single method call.

---

## Concepts

| Term | Meaning |
|---|---|
| **Line Sheet** | A `DialogueLineSheet` ScriptableObject that stores per-speaker audio clips, animator actions, and optional translated text (`PreviewText`) for every NPC line in a graph. |
| **Named Line Sheet** | A pairing of a language label (e.g. `"English"`, `"French"`) with a Line Sheet asset. Stored in the graph's `LineSheets` list. |
| **Active Language** | A string held on `DialogueManager` at runtime. Determines which Named Line Sheet is selected when a graph plays. |
| **PreviewText** | A text field on each line sheet row. When non-empty, it overrides the original NPC node text at runtime — this is the primary mechanism for translating dialogue lines. |

---

## Setup

### 1. Create one Line Sheet per language

For each graph that needs translation, create one `DialogueLineSheet` asset per language. The recommended naming convention is `{GraphName}_{LanguageCode}.asset`:

```
VillagerGraph_EN.asset
VillagerGraph_FR.asset
VillagerGraph_JP.asset
```

You can create sheets via:

- **From the graph editor** — open the sidebar **PROJECT** section and click **Line Sheet Editor**, or click **Line Data** on any NPC node
- **Batch creation** — **Threader → Create & Sync All Line Sheets** scans all graphs and creates missing sheets

### 2. Register sheets on the graph

Open the `DialogueGraph` asset in the Inspector (or use the Graph Editor sidebar). In the **Line Sheets** section, add one entry per language:

| Language | Sheet |
|---|---|
| `English` | `VillagerGraph_EN` |
| `French` | `VillagerGraph_FR` |
| `Japanese` | `VillagerGraph_JP` |

Click **+ Add Language** to add a new row. Each row has a text field for the language label and an object field for the sheet asset.

> The **first entry** in the list is the default/primary language. When no active language is set, or when the requested language is not found, this sheet is used as the fallback.

### 3. Fill in translations

Open each language sheet and populate:

- **PreviewText** — the translated line text for each NPC line row. Leave empty to fall back to the original node text.
- **Audio Clips** — language-specific voice-over clips per speaker per line.
- **Animator Actions** — language-specific animation triggers per speaker per line (e.g. different lip-sync data).

When assigning a new sheet to a non-primary language slot in the Inspector, Threader automatically **seeds** the new sheet from the primary sheet — copying all rows, speaker entries, and animator triggers but leaving audio clips empty. This gives you the correct structure to fill in without manually recreating every row.

### 4. Set the active language at runtime

Before starting any dialogue, call:

```csharp
DialogueManager.Instance.SetActiveLanguage("French");
```

This sets the active language for **all subsequent dialogue playback** — both main conversations and barks. You typically call this once from a settings menu or game startup script.

To read the current language:

```csharp
string current = DialogueManager.Instance.ActiveLanguage;
```

---

## How it works at runtime

### NPC line text

When an NPC node plays, `DialogueManager` resolves text for each line through this chain:

1. Call `graph.GetSheet(activeLanguage)` to get the active language's sheet
2. Look up the sheet row for this node/line via `sheet.LookupRow(nodeGuid, lineIndex)`
3. If the row exists **and** its `PreviewText` is non-empty → use `PreviewText` as the line text
4. Otherwise → use the original `NPCLine.Text` from the node itself
5. Apply `{variable}` token substitution to the resolved text
6. Fire `OnNPCLine` with the final text

This means you can have partially translated sheets — any line without a `PreviewText` gracefully falls back to the source language text on the node.

### Audio clips

Audio clips are resolved from the same active sheet:

1. Call `sheet.Lookup(nodeGuid, lineIndex, speakerName)` to get the `LineSheetSpeakerEntry`
2. If a `Clip` is assigned → play it (3D spatial if a speaker transform is registered, 2D fallback otherwise)
3. If no clip → the line displays with no audio; the runner waits for the typewriter to finish instead

Each language sheet can have completely different clips per speaker — for example, English VO in one sheet and French VO in another.

### Animator actions

Per-line animator actions (e.g. lip-sync triggers) are also resolved from the active sheet's `LineSheetSpeakerEntry.AnimatorActions`. This means you can have different animation data per language if needed.

### Choice text

> **Note:** Choice text localization via `ChoiceSheetRow.PreviewText` is currently used for **editor preview only**. At runtime, choice text comes from `ChoiceData.Text` on the node (with `{variable}` substitution applied). If you need localized choice text at runtime, override the text in your `OnChoiceNode` handler by looking up the active sheet yourself:

```csharp
void HandleChoices(List<ChoiceData> choices)
{
    var graph = /* your reference to the active DialogueGraph */;
    var sheet = graph.GetSheet(DialogueManager.Instance.ActiveLanguage);

    for (int i = 0; i < choices.Count; i++)
    {
        var choice = choices[i];
        if (choice.IsHidden) continue;

        // Look up localized text from the sheet
        string displayText = choice.Text;
        if (sheet != null)
        {
            var choiceRow = sheet.LookupChoiceRow(choice.ChoiceKey?.Split(':')[0], i);
            if (choiceRow != null && !string.IsNullOrEmpty(choiceRow.PreviewText))
                displayText = choiceRow.PreviewText;
        }

        // Use displayText for your UI button
    }
}
```

---

## Switching languages at runtime

Language switching takes effect immediately for all **subsequent** dialogue. Any dialogue already in progress continues with the sheet that was active when it started.

### From a settings menu

```csharp
public class LanguageSettings : MonoBehaviour
{
    public void SetLanguage(string language)
    {
        DialogueManager.Instance.SetActiveLanguage(language);
        // Optionally save the preference
        PlayerPrefs.SetString("Language", language);
        PlayerPrefs.Save();
    }
}
```

### On game startup

```csharp
void Start()
{
    string saved = PlayerPrefs.GetString("Language", "");
    if (!string.IsNullOrEmpty(saved))
        DialogueManager.Instance.SetActiveLanguage(saved);
}
```

### Clearing the active language

To revert to the default (first sheet in the list):

```csharp
DialogueManager.Instance.SetActiveLanguage(null);
// or
DialogueManager.Instance.SetActiveLanguage("");
```

Both `null` and `""` cause `GetSheet` to return the first entry in the `LineSheets` list.

---

## GetSheet fallback chain

`DialogueGraph.GetSheet(language)` resolves through this priority:

| Condition | Returns |
|---|---|
| `LineSheets` is empty | Legacy single `LineSheet` field (for pre-multi-language graphs) |
| `language` is non-empty and a matching entry exists | That entry's sheet |
| `language` is non-empty but no match found | **First sheet** in the list (default language) |
| `language` is null or empty | **First sheet** in the list |

The first entry always acts as the fallback. A graph with at least one entry in `LineSheets` will always return a sheet (unless the first entry's Sheet field is unassigned).

The language string comparison is **exact and case-sensitive**: `"French"` ≠ `"french"`.

---

## Bark graph translation

Bark graphs use the same translation system. When a bark plays, `DialogueManager` calls `graph.GetSheet(activeLanguage)` on the bark graph and resolves text and audio identically to main dialogue. The `OnBark` event receives the resolved (potentially translated) text.

No additional setup is needed beyond adding language sheets to the bark graph's `LineSheets` list.

---

## Editor preview

The graph editor supports a **preview language** dropdown. When a preview language is selected:

- NPC nodes display the `PreviewText` from the matching sheet instead of the original node text
- Player Choice nodes display `ChoiceSheetRow.PreviewText` for each choice

This lets you verify translations directly in the graph editor without entering Play mode.

---

## Migration from legacy single-sheet

Graphs created before multi-language support have a single `LineSheet` field (hidden in the Inspector). When you open such a graph in the Inspector, a warning appears with a **Migrate to Multi-Language Sheets** button.

Clicking the button:

1. Creates a new `NamedLineSheet` entry with language label `"Default"` pointing to the existing sheet
2. Adds it to the `LineSheets` list
3. Clears the legacy `LineSheet` field

After migration, rename `"Default"` to your actual primary language (e.g. `"English"`) and add additional language entries as needed.

---

## Checklist

- [ ] Each graph has one `DialogueLineSheet` asset per supported language
- [ ] The graph's `LineSheets` list has one entry per language, with the primary language **first**
- [ ] Each sheet's `PreviewText` fields contain the translated text for that language
- [ ] Each sheet's audio clips contain the language-specific voice-over
- [ ] `SetActiveLanguage()` is called before dialogue starts (e.g. from a settings menu or game startup)
- [ ] Language strings are consistent everywhere — `"French"` in the graph Inspector must match `"French"` in `SetActiveLanguage("French")`
- [ ] For bark graphs: language sheets are added to the bark graph's `LineSheets` list the same way as main graphs
