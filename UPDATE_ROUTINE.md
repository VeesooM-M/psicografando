# Psicografando — Update Routine (v2, automated)

This replaces the earlier routine entirely. The old structure (one giant
`generate_site.py` with every post's text embedded inside it) has been
retired — it was the actual cause of the token/session costs that made
updates slow. This version fixes that at the root, not just the symptom.

## How to publish a new post (the whole process)

1. Write one new file: `posts_content/<slug>.md`
2. Format:
   ```
   ---
   slug: your-slug-here
   date: Month DD, YYYY
   title: Your Title
   excerpt: One or two sentences for the homepage preview.
   ---

   First paragraph.

   Second paragraph. *italic* and **bold** both work inline.

   > A blockquote, if needed.
   ```
3. Push that one file to `main`.
4. That's it. A GitHub Actions workflow (`.github/workflows/build.yml`)
   notices the new file automatically, runs `generate_site.py` itself,
   and commits the resulting `index.html`, `posts/<slug>.html`,
   `posts.json`, and `sitemap.xml` back to the repo — no session of
   Claude needs to run anything by hand anymore for a normal post.

Content gets reviewed and approved *before* this step — writing and
editorial judgment still happen the way they always have. What's
automated is only the mechanical build-and-publish afterward.

## Why this is structured the way it is

- **Post text lives in its own small file, never inside the generator
  script.** The old version embedded every post's full text directly
  inside `generate_site.py`, so the script grew with every post, and
  touching it meant pulling the entire back-catalogue into context just
  to add one new thing. That was the real source of the session/token
  cost — not the mechanics of git itself.
- **`generate_site.py` should stay roughly the same size forever.** If
  you ever find yourself pasting a post's text into that file, stop —
  that is the exact mistake this restructuring exists to undo.
- **Posts are sorted by the `date` field, not by filename or folder
  order.** This was a real bug caught after the first live test: the
  original version sorted files alphabetically, which happened to
  produce a plausible-looking but wrong order. If you ever touch the
  sorting logic again, test it against at least three posts with dates
  that are *not* already in filename order, the way this bug was
  actually caught.
- **The GitHub Actions workflow only triggers on changes to
  `posts_content/**.md` or `generate_site.py` itself**, on the `main`
  branch specifically. If a future restructuring moves to a different
  branch strategy, this trigger has to be updated too, the same way it
  had to be moved from `markdown-restructure` to `main` during this
  migration — forgetting that step means posts stop auto-publishing
  with no obvious error, just silence.

## Dates

- Drafts stay undated until the moment they are published. A post can be written on one day and published days later.
- At publish time, take the date from the session context, never from memory, and compare it with the date of the newest existing post before pushing. The homepage sorts by `date`, so a wrong month silently puts the post in the wrong place. This nearly happened: "September 8" was typed on October 8, which would have placed the new post below one dated September 30.
- Re-read the date at the moment of publishing. A date mentioned earlier in the conversation as the planned publish day is not the publish date: one post was planned for October 15, went out on October 16, and was first stamped with the planned date.
- Keep relative time out of the body text ("this month", "four days ago", "recently published"). It goes stale the moment the publish date moves. A post written for one date and published under another kept saying "this month" about a month that no longer matched.
- If the date in the session context is missing or looks wrong, ask before publishing.

## If the automation breaks

The manual fallback still exists and still works: pull
`generate_site.py`, run it locally with `CONTENT_DIR` and `OUTDIR`
pointed at wherever you're working, verify the output, push the
resulting files by hand. This is exactly how the first four posts were
tested before the automation was trusted. Nothing about the manual path
was removed — only made optional for the common case.

## History, briefly, for context

The very first version of this blog kept all content inline in one
script. That version worked but didn't scale — every edit cost more
than the last as the back-catalogue grew. The old structure was kept,
renamed, as a reference rather than deleted, in case anything about it
is ever useful to compare against. It should not be built on top of
going forward.
