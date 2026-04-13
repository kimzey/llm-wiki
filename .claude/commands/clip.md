---
description: "Clip a web page to raw/clips/ using defuddle. Usage: /clip [url]"
---

Clip the following URL into the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Extract content using defuddle:
   ```
   defuddle parse $ARGUMENTS --md
   ```
   Also extract metadata:
   ```
   defuddle parse $ARGUMENTS -p title
   defuddle parse $ARGUMENTS -p description
   defuddle parse $ARGUMENTS -p domain
   ```

2. Generate filename from the page title:
   - slug = title converted to lowercase-hyphens (max 60 chars)
   - filename = `raw/clips/[slug].md`
   - If slug can't be determined → use domain + date: `raw/clips/[domain]-[YYYY-MM-DD].md`

3. Prepend metadata header to the defuddle output:
   ```markdown
   ---
   title: "[page title]"
   url: "[original URL]"
   clipped: YYYY-MM-DD
   domain: "[domain]"
   type: clip
   ---
   ```

4. Write to `raw/clips/[slug].md` (absolute path)

5. Confirm: "Clipped to raw/clips/[filename].md"
   Then ask: "ต้องการ ingest เข้า wiki เลยไหม?"
   - Yes → run /ingest raw/clips/[filename].md
   - No → done (file is saved, can ingest later)

If $ARGUMENTS is empty, ask: "URL ที่ต้องการ clip คืออะไร?"

Notes:
- defuddle must be installed: `npm install -g defuddle`
- If defuddle is not installed: ask user to install it, or offer to use WebFetch instead (output will be less clean)
- Never clip URLs ending in `.md` — use /ingest directly on local files
