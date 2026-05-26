# Audio System

Threader abstracts all audio playback behind an `IDialogueAudioProvider` interface. This means you can use Unity's built-in `AudioSource`, FMOD Studio, Wwise, or any custom backend — without changing a single line of Threader's core code.

---

## How it works

Every time a dialogue line plays, `DialogueManager` calls:

```csharp
_audio.Play(clip, lineKey, speakerName, worldPosition);
```

`_audio` is whatever `IDialogueAudioProvider` is active. By default it is a `UnityAudioProvider` created automatically. You can replace it by assigning a custom provider to the **Audio Provider Override** slot on `DialogueManager`.

After calling `Play`, the dialogue coroutine waits:

```csharp
yield return new WaitWhile(() => _audio.IsPlaying && !DialogueUI.SkipLineRequested);
_audio.Stop();
```

This means the system waits for your provider to report that playback has finished before moving to the next line — whether that is a Unity `AudioSource`, an FMOD instance, or a Wwise event.

---

## Default provider — UnityAudioProvider

When no override is assigned, `DialogueManager` adds a `UnityAudioProvider` component to itself automatically. No setup is needed for standard Unity audio.

`UnityAudioProvider` uses two `AudioSource` components internally:

| Source | Used when |
|---|---|
| **2D source** | No speaker transform is registered for the current line |
| **Spatial source** | A speaker transform is registered — repositioned to the speaker's world position each line (`spatialBlend = 1`, `rolloffMode = Linear`) |

You can also add `UnityAudioProvider` manually via **Add Component → Threader → Unity Audio Provider** if you need to configure the 2D source in the Inspector.

---

## Line Keys

A **Line Key** is a short, stable identifier assigned to each line in the Line Sheet editor. It is the bridge between three separate systems:

```
Dialogue tool (Threader)
        ↓  LineKey
VO recording spreadsheet
        ↓  same key
Audio middleware event bank (FMOD / Wwise)
```

When a line plays, both the `AudioClip` (for Unity) and the `LineKey` (for middleware) are passed to `IDialogueAudioProvider.Play()`. The active provider decides what to do with each:

| Provider | Uses `clip` | Uses `lineKey` |
|---|---|---|
| `UnityAudioProvider` | Yes — plays the `AudioClip` directly | Ignored |
| `FMODAudioProvider` | Ignored | Constructs FMOD event path: `event:/Dialogue/{lineKey}` |
| `WwiseAudioProvider` | Ignored | Posts Wwise event: `Play_{lineKey}` (prefix configurable) |
| Custom provider | Up to you | Up to you |

### Naming conventions

Line Keys should be:

- **SCREAMING_SNAKE_CASE** — `GUARD_GREET_01`, `CAT_LADY_FIND_CAT_002`
- **Globally unique** within your project — they are used as event names in your audio bank
- **Descriptive and stable** — once a key is set and the VO is recorded against it, changing it means renaming events in your middleware project and re-exporting
- **Shared across languages** — all language sheets for the same graph use the same Line Key per line; only the `AudioClip` and `PreviewText` differ per language

A common format is:

```
{NPC_NAME}_{SCENE_OR_TOPIC}_{LINE_NUMBER}
```

Examples:

| Key | NPC | Topic |
|---|---|---|
| `GUARD_GATE_GREET_01` | Gate Guard | Greeting at the gate |
| `CAT_LADY_FIND_CAT_001` | Cat Lady | First line of "find my cat" quest |
| `MERCHANT_HAGGLE_REFUSE_03` | Merchant | Third refusal line during haggling |

Pad numbers to at least two digits so alphabetical sorting in spreadsheets and audio editors stays consistent.

### Setting Line Keys

Line Keys are set in the **Line Sheet Editor**, which you open from the graph editor sidebar (**PROJECT → Line Sheet Editor**) or by clicking **Line Data** on any NPC node.

Each line has a **Line Key** field above the speaker entries. Set it once — when you sync the sheet to other languages, the key propagates automatically to every language sheet. You never need to re-enter it.

### The VO pipeline

For large productions the recommended workflow is:

1. **Author dialogue** in the graph editor — write lines, set speaker names
2. **Sync line sheets** — run **Threader → Create & Sync All Line Sheets** to generate rows for every line
3. **Assign Line Keys** in the Line Sheet Editor — one key per line, following your naming convention
4. **Export to spreadsheet** — share a CSV or sheet with your VO director; the Line Key column maps every line to a recording session slot
5. **Record VO** — the director labels each take with the corresponding Line Key
6. **Import into middleware** — in FMOD or Wwise, create one event per Line Key (batch import from the naming convention); the provider constructs the path from the key at runtime
7. **Assign clips (optional)** — for Unity AudioSource projects, drag the recorded clips back into the Line Sheet speaker entries; for middleware projects, leave Clip empty and rely entirely on the key

Steps 1–3 and 7 happen in Unity. Steps 4–6 happen outside Unity. The Line Key is the handshake between them.

---

## Using a middleware provider

### FMOD

1. Install **FMOD for Unity** from the FMOD website. The package adds the `FMOD_INSTALLED` scripting define symbol automatically.
2. Copy `Assets/Threader/Samples/FMOD/FMODAudioProvider.cs` into your project (or leave it in Samples — it compiles once `FMOD_INSTALLED` is defined).
3. Add **FMODAudioProvider** as a component on the `DialogueManager` GameObject.
4. Drag it into the **Audio Provider Override** slot on `DialogueManager`.
5. Set the **Event Path Prefix** field to match your FMOD Studio project's folder structure (e.g. `event:/Dialogue`).

At runtime, a line with `LineKey = "GUARD_GATE_GREET_01"` and prefix `event:/Dialogue` plays `event:/Dialogue/GUARD_GATE_GREET_01`. Name your FMOD events to match and they resolve automatically.

### Wwise

1. Install the **Wwise Unity Integration** from Audiokinetic. The package adds the `AK_WWISE_UNITY_INTEGRATION` scripting define symbol automatically.
2. Copy `Assets/Threader/Samples/WWISE/WwiseAudioProvider.cs` into your project.
3. Add **WwiseAudioProvider** as a component on the `DialogueManager` GameObject.
4. Drag it into the **Audio Provider Override** slot on `DialogueManager`.
5. Set the **Event Prefix** field (default: `Play_`). A line with `LineKey = "GUARD_GATE_GREET_01"` will post event `Play_GUARD_GATE_GREET_01`.
6. Optionally set a **Volume RTPC Name** to wire dialogue volume to a Wwise RTPC.

### Custom backend

Implement `IDialogueAudioProvider` on a `MonoBehaviour`:

```csharp
using UnityEngine;
using Threader;

public class MyAudioProvider : MonoBehaviour, IDialogueAudioProvider
{
    public float Volume { get; set; } = 1f;
    public bool IsPlaying { get; private set; }

    public void Play(AudioClip clip, string lineKey, string speakerName, Vector3? worldPosition)
    {
        // resolve and play audio using lineKey, clip, or both
        IsPlaying = true;
    }

    public void Stop()
    {
        IsPlaying = false;
    }
}
```

The only contract that must be correct is `IsPlaying` — it must return `true` while audio is playing and `false` when it has finished. Returning `false` too early cuts lines short; never returning `false` hangs the dialogue permanently.

---

## Spatial audio

When a speaker's transform is registered with `DialogueManager.RegisterSpeaker`, its world position is passed as `worldPosition` to `Play()`. Providers use this for 3D positioning:

- **UnityAudioProvider** repositions a child `AudioSource` to that position
- **FMODAudioProvider** calls `instance.set3DAttributes(RuntimeUtils.To3DAttributes(worldPosition.Value))`
- **WwiseAudioProvider** moves a registered child `GameObject` to that position before posting the event

When no speaker transform is found, `worldPosition` is `null` and the provider falls back to 2D playback.

Speaker transforms are registered automatically by `NPCDialogue` in `Start()`. You only need to register manually for NPCs added dynamically at runtime via `DialogueManager.Instance.RegisterSpeaker(name, transform)`.

---

## Checklist

- For Unity audio: leave **Audio Provider Override** empty — `UnityAudioProvider` is created automatically
- For FMOD/Wwise: add the sample provider component to the `DialogueManager` GameObject and assign it to **Audio Provider Override**
- Line Keys are set once in the Line Sheet Editor and propagate to all language sheets automatically
- Line Keys must match the event names in your middleware bank exactly (accounting for any prefix)
- Custom providers must implement `IsPlaying` correctly — the dialogue coroutine blocks on it
- `Play()` may receive a null `clip` (middleware) or an empty `lineKey` (Unity-only) — handle both gracefully
