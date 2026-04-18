# Roadmap

This page tracks what's coming to Threader, what's being considered, and what's intentionally out of scope. It's updated as plans change.

If you have a feature request, raise it on the [GitHub repository](https://github.com/StefanVorster17/Threader-Docs/issues) — community requests that get traction move up into a planned release.

---

## Released

### v1.0 — April 16, 2026

Initial release. Full feature set including the visual graph editor, 15 node types, variable system, conditions, entry points, bark system, sub-graph system, line sheet system with multi-language support, node templates, bookmarks, export script, type-aware variable inspector, custom right-click context menu, and all associated components and APIs. See the [Changelog](changelog.md) for the complete list.

---

## Considering

Features on the radar but not yet committed to a version. These may or may not ship depending on scope, community interest, and architectural fit.

### Unity Localization package integration

Threader ships built-in multi-language support via the `LineSheets` list and `SetActiveLanguage()`. This covers text, audio, and animator actions per language with no external dependencies.

An optional bridge assembly that reads from `LocalizedString` fields via Unity's official Localization package is being considered for projects that already use it. This would be compiled only when the Unity Localization package is detected (`#if` conditional), so existing buyers are never affected and no unwanted dependencies are introduced.

**Status:** Under consideration. The built-in system covers most use cases; the bridge would be a convenience integration for teams already invested in Unity Localization.

### Abstract audio provider

Threader currently plays audio through Unity's built-in `AudioSource`. Rather than hard-coding integrations for specific middleware (FMOD, Wwise, etc.) — which creates a maintenance burden every time those systems update — the plan is to introduce an `IAudioProvider` interface that Threader calls instead of `AudioSource` directly.

- The built-in `AudioSource` provider would ship as the default implementation
- Users implement the interface for their audio system (FMOD, Wwise, Master Audio, or any custom solution)
- Example adapters may be included as optional samples, but third-party dependencies are never required

This follows the same pattern as `ConditionProvider` for custom conditions, and could extend to other "bring your own system" integration points (TTS, lip sync, etc.) in the future.

**Status:** Under consideration. Depends on community demand for middleware audio support.

---

## Out of scope

These are features that have been considered and deliberately won't be added to Threader core. If you need them, alternatives are noted.

### Built-in branching story format export (Ink, Yarn, Twine)

Threader's graph format is Unity-native and optimised for runtime execution, not text-based story formats. Converting between the two would require lossy translation of variable scopes, entry points, and events. Use Yarn Spinner or ink directly if you need a text-based authoring workflow.

### Multiplayer / networked dialogue synchronisation

Syncing dialogue state across a network is highly game-specific (what to sync, who is the authority, how to handle latency). Threader's runtime is single-player by design. Wire your own netcode layer on top of `OnNPCLine`, `OnChoiceNode`, and `SelectChoice` if you need multiplayer dialogue.
