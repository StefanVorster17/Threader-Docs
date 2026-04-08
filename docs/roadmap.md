# Roadmap

This page tracks what's coming to Threader, what's being considered, and what's intentionally out of scope. It's updated as plans change.

If you have a feature request, raise it on the [GitHub repository](https://github.com/StefanVorster17/Threader-Docs/issues) — community requests that get traction move up into a planned release.

---

## Upcoming

### v1.0.1

Editor debugging improvements: GUID tooling (show, search, copy), validator enhancements with new checks and live updates, enriched runtime error messages, and runtime performance optimisations. See the [Changelog](changelog.md) for the full list on release.

---

## Planned

### v1.1

#### Sub-Graph support

Call a separate `SubGraph` asset from within a running dialogue graph, then return to the calling graph when it ends.

**Design intent:**
- A new `SubGraph` graph type that carries no actor, no LookAt setting, and fires no `OnDialogueStarted` / `OnDialogueEnded` events of its own
- The calling graph passes its actor, LookAt state, and blocked state into the sub-graph automatically
- A **Call Graph node** in the main graph triggers the call and resumes on the next connected node when the sub-graph ends
- Existing graphs are completely unaffected — this is additive, nothing breaks

**Use cases this enables:**
- Shared "rumour" or "ambient" dialogue that any NPC can invoke mid-conversation
- Reusable shop, trade, or minigame dialogue sequences called from multiple graphs
- Breaking very large story graphs into smaller chapter graphs that chain together

**Status:** Architecture designed. Implementation planned for v1.1.

---

## Considering

Features on the radar but not yet committed to a version. These may or may not ship depending on scope, community interest, and architectural fit.

### Localization support

Allow all dialogue text to be translated and swapped at runtime by language.

**Two approaches are being considered:**

**Option A — Built-in (no dependency)**
Threader ships its own `LocalizationTable` ScriptableObject: a simple line ID → translated string mapping. Swap the active table at runtime and all dialogue text updates automatically. No external packages required — works for everyone out of the box.

**Option B — Unity Localization package integration**
Threader reads from `LocalizedString` fields via Unity's official Localization package. Buyers who already use it get seamless integration with their existing workflow.

The most likely approach is to ship both: Option A as the default with zero setup friction, and Option B as an optional bridge assembly compiled only when the Unity Localization package is detected (`#if` conditional). This way existing buyers are never affected and no unwanted dependencies are introduced.

**Status:** Under consideration. Dependency strategy needs to be resolved before implementation begins.

---

## Out of scope

These are features that have been considered and deliberately won't be added to Threader core. If you need them, alternatives are noted.

### Built-in branching story format export (Ink, Yarn, Twine)

Threader's graph format is Unity-native and optimised for runtime execution, not text-based story formats. Converting between the two would require lossy translation of variable scopes, entry points, and events. Use Yarn Spinner or ink directly if you need a text-based authoring workflow.

### Multiplayer / networked dialogue synchronisation

Syncing dialogue state across a network is highly game-specific (what to sync, who is the authority, how to handle latency). Threader's runtime is single-player by design. Wire your own netcode layer on top of `OnNPCLine`, `OnChoiceNode`, and `SelectChoice` if you need multiplayer dialogue.
