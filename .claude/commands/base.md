---
description: "Create an Obsidian Bases database view for the wiki. Usage: /base [name] or /base (for preset options)"
---

Create an Obsidian Bases view: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

## Presets

If $ARGUMENTS matches a preset name, use the definition below.
Otherwise, create a custom base from the description.

### Preset: `sources`
Table view of all wiki sources, sortable by date and tag.
```yaml
filters:
  and:
    - file.inFolder("wiki/sources")
    - 'file.ext == "md"'
views:
  - type: table
    name: "All Sources"
    order:
      - file.name
      - tags
      - file.mtime
```

### Preset: `concepts`
Card view of all concepts with tag filtering.
```yaml
filters:
  and:
    - file.inFolder("wiki/concepts")
    - 'file.ext == "md"'
views:
  - type: cards
    name: "Concept Gallery"
    order:
      - file.name
      - tags
      - updated
```

### Preset: `books`
Table view of all book pages with author and year.
```yaml
filters:
  and:
    - file.inFolder("wiki/books")
    - 'file.ext == "md"'
views:
  - type: table
    name: "Book Library"
    order:
      - file.name
      - author
      - year
      - tags
```

### Preset: `all` (full dashboard)
Multi-view base showing all wiki content.
- View 1 (table): all sources by date
- View 2 (cards): all concepts
- View 3 (list): all synthesis pages

---

## Steps

1. Determine the base definition:
   - Named preset → use definition above
   - Custom description → design using **obsidian-bases** skill

2. Create the base file using **obsidian-bases** skill:
   - Path: `wiki/bases/[name].base`
   - Validate: YAML is valid, all formula references resolve, property names exist

3. Report: "Created wiki/bases/[name].base — open in Obsidian to see the view"
   Optionally embed in a note: `![[wiki/bases/[name].base]]`

If $ARGUMENTS is empty, show preset options:
```
Available presets:
  /base sources   — table view of all sources
  /base concepts  — card gallery of all concepts
  /base books     — book library table
  /base all       — full wiki dashboard

Or describe a custom view: /base [description]
```

Notes:
- `.base` files require Obsidian Bases plugin (built into Obsidian 1.9+)
- Bases read frontmatter properties — ensure wiki pages have consistent frontmatter
- Use **obsidian-bases** skill for complex formula or filter logic
