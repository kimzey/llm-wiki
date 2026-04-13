---
description: "Create a visual knowledge map canvas for a topic. Usage: /canvas [topic]"
---

Create a visual knowledge map canvas for: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Read index.md to find all concepts and sources related to $ARGUMENTS
2. Read the relevant concept pages to understand relationships
3. Plan the canvas layout:
   - Central node: main topic
   - Surrounding nodes: key concepts, sub-topics, related sources
   - Edges: relationships between nodes (labeled with relationship type)
   - Groups: cluster related nodes together

4. Create the canvas file using **json-canvas** skill:
   - Path: `wiki/canvas/[slug].canvas`
   - slug = topic name in lowercase-hyphens
   - Use `file` nodes to link to existing wiki pages: `wiki/concepts/[slug].md`
   - Use `text` nodes for summary labels and annotations
   - Use `group` nodes to cluster related concepts
   - Color coding: concepts = green ("4"), sources = cyan ("5"), synthesis = purple ("6")

5. Validate the canvas JSON:
   - All node IDs are unique
   - All edge fromNode/toNode reference existing node IDs
   - JSON is valid and parseable

6. Update index.md — add row to a "Canvas" section (create section if not exists):
   `[[wiki/canvas/slug|Title]] | Visual map of [topic] | YYYY-MM-DD`

7. Report: "Created wiki/canvas/[slug].canvas with N nodes and N edges"

If $ARGUMENTS is empty, ask: "หัวข้อที่ต้องการสร้าง knowledge map คืออะไร?"

Notes:
- Canvas files open automatically in Obsidian's Canvas view
- Use `file` nodes (not `text`) for existing wiki pages — Obsidian renders them as live previews
- Keep canvas readable: max ~15 nodes per canvas, split large topics into sub-canvases
