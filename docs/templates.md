# Node Templates

Node templates let you save any selection of nodes as a reusable stamp. Drag a template pill onto any graph to instantiate a fresh copy — all nodes get new GUIDs, internal connections are re-wired, and you can immediately edit the stamp without affecting the original.

Templates are **stamp-and-detach**: once placed, the copy is completely independent of the template asset. For *live-linked* reuse (one graph called from many places without duplication), use the [Sub Graph Node](sub-graph.md) instead.

---

## Saving a template

1. Select the nodes you want to capture in the [graph editor](graph-editor.md)
2. In the **[NODE TEMPLATES](graph-editor.md#node-templates)** sidebar section, click **Save Selection**
3. A save-file dialog appears — choose a folder and name for the `.asset`
4. The template is created and immediately appears in the sidebar panel

Internal connections (edges between selected nodes) are preserved. External connections (edges to nodes outside the selection) are dropped.

---

## Using a template

**From the NODE TEMPLATES sidebar:**

Drag a template pill from the sidebar directly onto the graph canvas. The nodes are stamped at the drop position, preserving their relative layout. All nodes receive fresh GUIDs; internal wiring is re-connected automatically. The pasted nodes are selected immediately so you can move them as a group.

**From the Project window:**

Dragging a `DialogueNodeTemplate` `.asset` directly from Unity's Project window onto the graph canvas also works — same result as dragging from the sidebar.

---

## Managing templates

### Rename

Right-click any template pill → **Rename…**

A small modal window opens with a text field. Edit the name and click **Rename** (or press Enter) to confirm. This updates both the display name stored in the asset and the `.asset` filename on disk.

### Delete

Right-click any template pill → **Delete**

A confirmation dialog appears before the asset is removed from disk.

### Refresh

Click the **Refresh** button in the NODE TEMPLATES section to re-scan the project for template assets. The list updates automatically when you save or rename a template, but use Refresh if you move or add assets manually.

---

## DialogueNodeTemplate asset

`DialogueNodeTemplate : ScriptableObject`

Create manually via **Assets → Create → Threader → Dialogue Node Template**, or automatically via **Save Selection**.

The asset stores:

- A deep-copy snapshot of each node in the selection
- A `TemplateEdge` list of internal connections
- A `BoundsMin` value for positioning nodes relative to the drop point
- A `TemplateName` string (also reflected in the `.asset` filename)

The asset is plain JSON under the hood and safe to version-control. Moving it between projects works as long as all node types referenced in the template are available.

---

## Contrast with Sub-Graph Node

| | Templates | Sub-Graph Node |
|---|---|---|
| How it works | Stamps a copy; nodes are independent | Live-links to another graph at runtime |
| Edit after placing | Yes — each stamp is its own copy | Edits to the sub-graph affect all callers |
| Speaker fallback | Each copy has its own speaker fields | Inherits caller speaker chain |
| Best for | Boilerplate patterns you customise per NPC | Shared content reused verbatim |
