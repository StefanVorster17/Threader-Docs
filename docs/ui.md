# UI

Threader ships with a built-in `DialogueUI` component that powers the demo scene. It is there to get something on screen immediately — **it is not intended to be your final UI**. Most projects will replace it entirely with a custom implementation to match their own visual style and UI framework.

> **UI Toolkit is not required for runtime UI.** The built-in `DialogueUI` uses UI Toolkit, but `DialogueManager` is fully event-driven and has no dependency on it. You can drive a Unity Canvas (uGUI), a third-party UI framework, or any other system by subscribing to the runtime events described in [Building a custom UI](#building-a-custom-ui). Both approaches are shown with code examples below.

---

## The built-in DialogueUI

`DialogueUI` is a **Unity UI Toolkit** component (`UIDocument`-based). It is `[RequireComponent(typeof(UIDocument))]`, so it must live on a GameObject that already has a `UIDocument`.

### What it does

- Displays NPC lines with a configurable typewriter effect
- Shows speaker name
- Renders player choice buttons with staggered fade-in, visited/locked states, and an animated dismiss on selection
- Handles **Space** to skip the current typewriter animation or advance the line
- Handles **Escape** to call `DialogueManager.CancelDialogue()` (respects per-node **Prevent Dialogue Exit**)
- Manages optional **Skip** and **Exit** HUD elements automatically (shown/hidden as dialogue state changes)

### Inspector fields

| Field | Default | Description |
|---|---|---|
| **Choice Button Template** | none | A `VisualTreeAsset` UXML template used for each choice button. If left blank, plain `Button` elements are created. |
| **Chars Per Second** | `10` | Typewriter speed. |
| **Punctuation Pause** | `0.15` | Extra seconds added after `.` `,` `!` `?` `;` |
| **Choice Stagger Delay** | `0.5` | Seconds between each choice card fading in. |

### Expected UXML element names

`DialogueUI` queries these named elements from the `UIDocument` root at startup:

| Name | Type | Purpose |
|---|---|---|
| `Root` | `VisualElement` | Top-level container. Shown/hidden when dialogue starts/ends. |
| `Text` | `Label` | Receives the typewritten line text. |
| `SpeakerName` | `Label` | Displays the speaking character's name. Hidden automatically when blank. |
| `ChoicesContainer` | `ScrollView` | Choice buttons are instantiated and added here. |
| `ScreenFader` | `VisualElement` | Reserved for fades. Not filled automatically — available for your USS transitions. |
| `Skip` | `VisualElement` | Shown during NPC lines; hidden during choice nodes. Optional — omit from UXML to disable. |
| `Exit` | `VisualElement` | Shown when `DialogueManager.CanCancel` is `true` (dialogue is active and the current node allows cancellation). Optional — omit from UXML to disable. |

If any element is missing from the UXML, that feature silently does nothing rather than throwing an error.

### It is for the demo scene

The demo scene ships with `DialogueUI` already set up. **Remove or replace it in your own scene.** The `DialogueUI` component is not required by any other part of Threader — the dialogue runner operates entirely through C# events described below.

---

## Building a custom UI

You only need three event subscriptions and one method call. Everything else is up to you — uGUI (Canvas), UI Toolkit, third-party frameworks, whatever your project uses.

### Canvas / uGUI example

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class MyDialogueUI : MonoBehaviour
{
    [SerializeField] GameObject    _panel;
    [SerializeField] TMP_Text      _textLabel;
    [SerializeField] TMP_Text      _speakerLabel;
    [SerializeField] Transform     _choicesContainer;
    [SerializeField] Button        _choiceButtonPrefab;

    void OnEnable()
    {
        DialogueManager.Instance.OnNPCLine     += HandleLine;
        DialogueManager.Instance.OnChoiceNode  += HandleChoices;
        DialogueManager.Instance.OnDialogueEnd += HandleEnd;
    }

    void OnDisable()
    {
        DialogueManager.Instance.OnNPCLine     -= HandleLine;
        DialogueManager.Instance.OnChoiceNode  -= HandleChoices;
        DialogueManager.Instance.OnDialogueEnd -= HandleEnd;
    }

    void HandleLine(NPCLine line)
    {
        _panel.SetActive(true);
        _speakerLabel.text = line.SpeakerName;
        _textLabel.text    = line.Text;

        foreach (Transform child in _choicesContainer)
            Destroy(child.gameObject);
    }

    void HandleChoices(List<ChoiceData> choices)
    {
        foreach (Transform child in _choicesContainer)
            Destroy(child.gameObject);

        for (int i = 0; i < choices.Count; i++)
        {
            var choice = choices[i];
            if (choice.IsHidden) continue;

            int capturedIndex = i;
            var btn = Instantiate(_choiceButtonPrefab, _choicesContainer);
            btn.GetComponentInChildren<TMP_Text>().text = choice.Text;
            btn.interactable = !choice.IsLocked;
            btn.onClick.AddListener(() => DialogueManager.Instance.SelectChoice(capturedIndex));
        }
    }

    void HandleEnd()
    {
        _panel.SetActive(false);

        foreach (Transform child in _choicesContainer)
            Destroy(child.gameObject);
    }
}
```

Assign your Canvas panel, TextMeshPro labels, choices container `Transform`, and a Button prefab in the Inspector. No UI Toolkit or UIDocument required.

### UI Toolkit example

This is what the built-in `DialogueUI` uses internally. If you prefer to stay in UI Toolkit:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;

[RequireComponent(typeof(UIDocument))]
public class MyDialogueUI : MonoBehaviour
{
    Label          _textLabel;
    Label          _speakerLabel;
    VisualElement  _panel;
    VisualElement  _choicesContainer;

    void Awake()
    {
        var root       = GetComponent<UIDocument>().rootVisualElement;
        _panel             = root.Q<VisualElement>("Root");
        _textLabel         = root.Q<Label>("Text");
        _speakerLabel      = root.Q<Label>("SpeakerName");
        _choicesContainer  = root.Q<VisualElement>("Choices");
        _panel.style.display = DisplayStyle.None;
    }

    void OnEnable()
    {
        DialogueManager.Instance.OnNPCLine     += HandleLine;
        DialogueManager.Instance.OnChoiceNode  += HandleChoices;
        DialogueManager.Instance.OnDialogueEnd += HandleEnd;
    }

    void OnDisable()
    {
        DialogueManager.Instance.OnNPCLine     -= HandleLine;
        DialogueManager.Instance.OnChoiceNode  -= HandleChoices;
        DialogueManager.Instance.OnDialogueEnd -= HandleEnd;
    }

    void HandleLine(NPCLine line)
    {
        // line.SpeakerName — resolved speaker name (graph default applied when node Speaker is blank)
        // line.Text — fully resolved string (variable tokens already substituted)
        // line.Clip — AudioClip, may be null
        _panel.style.display  = DisplayStyle.Flex;
        _speakerLabel.text    = line.SpeakerName;
        _textLabel.text       = line.Text;
        _choicesContainer.Clear();
    }

    void HandleChoices(List<ChoiceData> choices)
    {
        _choicesContainer.Clear();
        for (int i = 0; i < choices.Count; i++)
        {
            var choice = choices[i];
            if (choice.IsHidden) continue;  // filter hidden choices before rendering

            int capturedIndex = i;
            var btn = new Button(() => DialogueManager.Instance.SelectChoice(capturedIndex));
            btn.text = choice.Text;
            btn.SetEnabled(!choice.IsLocked);
            _choicesContainer.Add(btn);
        }
    }

    void HandleEnd()
    {
        _panel.style.display = DisplayStyle.None;
        _choicesContainer.Clear();
    }
}
```

### The line advance contract

The runner handles line timing internally — **you do not call `Continue()`**. After `OnNPCLine` fires the runner:

1. If the line has an `AudioClip`, waits for the clip to finish playing.
2. Otherwise, waits until `DialogueUI.IsTyping` is `false`.
3. Waits `linePause` seconds (set on the `DialogueManager` Inspector).
4. Checks `DialogueUI.SkipLineRequested` (set by Space key in the built-in UI).
5. Calls `runner.Continue()` itself and moves to the next node.

**If you have no `DialogueUI` in the scene**, step 2 resolves immediately (the static `IsTyping` flag is `false` by default), so lines advance after only the `linePause` delay. If your custom UI has a typewriter and you want the runner to wait for it, you have two options:

- Set `linePause` long enough to cover the typewriter duration (simple, not dynamic).
- Provide Space-to-advance input on your own — the runner will hold at step 3/4 until `linePause` elapses regardless.

### Speaker name

`OnNPCLine` carries the speaker name in `line.SpeakerName` alongside `line.Text` and `line.Clip`. Use it directly in your `HandleLine` handler:

```csharp
void HandleLine(NPCLine line)
{
    _speakerLabel.text = line.SpeakerName;
    _textLabel.text    = line.Text;
}
```

### Cancelling dialogue

`DialogueManager.CancelDialogue()` immediately ends active dialogue. Call it from any button click or key handler in your custom UI:

```csharp
// Example: Escape key in Update(), or a cancel button callback
DialogueManager.Instance.CancelDialogue();
```

The call is a no-op if no dialogue is running, or if the current node has **Prevent Dialogue Exit** enabled. Check `DialogueManager.Instance.CanCancel` first if you want to show or hide a cancel button reactively.

### Choices — `SelectChoice`

```csharp
// Call this from whichever button the player presses
DialogueManager.Instance.SelectChoice(index);
```

`index` is the **0-based position** in the list passed to `OnChoiceNode` — not the position of the button in your layout (which may differ if you filtered out hidden choices). Store the original index when creating buttons, as shown in the example above.

### Show / hide the panel from the Inspector

For simple show/hide without any C#, use the `UnityEvent` fields on `DialogueManager`:

| Field | Fires when |
|---|---|
| **On Dialogue Started** | Any conversation begins |
| **On Dialogue Ended** | Any conversation ends |

Drag your UI panel GameObject into either slot and call `SetActive(true)` / `SetActive(false)`.

---

## Blocking player input during dialogue

Implement `IDialogue` on any component that should pause when dialogue is active (player movement, hotkeys, inventory, etc.):

```csharp
public class PlayerController : MonoBehaviour, IDialogue
{
    void OnEnable()  => DialogueManager.Instance.RegisterBlockable(this);
    void OnDisable() => DialogueManager.Instance.UnregisterBlockable(this);

    public void Block()   { /* disable movement */ }
    public void Unblock() { /* re-enable movement */ }
}
```

`RegisterBlockable` in `OnEnable`, `UnregisterBlockable` in `OnDisable`. The manager calls `Block()` on all registered blockables when `StartDialogue` is called and `Unblock()` when dialogue ends.

---

## Camera transitions before the first line

Implement `IDialogueReady` alongside `IDialogue` when your UI or camera needs a moment (a fade, a dolly move) before the first line should display. The manager waits on `IsDialogueReady` before starting the runner:

```csharp
public class DialogueCamera : MonoBehaviour, IDialogue, IDialogueReady
{
    bool _ready;
    public bool IsDialogueReady => _ready;

    public void Block()
    {
        _ready = false;
        StartCoroutine(TransitionIn());
    }

    IEnumerator TransitionIn()
    {
        // play camera cut, fade, etc.
        yield return new WaitForSeconds(0.4f);
        _ready = true;
    }

    public void Unblock() { /* reverse transition */ }
}
```

---

## Camera look-at during dialogue

Implement `IDialogueFocus` on your camera controller. When a graph has **Look At Speaker** enabled and a speaker transform is registered, the manager calls `FocusOn` each time a new NPC node is reached:

```csharp
public class DialogueCamera : MonoBehaviour, IDialogue, IDialogueFocus
{
    public void FocusOn(Transform target)
    {
        // smoothly look at target, or cut — your choice
    }

    public void ReleaseFocus()
    {
        // return to default camera behaviour
    }
}
```

Enable **Look At Speaker** in the **Graph Editor** sidebar (the toggle is in the graph settings panel on the left), then make sure every NPC has their **Speaker Name** set on `NPCDialogue` so the manager can resolve their transform.
