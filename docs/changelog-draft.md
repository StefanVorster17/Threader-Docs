# Changelog Draft

*Internal working doc — not in nav. Copy entries to changelog.md before release.*

---

## [1.1.0] — April 20, 2026

### Graph Editor

- **Drag-to-create node spawner** — drag from any port and release on an empty area of the canvas; a searchable popup opens at the cursor, letting you pick a node type to create and automatically wire the edge in one gesture
- **Node spawn popup** — categorised node list with colour-coded accent stripes matching each node's title colour; real-time search filter; keyboard navigation (↑↓, Enter to confirm, Escape to dismiss)
- **Right-click "Add Node…"** — the right-click context menu now has a single "Add Node…" entry that opens the same popup, replacing the previous flat list of individual node entries; the menu is compact and consistent with the port-drag flow
- **Sidebar UI consistency** — GRAPH panel label rows aligned across font size, colour, and spacing

### Node Types

- **Player Node `[P]`** — new node type for scripted, auto-advancing player lines, distinct from Player Choice Node (interactive choices) and NPC Node (NPC-attributed lines). Fires a new `OnPlayerLine` event (separate from `OnNPCLine`) so UI can style player lines differently — e.g. a right-aligned bubble with no speaker label. Supports Events, Set Variable actions, and localized text via Line Data. Has no Speaker field; the player's display name comes from the new **Player Name** field on `SpeakerRoster` (default `"Me"`). Not available in bark graphs. See [Node Reference — Player Node](nodes.md#player-node-p).
- **Bark NPC Node** — bark-graph line node with built-in per-speaker support: one shared text line plus a list of speaker entries (Speaker Name, Clip, Line Key, Animator Actions). Replaces NPC Node in the CREATE sidebar for bark graphs; existing NPC nodes in bark graphs continue to work unchanged. Has its own [multi-language Line Sheet support](line-sheet.md) — inline speaker entries act as a fallback when no sheet row exists. See [Node Reference — Bark NPC Node](nodes.md#bark-npc-node).

### Bark System

- **`BarkSpeakerRoster`** asset — declares the speaker types available to bark graphs, separate from the regular `SpeakerRoster` list. Create via **Create → Threader → Speakers → Bark Speaker Roster** and assign to `DialogueManager`'s new **Bark Speaker Rosters** list.
- **Per-graph Bark Speakers list** — each bark graph now has its own filtered subset of speakers (picked from the assigned rosters) via the sidebar's **SPEAKERS → Bark Speakers** picker. `BarkNPCNode` speaker dropdowns and `BarkSource.speakerName` are scoped to this per-graph list, so unrelated speakers never clutter the picker.
- **`BarkSource.randomiseSpeaker`** — new toggle; when enabled, `Start()` picks a random speaker from the graph's Bark Speakers list instead of using a fixed Speaker Name. Useful for crowds of interchangeable NPCs sharing one prefab.
- Full write-up: [Bark System — multi-speaker barks with Bark NPC Node](bark.md#multi-speaker-barks-with-bark-npc-node).

### Speaker Roster

- **Player Name field** — new field on `SpeakerRoster` (default `"Me"`) driving the display name used for Player Node lines. `DialogueManager.GetPlayerName()` reads the first non-empty value across all assigned rosters.

### Editor / UX

- **`DialogueGraph` Inspector cleanup for Bark graphs** — the Line Sheets section and the "migrate legacy sheet" helper are now hidden entirely for Bark graphs (they use `BarkNPCNode`'s own per-speaker line sheets instead); the "Look At Speaker" toggle was removed from the Inspector (still available, dialogue-graph-only, in the graph editor's SPEAKERS sidebar); the Inspector header now reads "Bark Graph" or "Dialogue Graph" instead of always "Dialogue Graph"; the redundant blue "BARK" tag label next to the title was removed.
- **Graph Type moved out of the graph editor sidebar** — switching a graph between Dialogue and Bark is now done from the **Graph Type** dropdown at the top of the `DialogueGraph` asset's own Inspector, not the graph editor's sidebar. The editor's top bar shows a read-only **`· Dialogue`** / **`· Bark`** badge next to the graph name instead. Changing the type on a graph that already has nodes now shows a confirmation dialog warning that it may break existing nodes, line sheets, and connections.
- **GRAPH sidebar section renamed to SPEAKERS** — now focused purely on speaker configuration: Default Speaker / Look At Speaker for Dialogue graphs, or the Bark Speakers roster-picker for Bark graphs.
- **Line Sheet Editor sidebar button hidden for Bark graphs** — bark graphs no longer show a button that opened an editor with no applicable data.
- **`Threader/` Create Asset menu reorganized** into four submenus: **Graph** (Dialogue Graph, Dialogue Node Template), **Speakers** (Speaker Roster, Bark Speaker Roster), **Variables** (Variables Store, Variable Condition, Condition (Custom)), **Lines** (Line Sheet, Language Library) — replacing the previous flat list.

### Bug Fixes

- **Player Node text missing in the Dialogue Preview Window** — the preview window was reading text directly off the node (always empty for Player Node, whose text lives only in the line sheet); it now falls back to the primary sheet's `PreviewText`, matching runtime behaviour.
- **Player Node showing `<empty>` intermittently on reopening a graph** — the node view only checked the line sheet when a preview language was selected; it now always checks the primary sheet when the node has no language override, with a deferred rebuild to cover the case where the sheet asset hasn't finished loading when the graph first opens.
- **Edge save/load bug with Player Node** — Player Node was missing from four internal edge-handling paths in the graph view (load, apply, clear, template-paste reconnect), which could silently drop its wiring in specific save/reload sequences. Fixed.
- **`DialogueUI` not showing Player Node lines** — the reference UI only subscribed to `OnNPCLine`, so Player Node lines raced through instantly with no typewriter effect. It now subscribes to `OnPlayerLine` as well; the player's speaker name renders in green by default.

---