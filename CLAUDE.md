# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Hugo static blog, deployed to GitHub Pages at `https://codemonkeybr.github.io/`. Posts are
write-ups about AI-assisted development work — mostly Claude Code sessions — drawn from real
projects that live under `~/dev/` (a sibling directory, one level up from this repo). This repo
only contains the blog itself; the source projects it writes about are elsewhere on disk.

## Git conventions

No AI attribution in commits or PRs on GitHub — do not add `Co-Authored-By: Claude` trailers,
"Generated with Claude Code" footers, or session links to commit messages, PR descriptions, or
anything else pushed to this repo's GitHub remote.

## Commands

Hugo is not preinstalled system-wide (`apt`'s `hugo` package is too old — see Hugo version below).
It's installed to `~/.local/bin/hugo` (already on `PATH`).

```bash
# local dev server with drafts visible, live reload at http://localhost:1313/
hugo server --buildDrafts

# production build (what CI runs) — outputs to ./public
hugo --gc --minify

# scaffold a new post (uses archetypes/posts.md — sets draft=true, empty tags/description)
hugo new content posts/<slug>.md

# after cloning fresh, or if themes/PaperMod is empty
git submodule update --init --recursive
```

There is no lint or test suite — correctness is "does `hugo` build without errors/warnings" and
"does the page render correctly," checked via `hugo server`.

New posts default to `draft = true`; set `draft = false` in the front matter to publish. A draft
never appears in a production build (`buildDrafts = false` in `hugo.toml`), only in `hugo server
--buildDrafts`.

## Hugo version

The `PaperMod` theme submodule requires Hugo **0.146.0+** (**extended** edition, for Sass/SCSS).
The Ubuntu `apt` package (`0.123.7`) is too old and will fail with a module-compatibility warning
on build/new-content. `~/.local/bin/hugo` is currently `v0.148.2-extended`, installed by downloading
the `hugo_extended_<version>_linux-amd64.tar.gz` release directly from
`github.com/gohugoio/hugo/releases` (no sudo, no apt). The version pinned in
`.github/workflows/hugo.yml` (`HUGO_VERSION`) must be kept in sync with whatever's needed by the
theme — bump both together if the theme submodule is updated and starts requiring a newer minimum.

## Architecture

- `hugo.toml` — site config: theme selection, PaperMod params (`[params]`), nav menu
  (`[[menu.main]]`), syntax highlighting (`[markup.highlight]`), tags taxonomy. This is the file
  to touch for site-wide settings (title, description, social links, menu structure).
- `content/posts/` — blog posts, one Markdown file (or `<slug>/index.md` bundle) per post, TOML
  front matter (`+++ ... +++`). `tags` in front matter populate `/tags/<tag>/` automatically via
  the `taxonomies` config.
- `archetypes/posts.md` — front-matter template used by `hugo new content posts/<slug>.md`
  (`archetypes/default.md` is Hugo's generic fallback for other sections; posts should use the
  `posts` archetype by virtue of being created under `content/posts/`).
- `themes/PaperMod/` — git submodule, not vendored/modified in this repo. Don't edit files inside
  it; theme-level overrides belong in `layouts/`, `assets/`, or `static/` at the repo root (Hugo's
  standard override mechanism: same relative path under the site root wins over the theme).
- `.github/workflows/hugo.yml` — CI/CD: on push to `main`, builds with `hugo --gc --minify` and
  deploys `./public` to GitHub Pages via `actions/deploy-pages`. Checkout uses
  `submodules: recursive` so the theme is pulled in. No separate preview/staging deploy.
- `public/` and `resources/` are build output / Hugo's asset cache — gitignored, never commit them.

## Writing posts sourced from `~/dev/` projects

When a post is about a project in `~/dev/<project>/`, that project's own `.claude/` folder (if
present) and its git history/commit messages are the primary source material — read those rather
than asking the user to re-explain context already captured there. Treat this blog repo and the
project repo as separate: never copy source code wholesale into a post; write about it.

## Voice

Posts are written in first person from the site owner's perspective ("I set up...", "I learned...",
"my gateway") — never from an AI perspective ("we built...", "as an AI..."). Claude Code did the
research and drafting, but the narrator of the post is the human who did the work.

Don't mention AI/Claude Code assistance in post *content* either, not just git — no "built with
Claude Code," no "an AI helped me build it." The blog's about-page framing ("What this repo is,"
above) is internal guidance for me, not phrasing to echo in published posts.
