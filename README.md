# darknetian-posts

Public source mirror of blog posts and project writeups from
[darknetian.com](https://www.darknetian.com).

The Markdown files here are auto-synced from the private
`darknetian-web` repo on every push to its `master` branch.
Spot a typo? See something to add? Open a PR against this repo
and Nic will pull it back into the source.

## Layout

- `posts/` — blog posts (URL: `https://www.darknetian.com/posts/<slug>/`)
- `project/` — project writeups (URL: `https://www.darknetian.com/project/<slug>/`)

Filenames are date-prefixed: `YYYY-MM-DD-<slug>.md`.

## Editing

Each rendered post on darknetian.com has an "edit this post on GitHub" link
at the bottom that opens the matching file here in the GitHub editor.

This mirror is one-way: do not edit darknetian.com files here expecting
them to live in this repo as the source of truth — they don't.
The next sync would overwrite. Submit PRs; merged PRs land back in the
source repo manually.
