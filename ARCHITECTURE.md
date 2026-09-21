# Site Architecture

Reference doc for working on robertpmartin.com — a static site built with **Quartz 5** (not Quartz 4 — the plugin system, config format, and build pipeline are different) and deployed to **GitHub Pages**. Read this before making structural changes; it's the map that took real debugging to build (see the "gotchas" section — those aren't hypothetical).

## Repo layout at a glance

```
quartz-site/
├── content/                  ← your posts/pages. Git-tracked source of truth.
├── quartz/                   ← vendored Quartz 5 engine (rarely hand-edit)
│   ├── components/           ← core JSX components + frame types
│   ├── plugins/               ← plugin loader/types (roles: transformer/filter/emitter)
│   ├── styles/                ← base.scss, variables.scss, custom.scss ← YOUR styling surface
│   └── util/                  ← shared helpers (theme.ts, fileTrie.ts, path.ts, ...)
├── quartz.config.yaml         ← THE config file: theme, plugins, layout — all of it
├── quartz.ts                  ← CLI entrypoint (rarely touched)
├── package.json                ← npm deps, incl. every @quartz-community/* plugin
├── public/                    ← build output (generated, gitignored, deployed)
└── .github/workflows/deploy.yml ← CI: builds + publishes to GitHub Pages on push to main
```

There is **no `quartz.layout.ts`** in this version — layout (which components render where) lives entirely inside `quartz.config.yaml`'s `layout:` block plus each plugin's own `layout:` sub-key.

## Frontend components

Two tiers:

**1. Core components** — `quartz/components/*.tsx`, hand-written in this repo:
- `Body.tsx`, `Head.tsx`, `Header.tsx` — page skeleton
- `Date.tsx`, `Flex.tsx`, `PageList.tsx`, `Spacer.tsx`, `ConditionalRender.tsx`, `DesktopOnly.tsx`/`MobileOnly.tsx` — small primitives
- `registry.ts` / `index.ts` — component registration
- `frames/` — the page-frame variants: `DefaultFrame.tsx` (sidebars + header/footer), `FullWidthFrame.tsx` (no sidebars, full-bleed center), `MinimalFrame.tsx` (no chrome at all). Selected per-page via frontmatter (see below). `@quartz-themes/core` also injects `canvas` and `excalidraw` frame variants at the CSS level, not as files here.
- `pages/404.tsx` — the 404 page

**2. Plugin-provided components** — one per `@quartz-community/*` npm package (installed in `node_modules`, **don't hand-edit** — see "Library modules" below). Each is declared in `quartz.config.yaml`'s `plugins:` list with a `layout:` block controlling *where* it renders (`position: left/right/beforeBody/afterBody/footer`, `priority`, optional `group`). Currently active: `explorer`, `search`, `graph`, `backlinks`, `table-of-contents`, `article-title`, `content-meta`, `page-title`, `darkmode`, `reader-mode`, `breadcrumbs`, `footer`, `note-properties`, `spacer`. A few are installed but disabled (`tag-list`, `comments`, `recent-notes`, `stacked-pages`) — flip `enabled: true` to turn one on.

The `layout:` top-level block in `quartz.config.yaml` also defines **groups** (e.g. `toolbar`, which bundles search + darkmode + reader-mode into one row) and **per-page-type overrides** (`byPageType.folder`/`tag` hide reader-mode and the right sidebar; `404` hides everything).

## "API layers" → the build pipeline

This is a static site generator, not a server — there's no runtime API. The equivalent concept is the **build pipeline**, which every plugin entry in `quartz.config.yaml`'s single `plugins:` list participates in by declaring a role internally (Quartz 5 unified what used to be three separate arrays in Quartz 4 — `transformers[]`/`filters[]`/`emitters[]` — into one ordered YAML list; each package still plays one of those roles under the hood, visible in `quartz/plugins/index.ts` as `ctx.cfg.plugins.transformers` / `.emitters`):

1. **Transformers** — parse/mutate the markdown AST, in `order:` sequence: `created-modified-date`(10) → `syntax-highlighting`(20) → `obsidian-flavored-markdown`(30, wikilinks/callouts/mermaid/tags) → `github-flavored-markdown`(40) → `table-of-contents`(50) → `crawl-links`(60) → `description`(70) → `latex`(80) → `hard-line-breaks`(90).
2. **Filters** — decide which parsed files actually get emitted: `remove-draft`, `unlisted-pages`, `explicit-publish` (disabled), `encrypted-pages`.
3. **Emitters** — write output files: `content-page` (the actual HTML pages), `folder-page`, `tag-page`, `canvas-page`, `bases-page`, `content-index` (sitemap + RSS), `favicon`, `og-image`, `cname`, `alias-redirects`.

`order:` on a plugin entry controls transformer sequencing; plugins without `order:` (most components/emitters) run in list order or are order-independent.

## Library modules

Three distinct tiers — know which one you're touching:

| Tier | Location | Edit it? |
|---|---|---|
| This repo's own helpers | `quartz/util/*.ts` (`theme.ts`, `fileTrie.ts`, `path.ts`, `slugCollisions.ts`, `resources.tsx`, ...) | Yes, if you need to change shared build logic |
| Vendored Quartz 5 core | rest of `quartz/` (components, plugin loader, styles/base.scss) | Rarely — these are the engine; upstream changes would need re-vendoring |
| Installed plugin packages | `node_modules/@quartz-community/*`, `node_modules/@quartz-themes/*` | No — these are real npm packages (see `package.json`). Configure via `quartz.config.yaml`, update via version bump + `npm install` |

`theme.ts` is worth knowing specifically: it turns `quartz.config.yaml`'s `theme.colors`/`typography` into the `--light`/`--dark`/`--headerFont`/etc. CSS custom properties injected at build time, and builds the Google Fonts URL from the `weights:` arrays you configure per font role.

## Styling architecture

- **`quartz/styles/variables.scss`** — breakpoints (`$mobile: 800px`, `$desktop: 1200px`), the 3-column grid definitions (`$sidePanelWidth: 320px`), font-weight constants.
- **`quartz/styles/base.scss`** — Quartz's own base styles. Part of the vendored engine; avoid hand-editing.
- **`quartz/styles/custom.scss`** — **this is the real customization surface.** Emitted *outside* any CSS `@layer`, so it normally beats everything in `@layer quartz-base` regardless of selector specificity or source order. All of the neobrutalist design tokens (`--radius`, `--brutal-border`, `--brutal-shadow`, `--brutal-press`, the full color palette) live here — see the file's own header comments for the Figma reference they were verified against.

### ⚠️ Gotcha: `@quartz-themes/core` can still outrank `custom.scss`

`@quartz-themes/core` (the "ported Obsidian theme" plugin, `theme: default` in config) injects its **own separate unlayered stylesheet**, not just the `@layer obsidian-theme` block you'd expect. That extra stylesheet uses **ID-selector rules** (`div#quartz-root.page`, `div#quartz-root.page div#quartz-body`) for its own responsive layout logic. An ID selector beats a class selector regardless of layering or file order — so a `custom.scss` rule on `.page { ... }` can silently do nothing, because it's not a layering problem, it's a specificity problem from a source you can't see in `custom.scss` itself.

**If a custom.scss change doesn't visibly apply:** open devtools (or ask Claude to `getComputedStyle` + walk `document.styleSheets`) before assuming it's a cascade-layer issue. If the winning rule has an ID in its selector, match that specificity (mirror the same ID-anchored selector) and add `!important` — that's the only reliable way to beat it. This bit us twice on this project: once for color tokens (documented in `custom.scss`'s own header), once for the page-width fix (see the "Wider page frame" section of `custom.scss`).

## Content model

- **`content/`** is git-tracked directly — it is **not** a symlink into the Obsidian vault. (It used to be the other way in an earlier commit; that got flipped specifically so GitHub's checkout doesn't need to resolve a symlink pointing outside the repo.) If you edit through Obsidian, the vault side has a symlink pointing *into* this folder — same files either way.
- Folder structure under `content/` mostly mirrors old Squarespace/WordPress date URLs (`YYYY/M/D/slug.md`) for migrated posts. New pages don't need to follow that — any path works.
- Frontmatter in use: `title`, `date`, `slug` (explicit slug overrides the path-derived one — migrated posts set it explicitly to preserve old URLs), `status`, plus `tags`/`description`/`aliases` where relevant (surfaced by the `note-properties` plugin).
- `quartz.config.yaml`'s `ignorePatterns` excludes `private/`, `templates/`, `.obsidian/`, and a few named draft files from the build — add a path there instead of deleting a draft you're not ready to publish.
- Obsidian-flavored markdown is fully supported: `[[wikilinks]]`, `> [!callout]`, `#tags`, mermaid diagrams, `- [ ]` checkboxes, block references.

## How to add a new page

1. Create a `.md` file anywhere under `content/` (e.g. `content/2026/new-post.md`).
2. Add frontmatter — at minimum `title:`; add `tags:`/`date:` if you want them surfaced. You generally don't need to set `slug:` — Quartz derives it from the file path unless you override it.
3. Write content in Obsidian-flavored markdown.
4. To change the page frame (e.g. a full-bleed landing page instead of the default sidebar layout), set frontmatter `frame: full-width` or `frame: minimal` — see `quartz/components/frames/`.
5. Preview locally (below) before pushing.

## Preview locally

```bash
npx quartz build --serve --watch
```

Serves at `http://localhost:8080` with live rebuild on save. This repo has a `.claude/launch.json` one directory up (`robertpmaritn_website/.claude/launch.json`) so Claude Code's browser-preview tooling can launch it directly. **Only one process can hold port 8080 at a time** — if you're running this yourself while also asking Claude to verify something, tell it, so it doesn't spin up a conflicting instance or (worse) tries to kill what it thinks is a stray process on that port.

## How to push updates to GitHub

Standard git flow from `quartz-site/`:

```bash
git status                          # see what changed
git add <files>                     # or `git add -A` if you're sure everything's yours
git commit -m "Describe the change"
git push
```

Remote is `origin` → `https://github.com/rpm44/robertpmartin-website.git`, tracking `main`.

**Pushing to `main` triggers the deploy automatically** — `.github/workflows/deploy.yml` runs on every push to `main`: checks out, `npm ci`, `npx quartz build`, then publishes the `public/` output to GitHub Pages via `actions/deploy-pages`. No separate deploy step needed; check the **Actions** tab on the repo to watch a run or diagnose a failed build.
