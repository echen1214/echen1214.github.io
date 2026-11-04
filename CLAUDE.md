# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Eric Chen's personal academic website, served at https://echen1214.github.io by GitHub Pages from the `master` branch. It is a customized (detached) copy of the [academicpages](https://github.com/academicpages/academicpages.github.io) Jekyll template, which is itself a fork of Minimal Mistakes. There is no GitHub Actions build workflow for the site; GitHub Pages builds it on push.

Because this started as a template, much of the tree is still generic template demo content rather than Eric's. See "Template leftovers" below before touching anything.

## Commands

Local preview, with live rebuild on Markdown/HTML change:

```bash
bundle exec jekyll serve -l -H localhost
```

`_config.yml` is not reloaded on change. Restart the server after editing it.

First-time setup, or after a Gemfile change:

```bash
bundle install
```

If `bundle install` hits permission errors, install gems into the repo instead:

```bash
bundle config set --local path 'vendor/bundle'
```

Docker alternative, if the Ruby toolchain is uncooperative. This serves on port 4000 and layers `_config_docker.yml` over `_config.yml` to blank out `url`:

```bash
docker compose up
```

Rebuild the minified JS bundle after editing `assets/js/_main.js` or anything in `assets/js/plugins/`. The committed `assets/js/main.min.js` is what the site actually loads, so an edit to the sources alone changes nothing:

```bash
npm install && npm run build:js
```

Regenerate `_data/cv.json` from the Markdown CV, for the unused JSON CV page:

```bash
./scripts/update_cv_json.sh
```

There are no tests and no linter in this repo.

## Architecture

**Content lives in collections, not pages.** `_config.yml` defines four output collections: `publications`, `talks`, `teaching`, `portfolio`. Each entry is one Markdown file whose YAML front matter carries the real content, and whose body is usually empty. The pages under `_pages/` are thin loops that iterate a collection and delegate rendering to an include. So adding a publication means adding a file to `_publications/`, never editing a page.

**Rendering flows page to include.** `_pages/publications.html` groups `site.publications` by `post.category` against the `publication_category` map in `_config.yml`, then calls `_includes/archive-single.html` per entry. `_pages/cv.md` loops the same three collections but calls the compact `-cv` variants. The result is two renderers per collection, and they have drifted:

| Collection | Full listing include | CV listing include |
|---|---|---|
| publications, teaching | `archive-single.html` | `archive-single-cv.html` |
| talks | `archive-single-talk.html` | `archive-single-talk-cv.html` |

`archive-single.html` handles `paperurl`/`slidesurl`/`bibtexurl` through a long `elsif` chain over all eight combinations. The talk includes were customized here to add `posterurl`, which the publication include does not know about. If you add a new front-matter download field, you must add it to both includes in a pair or it silently disappears from one view.

**Front matter is the schema.** Publications use `title`, `collection`, `category`, `permalink`, `date`, `venue`, `citation`, and optional `paperurl`/`slidesurl`/`bibtexurl`. Talks use `type`, `venue`, `date`, optional `location`, and `slidesurl`/`posterurl`. `date` drives sort order and the displayed year, and `reversed` in the page loops puts newest first.

**Navigation is explicit and narrower than the site.** `_data/navigation.yml` lists only Publications, Portfolio, Blog Posts, and CV. Talks and Teaching are commented out but still build and are still reachable at `/talks/` and `/teaching/`, and talks still appear inside the CV page. A page existing is not the same as it being linked.

**The sidebar comes from `_config.yml`.** The `author:` block drives `_includes/author-profile.html`; a blank field drops its icon and link entirely. This fork added `pronouns` and `employer` fields that upstream academicpages does not have.

**Styling.** `assets/css/main.scss` pulls in `_sass/`, which contains vendored Susy, Breakpoint, and Font Awesome alongside the site's own partials under `_sass/layout/`. Themes live in `_sass/theme/` and are selected by `site_theme` in `_config.yml` (`default` or `air`, each light and dark). Do not edit anything under `_sass/vendor/`.

**PDFs and assets.** Files in `files/` are published at `/files/<name>.pdf` and referenced from front matter. `files/` is listed under `include:` in `_config.yml` because Jekyll would otherwise skip some of it.

Front-matter paths must match the filename's case exactly. macOS is case-insensitive, so a wrong-case link works in local preview and 404s once GitHub Pages serves it from Linux. This has bitten the repo once already, in `_talks/2025-poster.md`. After adding a PDF link, verify it against the real filename rather than against a local preview.

## Template leftovers

The visitor-facing placeholder content has been removed: the two demo teaching entries, the lorem-ipsum draft, the demo comment data, the fake author and JSON CV data, the template documentation pages, and the dummy PDFs and demo images. What remains, and why:

- `_includes/footer.html` carries a hard-coded Recurse Center referral banner with a tracking link, added deliberately in commit `ccd6474`. It shows on every page.
- `_portfolio/` and `_posts/` are empty, so `/portfolio/` and `/year-archive/` render as empty pages despite both being in the main nav. Either add content or drop the nav entries in `_data/navigation.yml`.
- `_teaching/` is now empty and the Teaching section of `_pages/cv.md` is commented out. `/teaching/` still builds as an empty page. Uncomment the CV section when real teaching entries exist.
- `markdown_generator/` still holds the upstream TSV examples and notebooks. They are example input for tooling, never rendered, and the TSVs point at upstream's URLs rather than this repo's `files/`. Publication entries here are written by hand.
- `README.md` is still upstream's template README, aimed at someone forking academicpages. Its local-setup instructions are accurate for this repo; the "Use this template" framing is not.
- `/cv-json/` and `_data/cv.json` were removed along with their placeholder data, but `_includes/cv-template.html` and `scripts/` remain. To bring that page back, recreate `_pages/cv-json.md` and regenerate the data. The generator needs PyYAML, which is not installed, and its section regex (`^([A-Za-z\s]+)$`) silently drops any heading containing punctuation, so "Talks/Conferences/Posters" and "DEI & Outreach/Leadership" would be lost.
- The `scrape_talks.yml` workflow runs `talkmap.ipynb` on any `_talks/` push and commits back to the repo. The talkmap feature is disabled (`talkmap_link: false`) and no talk has a `location` field, so it geocodes nothing.
- `images/eric_prof.jpg` is a 7.3 MB, 2670x4000 original. It loads on every page via the sidebar and is displayed a few hundred pixels wide.
- `_site/` and `.sass-cache/` exist on disk from local builds. Both are gitignored and untracked. Never commit them.

## Conventions

To comment out a block in a Markdown page, use `{% comment %}` / `{% endcomment %}`, not `<!-- -->`. Liquid strips its own comments before kramdown runs. An HTML comment wrapped around Markdown headings and Liquid tags gets split by kramdown, and the closing `-->` then renders as visible text on the page. This happened once in `_pages/cv.md`.

Publication and talk filenames are `YYYY-slug.md`, and the `permalink` inside repeats the slug under `/publication/` or `/talks/`. Keep the two consistent; the filename does not generate the URL.

Asterisks in `citation` strings mark co-first authorship. Preserve them.

`url_slug` and the TSV workflow in `markdown_generator/` are not in use here. Publication entries are written by hand.
