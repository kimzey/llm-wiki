---
description: "Save a quick note or thought into raw/notes/ for later ingestion. Usage: /note [your note text]"
---

Save the following as a note: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Create a new file `raw/notes/[YYYY-MM-DD]-[slug].md`
   - slug = first 5 words of the note, lowercase-hyphens
   - Date = today's date
2. Content format:
   ```markdown
   ---
   created: YYYY-MM-DD
   type: note
   ---

   [note content]
   ```
3. Confirm: "Saved to raw/notes/[filename].md — run `/ingest raw/notes/[filename].md` when ready to add to wiki."

If $ARGUMENTS is empty, ask: "What would you like to note down?"
