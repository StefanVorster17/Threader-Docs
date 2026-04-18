# Saving

Threader owns **no runtime persistence**. Every project has a different save system, so the plugin gives you clean data-extraction APIs you can feed into whatever you're already using (PlayerPrefs, JSON, a database, a binary serializer, Steam Cloud, etc.).

There are three separate things to save when a player saves their game mid-playthrough:

| What | Where it lives | How to save/load |
|---|---|---|
| Variable values | `DialogueVariables` assets | Read with `GetBool`/`GetInt`/`GetString`, write back with `SetBool`/`SetInt`/`SetString` |
| NPC entry points | `IDialogueActor.ActiveEntryPointKey` per actor | Copy the string; restore with `SetEntryPoint` |
| Choice history | `DialogueChoiceHistory` static class | `GetSaveData()` / `LoadSaveData()` |

---

## 1. Saving variable values

`DialogueVariables` is a `ScriptableObject`. Its runtime dictionaries are in-memory only and reset whenever the asset is unloaded or the game exits. After loading a save, repopulate the dictionaries before any dialogue starts.

```csharp
// ─── Saving ──────────────────────────────────────────────────────────────
DialogueVariables vars = dialogueManager.VariablesList[0]; // or find by name

// Booleans
bool hasSword     = vars.GetBool("hasSword");
bool talkedToKing = vars.GetBool("talkedToKing");

// Integers
int relationshipScore = vars.GetInt("relationship");
int gold              = vars.GetInt("gold");

// Strings
string playerName = vars.GetString("playerName");

// Write to your preferred system — PlayerPrefs shown for simplicity
PlayerPrefs.SetInt("hasSword",     hasSword ? 1 : 0);
PlayerPrefs.SetInt("relationship", relationshipScore);
PlayerPrefs.SetString("playerName", playerName);

// ─── Loading ──────────────────────────────────────────────────────────────
vars.SetBool  ("hasSword",     PlayerPrefs.GetInt("hasSword",     0) == 1);
vars.SetInt   ("relationship", PlayerPrefs.GetInt("relationship", 0));
vars.SetString("playerName",   PlayerPrefs.GetString("playerName", ""));

// No need to call InitRuntime() — SetBool/SetInt/SetString call EnsureInit() automatically
```

### Multiple assets

If you have more than one `DialogueVariables` asset assigned to `DialogueManager.VariablesList`, iterate all of them. You can use the asset name (`.name`) or a custom naming convention as save keys:

```csharp
foreach (var asset in dialogueManager.VariablesList)
{
    foreach (var variable in asset.Variables)
    {
        string saveKey = $"{asset.name}.{variable.Name}";

        switch (variable.Type)
        {
            case VariableType.Bool:
                int bit = asset.GetBool(variable.Name) ? 1 : 0;
                data[saveKey] = bit.ToString();
                break;
            case VariableType.Int:
                data[saveKey] = asset.GetInt(variable.Name).ToString();
                break;
            case VariableType.String:
                data[saveKey] = asset.GetString(variable.Name);
                break;
        }
    }
}
```

### New game (reset to defaults)

Call `InitRuntime()` explicitly to copy the asset's design-time defaults back into the runtime dictionaries. This is the canonical "new game" reset:

```csharp
void StartNewGame()
{
    foreach (var asset in dialogueManager.VariablesList)
        asset.InitRuntime();

    DialogueChoiceHistory.Clear();

    foreach (var actor in FindObjectsOfType<MonoBehaviour>()
                              .OfType<IDialogueActor>())
        actor.ResetEntryPoint();
}
```

> `InitRuntime()` resets **all** variables in that asset to their design-time defaults. If you only want to reset some variables, use individual `SetBool`/`SetInt`/`SetString` calls.

---

## 2. Saving entry points

Each `IDialogueActor` (a `DialogueTrigger` or `NPCDialogue` in your scene) stores a single string — `ActiveEntryPointKey`. Empty or null means "use the graph's start node."

```csharp
// ─── Saving ──────────────────────────────────────────────────────────────
var actor = npcGameObject.GetComponent<IDialogueActor>();
string key = actor.ActiveEntryPointKey; // may be null or ""
// Treat null and "" identically when saving — both mean "default start"
PlayerPrefs.SetString("npc_merchant_ep", key ?? "");

// ─── Loading ──────────────────────────────────────────────────────────────
string savedKey = PlayerPrefs.GetString("npc_merchant_ep", "");
actor.SetEntryPoint(savedKey); // "" or null restores the default
```

### Saving entry points for all scene actors

If you have many NPCs, collect them at load time and save/restore by a stable identifier (GameObject name, a unique ID component, etc.):

```csharp
// Save
var actors = FindObjectsOfType<MonoBehaviour>().OfType<IDialogueActor>();
foreach (var actor in actors)
{
    var mb = (MonoBehaviour)actor;
    string saveKey = "ep_" + mb.gameObject.name;
    PlayerPrefs.SetString(saveKey, actor.ActiveEntryPointKey ?? "");
}

// Load (call before first dialogue in new scene)
foreach (var actor in FindObjectsOfType<MonoBehaviour>().OfType<IDialogueActor>())
{
    var mb = (MonoBehaviour)actor;
    string saveKey = "ep_" + mb.gameObject.name;
    string stored  = PlayerPrefs.GetString(saveKey, "");
    actor.SetEntryPoint(stored);
}
```

> `FindObjectsOfType` is slow. Cache the list at `Start()` if performance matters, or keep a `Dictionary<string, IDialogueActor>` keyed by a stable ID.

---

## 3. Saving choice history

`DialogueChoiceHistory` tracks which choices the player has made during the session. It is used by nodes to:

- Style "already chosen" options differently in a custom UI
- Query from C# (e.g. quest system checking if the player asked a specific question)

The static class stores a `HashSet<string>`. Each entry is a key in the format `{choiceNodeGuid}:{choiceIndex}`.

```csharp
// ─── Saving ──────────────────────────────────────────────────────────────
string[] visitedChoices = DialogueChoiceHistory.GetSaveData();
// visitedChoices is a snapshot — serialize it however you like:
string json = JsonUtility.ToJson(new StringArrayWrapper { items = visitedChoices });
PlayerPrefs.SetString("choiceHistory", json);

// ─── Loading ──────────────────────────────────────────────────────────────
string json = PlayerPrefs.GetString("choiceHistory", "");
if (!string.IsNullOrEmpty(json))
{
    string[] keys = JsonUtility.FromJson<StringArrayWrapper>(json).items;
    DialogueChoiceHistory.LoadSaveData(keys);
}

// ─── New game ─────────────────────────────────────────────────────────────
DialogueChoiceHistory.Clear();
```

### Key stability

The key format is `{choiceNodeGuid}:{choiceIndex}`. It is **stable** as long as:
- The Player Choice node is not deleted and recreated
- Choices within that node are not reordered

If you reorganize choices in the editor, old save data keys for that node become orphaned (they are just ignored). Existing choices won't be marked as visited. Deletions and reorderings only affect the specific node touched.

### `LoadSaveData` merges, not replaces

`LoadSaveData` *adds* the supplied keys to the existing set. If you want a clean slate before loading, call `Clear()` first:

```csharp
DialogueChoiceHistory.Clear();
DialogueChoiceHistory.LoadSaveData(savedKeys);
```

### Querying in C#

```csharp
// From a quest script — check if player asked a specific question
bool askedAboutSword = DialogueChoiceHistory.IsVisited("abc123guid:1");
```

To get the key for a specific choice, look up the node GUID on the `DialogueGraph.asset` in the Inspector and note the choice index (0-based order in the card list).

---

## 4. ConditionStore (if using it)

If you use `ConditionStoreProvider` (see [Conditions](conditions.md)), its static dictionary is also in-memory only. Save/restore it the same way as variable values:

```csharp
// ─── Saving ──────────────────────────────────────────────────────────────
// ConditionStore does not expose an enumerator — save the keys you care about manually
int gold = ConditionStore.GetInt("Gold");
ConditionStore.TryGet("QuestState", out var quest);
// … write to save data …

// ─── Loading ──────────────────────────────────────────────────────────────
ConditionStore.SetInt("Gold",       savedGold);
ConditionStore.Set   ("QuestState", savedQuestState);
```

Alternatively, repopulate `ConditionStore` from your existing game-state objects (inventory, quest manager, etc.) rather than saving/loading it separately — it's just a cache that bridges game state into the condition system.

---

## 5. Delegate-registered conditions (ConditionService)

If your conditions are registered via `ConditionService.Register`, the delegates themselves don't need to be saved — they re-register in `Awake` when the scene loads. The **state they read** (inventory, quest flags, etc.) is what you save in your main save system.

Make sure delegates are registered before any dialogue starts. If you load a save and immediately trigger dialogue before `Awake` runs on the registering component, conditions will fall back to `WhenMissing`. Call `ConditionService.Register` in `Awake`, not `Start`.

---

## Full save/load example

```csharp
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Self-contained save/load manager for a single-save-slot game.
/// Attach to the same GameObject as DialogueManager.
/// </summary>
public class DialogueSaveManager : MonoBehaviour
{
    [SerializeField] private DialogueManager dialogueManager;
    [SerializeField] private List<MonoBehaviour> actors; // drag NPCDialogue/DialogueTrigger here

    const string PREF_VARS    = "dlg_vars";
    const string PREF_EP      = "dlg_ep_{0}";
    const string PREF_HISTORY = "dlg_history";

    public void Save()
    {
        var blob = new SaveBlob();

        // — Variables ——————————————————————————————————
        foreach (var asset in dialogueManager.VariablesList)
        {
            foreach (var v in asset.Variables)
            {
                if (string.IsNullOrEmpty(v.Name)) continue;
                string key = $"{asset.name}|{v.Name}";
                string val = v.Type switch
                {
                    VariableType.Bool   => asset.GetBool(v.Name) ? "1" : "0",
                    VariableType.Int    => asset.GetInt(v.Name).ToString(),
                    VariableType.String => asset.GetString(v.Name),
                    _                  => ""
                };
                blob.variables.Add(new KV { key = key, val = val });
            }
        }

        // — Entry points ———————————————————————————————
        foreach (var mb in actors)
        {
            if (mb is not IDialogueActor actor) continue;
            blob.entryPoints.Add(new KV
            {
                key = mb.gameObject.name,
                val = actor.ActiveEntryPointKey ?? ""
            });
        }

        // — Choice history —————————————————————————————
        blob.choiceHistory = DialogueChoiceHistory.GetSaveData();

        PlayerPrefs.SetString(PREF_VARS, JsonUtility.ToJson(blob));
        PlayerPrefs.Save();
    }

    public void Load()
    {
        string json = PlayerPrefs.GetString(PREF_VARS, "");
        if (string.IsNullOrEmpty(json)) return;

        var blob = JsonUtility.FromJson<SaveBlob>(json);

        // — Variables ——————————————————————————————————
        var assetMap = new Dictionary<string, DialogueVariables>();
        foreach (var asset in dialogueManager.VariablesList)
            assetMap[asset.name] = asset;

        foreach (var kv in blob.variables)
        {
            int sep = kv.key.IndexOf('|');
            if (sep < 0) continue;
            string assetName = kv.key[..sep];
            string varName   = kv.key[(sep + 1)..];
            if (!assetMap.TryGetValue(assetName, out var asset)) continue;

            var def = asset.Variables.Find(v => v.Name == varName);
            if (def == null) continue;

            switch (def.Type)
            {
                case VariableType.Bool:   asset.SetBool  (varName, kv.val == "1"); break;
                case VariableType.Int:    _ = int.TryParse(kv.val, out int i);
                                          asset.SetInt(varName, i); break;
                case VariableType.String: asset.SetString(varName, kv.val); break;
            }
        }

        // — Entry points ———————————————————————————————
        var actorMap = new Dictionary<string, IDialogueActor>();
        foreach (var mb in actors)
            if (mb is IDialogueActor a) actorMap[mb.gameObject.name] = a;

        foreach (var kv in blob.entryPoints)
            if (actorMap.TryGetValue(kv.key, out var actor))
                actor.SetEntryPoint(kv.val);

        // — Choice history —————————————————————————————
        DialogueChoiceHistory.Clear();
        DialogueChoiceHistory.LoadSaveData(blob.choiceHistory);
    }

    public void ResetForNewGame()
    {
        foreach (var asset in dialogueManager.VariablesList)
            asset.InitRuntime();

        foreach (var mb in actors)
            if (mb is IDialogueActor actor) actor.ResetEntryPoint();

        DialogueChoiceHistory.Clear();

        PlayerPrefs.DeleteKey(PREF_VARS);
        PlayerPrefs.Save();
    }

    // — Serialisable types ─────────────────────────────
    [System.Serializable] class KV { public string key, val; }

    [System.Serializable]
    class SaveBlob
    {
        public List<KV> variables   = new();
        public List<KV> entryPoints = new();
        public string[] choiceHistory = System.Array.Empty<string>();
    }
}
```

---

## When to call Save/Load

| Event | Action |
|---|---|
| Player opens save menu | Call `Save()` |
| Player selects a save file | Call `Load()`, then load your scene and start gameplay |
| New game | Call `ResetForNewGame()` before starting |
| Scene transition (same save) | Entry points are scene-persisted by their component's serialized field — no extra work unless NPCs are instantiated at runtime |
| Dialogue ends | No action needed — changes happen immediately in memory |

---

## What Threader does NOT save for you

| Item | Reason |
|---|---|
| `ConditionStore` values | Too project-specific; repopulate from your own game-state objects |
| Which choices are locked/hidden | Computed live from current variable values and condition results |
| Dialogue "in progress" state | No mid-dialogue saves — always save at a safe point between conversations |
| Audio or animation state | Managed by Unity and your own systems |
