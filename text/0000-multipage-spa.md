- Start Date: 2026-09-12
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# Multipage SPA support via file-based routing

## Summary

Add **zero-config-first**, declarative multipage SPA support to Chocola via file-based routing — inspired by SvelteKit `src/routes` (and Next.js `pages/`) — where the index page stays at `src/index.html` → `/` (SvelteKit `src/routes/+page`) and the root layout at `src/_layout.html` → shared chrome (SvelteKit `+layout`), while all other pages live in `src/pages/` declaratively: `src/pages/about.html` → `/about`, `src/pages/blog/[id].html` → `/blog/:id`.

**Zero-config-first** means it works **without any config file or config entry at all** — `src/index.html` alone is a valid app, adding `src/pages/*.html` or `src/_layout.html` just works with sensible defaults (`srcDir="src"`, `outDir="dist"`, `libDir="lib"`, `pagesDir="pages"`, MPA by default). No router config or imperative `createRouter()` is needed. Single-page `src/index.html`-only projects keep working unchanged; adding files under `src/pages/` progressively opts into multipage.

When a project *does* need customization, the same behavior is **optionally configurable through `chocola.config.json`** (already loaded by `utils.js:getConfig` → `compiler/config.js:loadConfig` with defaults when absent) — e.g. `bundle.pagesDir`, `bundle.spa`, `bundle.srcDir/outDir/libDir` — without changing file conventions or adding imperative code. Static build emits `dist/<route>/index.html` with shared hashed assets (`sc-*`, `run-*`, `css-*`), the dev server and `chocola/server` SSR build a route table from the same graph, and an opt-in progressive SPA navigation layer intercepts same-origin `<a>` clicks for instant client transitions without a full reload.

## Motivation

### What is hard today

Chocola today is single-page only. `compiler/pipeline.js:42-57` `getSrcIndex()` loads exactly one file — `src/index.html` — as `graph.page` (`compiler/module-graph.js:15` `PAGE_ID = "index.html"`, `198-212`). `compiler/index.js:61-86` `emit()` writes a single `dist/index.html` via `compiler/dom-processor.js:32`, and `server/index.js:103-112` `buildRouteTable()` maps only `"/"`, `"/index.html"`, `"/index"` to that one page module. There is no concept of additional pages.

Users who need more than one URL currently have three bad options:

1. **Manual multi-build hack** — run `app.build()` multiple times with different `srcDir`/`outDir` overrides and stitch `dist/` by hand. `chocola.config.json` (`compiler/config.js:4-16`) has no `pages` concept, so per-page props, layouts, and static assets must be copied manually.

2. **Single-page manual router** — render everything inside one `index.html` with `if`/`mount:if` chains (`compiler/render.js:24-117` `processPageConditionals`) keyed on `ctx` query params:
   ```html
   <!-- src/index.html today — imperative routing inside one page -->
   <app>
     <Navbar />
     <Home     if="{ctx.page === 'home'}" />
     <About    if="{ctx.page === 'about'}" />
     <BlogPost if="{ctx.page === 'post'}" postId="{ctx.id}" />
   </app>
   ```
   This forces all components and their `script`/`$runtime` chunks (`compiler/component-processor.js:495-572`, `compiler/runtime-generator.js:3-23`) into one payload even when only one route is active, breaks per-route `document.title`/`<head>` semantics, and makes deep-linking/SEO and per-route `ctx` middleware (`server/index.js:81-101` `loadMiddleware`) awkward.

3. **Eject to another framework** for anything with `/about`, `/blog/:id`, or a shared navbar/footer layout.

Concrete use-cases that require first-class multipage:

- Marketing site + app shell: `/`, `/about`, `/pricing`, `/blog`, `/blog/:slug` with a shared layout (nav + footer) but distinct HTML per route for SEO.
- E-commerce: `/products`, `/products/[id]`, `/cart`, `/checkout` — each route can lazy-load only its needed components; today `processAllComponents` expands the union of all routes.
- Dashboard with auth-gated routes where middleware supplies per-route `ctx` (`server/index.js:248-283`).

In each case the desired developer workflow is declarative: "create a file, get a route" — no registry, no bundler config change. And like SvelteKit, the entry points of that route tree — the index page and root layout — should live at `src/` itself (`src/index.html`, `src/_layout.html`), not nested inside `src/pages/`.

### What becomes possible

```bash
my-app/
├── src/
│   ├── index.html          # → /          (SvelteKit: src/routes/+page.svelte)
│   ├── _layout.html        # optional root layout (SvelteKit: +layout.svelte) with <slot />
│   ├── pages/              # ← other pages live here (zero-config)
│   │   ├── about.html      # → /about
│   │   ├── blog/
│   │   │   ├── index.html  # → /blog
│   │   │   ├── [id].html   # → /blog/:id
│   │   │   └── _layout.html # optional nested layout for /blog/*
│   │   └── contact.html    # → /contact
│   ├── lib/
│   │   ├── Navbar.html
│   │   └── Card.html
│   └── static/
│       └── logo.png
```

```html
<!-- src/_layout.html — declarative root chrome, like SvelteKit +layout.svelte -->
<template>
  <Navbar active="{ctx.pathname}" />
  <slot />
  <Footer />
</template>

<!-- src/pages/about.html — declarative page, same authoring as today's src/index.html -->
<head>
  <title>About — Chocola</title>
  <link rel="stylesheet" href="./styles.css">
</head>
<body>
  <app>
    <h1>About</h1>
    <Card title="Team" />
  </app>
</body>
```

Zero-config-first in practice (no `chocola.config.json` needed):

- **Works with no config file at all** — `src/index.html` is `/`, `src/_layout.html` (root layout) wraps all routes if present, and `src/pages/**/*.html` becomes routes automatically. `src/lib/` and `src/static/` remain excluded (as today `compiler/config.js` `libDir` + `pipeline.js:125-133` `copyStaticDir`). `app.build(rootDir)` resolves defaults via `compiler/config.js:loadConfig` when `chocola.config.json` is absent — same as today.
- Adding `src/pages/about.html` or `src/pages/blog/[id].html` is enough — `npm run build` now emits `dist/index.html`, `dist/about/index.html`, `dist/blog/index.html`, `dist/blog/[id]/index.html`-template + shared `sc-*.css`/`run-*.js` hashed assets reused across pages.
- `chocola/server` (`server/index.js:183-375` `createHandler`) serves all routes from one `routeTable`, with per-request `renderPage(pageModule, ctx)` and ETag/gzip unchanged.
- SPA navigation is progressive MPA by default; no config needed for same-origin `<a href="/about">`.

Only when you want to diverge from defaults do you touch `chocola.config.json` — e.g.:

```json
// chocola.config.json — entirely optional; shown overrides, not requirements
{
  "bundle": {
    "srcDir": "src",
    "outDir": "dist",
    "libDir": "lib",
    "pagesDir": "pages",
    "spa": true
  }
}
```

`spa: true` opts into the client router shim; `pagesDir` renames `src/pages/`; `srcDir`/`outDir`/`libDir` already exist today and keep their defaults. No new required keys — omitting the file or any key falls back to the zero-config defaults.

File rename/move = route rename/move. Deleting `src/pages/blog/[id].html` removes the route. No stale config to sync. Index and layout stay where SvelteKit users expect — at `src/` itself.

## Detailed design

### Technical Background

**Current pipeline** (`documentation/06-architecture/01-compiler-flow.md`):

1. `loadConfig` + `resolvePaths` (`compiler/config.js:4-24`) resolve `srcDir`/`outDir`/`libDir`.
2. `buildModuleGraph(rootDir)` (`compiler/module-graph.js:198-253`) creates `ModuleGraph { modules, components, loadedComponents, page }`:
   - `getSrcIndex(paths.src)` (`pipeline.js:42-57`) reads `src/index.html` → `pageModule { id:"index.html", kind:"page" }`.
   - `getComponents(paths.components)` (`pipeline.js:6-40`) loads `src/lib/*.html` → `ChocolaModule { kind:"component" }` with `compileComponentModule` parsing `script/template/style`, `extractPropsDefaults`/`extractTopLevelVariables/Functions`, `deterministicHash` for `cssId`/`fnId`, and `deps` via `import` regex + template `querySelectorAll("*")`.
   - `createDOM(page.source)` + `scanComponentDeps` + `scanAssetModules` (`module-graph.js:158-196`) populate `page.deps`.
3. `renderPage(graph, ctx)` (`compiler/render.js:136-182`) is pure: `createDOM` → `validateAppContainer` → `processPageConditionals(appContainer, ..., ctx)` → `processAllComponents(appElements, loadedComponents, page.sourcePath, page.source, ctx, treeShakeRuntime)` → `generateRuntimeScript(runtimeScript, csrSource, csrClasses)` → `processAssets` (hash `css-*`/`js-*`) → `serializeDOM` → `{ html, hashMap, files, copies }`. `ctx` merges `query` + middleware returns (`server/index.js:248-283`).
4. `emit(graph, opts)` (`compiler/index.js:61-86`) optionally clears `outDir`, calls `renderPage(graph, opts.ctx)`, writes `files`/`copies` + `index.html` (`dom-processor.js:32`) + `.chocola/hashes.json`.
5. `server/index.js:103-112` `buildRouteTable(graph)` returns `Map { "/","/index.html","/index" → graph.page }`; `createHandler` ingests virtual files (`run-*`, `sc-*`) on startup (`198-217`), then per request does route lookup → middleware → `renderPage(graph, ctx)` → `ingest` → `serveBuffer` with `ETag`/`Last-Modified`/`gzip` (`119-167`), falling back to static `src/` file serving (`318-352`).

**Key invariants to preserve**: `compileExpr` + `with(ctx)` proxy, `validateAppContainer` (`<app>`), `chid`/`fnId` determinism (`utils.js:112-114`), `runtimeMap` dedup, isomorphic `ChocolaComponent` (`runtime/index.js:173-312`), asset hashing (`pipeline.js:59-123`), `findElementLine` warnings, and the `buildModuleGraph → renderPage → emit/serve` separation that keeps SSR per-request pure.

**Prior art — SvelteKit as primary model, Next.js for comparison**: SvelteKit `src/routes/` maps filesystem → URL, `src/routes/+page.svelte` → `/`, `src/routes/+layout.svelte` → shared chrome with `<slot>`, `[param]`/`[...rest]` dynamic segments, nested `+layout`/`+layout.js` inheritance, `<a>` enhanced by SvelteKit client router. Next.js `pages/` is the same idea but with `pages/index.js` → `/`, `pages/_app.js` shared chrome and `[id]`/`[...slug]` params. Chocola adopts the hybrid requested here: **index + root layout at `src/`** (`src/index.html` → `/`, `src/_layout.html` → `+layout`) — matching SvelteKit placement for the entry points — while **remaining pages live in `src/pages/`** (`src/pages/about.html` → `/about`) for isolation and zero ambiguity with `src/lib/`/`src/static/`.

### Implementation

#### Overview

Introduce a **Pages collection** in place of the singleton `graph.page`, keeping perfect backwards compatibility with the hybrid placement:

- `src/index.html` is `/` (SvelteKit `+page` at root).
- `src/_layout.html` is the root layout (`+layout`) wrapping every page — including `src/index.html` and all `src/pages/**`. Excluded from route generation.
- Every `src/pages/**/*.html` file (excluding per-folder `_layout.html`) declaratively becomes a route: `src/pages/about.html` → `/about`, `src/pages/blog/[id].html` → `/blog/:id`.
- Single-page projects (today's `src/index.html`-only, no `src/pages/`) work unchanged — they are the one-element `pages` collection with optional `src/_layout.html`.
- No new required config — `src/index.html` + optional `src/_layout.html` + optional `src/pages/` is zero-config-first. `chocola.config.json` `bundle.libDir` already excludes `lib/`; layout exclusion is built-in.
- Pages reuse the existing `ChocolaModule { kind:"page" }` and `ChocolaComponent` pipeline; only discovery, graph shape, rendering, emitting, and routing become multi-entry.

#### A. Config & paths — zero-config-first, optional `chocola.config.json`

`compiler/config.js:4-16` `loadConfig` stays **zero-config-first**: it works with no file at all (`utils.js:getConfig` returns `{}` when `chocola.config.json` is absent, then defaults apply) and only *optionally* reads `chocola.config.json` when present:

```js
// compiler/config.js — loadConfig(rootDir)
const config = await getConfig(rootDir).catch(() => ({})); // no file → {}
const bundleConfig = config.bundle || {};
const pagesDir = bundleConfig.pagesDir || "pages"; // inside srcDir, like libDir
const spa = bundleConfig.spa ?? null; // null = auto (MPA default, SPA progressive)
return { srcDir, outDir, libDir, pagesDir, emptyOutDir, treeShakeRuntime, spa };
```

`resolvePaths` adds `pages: path.join(rootDir, config.srcDir, config.pagesDir)` alongside `src`/`components`/`outDir`. Index and root layout remain at `paths.src` (`src/index.html`, `src/_layout.html`) — not under `pages`. `utils.js:getConfig` already forwards unknown keys passively, so `chocola.config.json` stays entirely optional.

**Zero-config-first contract:**

- No `chocola.config.json` at all → `srcDir="src"`, `outDir="dist"`, `libDir="lib"`, `pagesDir="pages"`, `spa=null` (MPA), `emptyOutDir=true`, `treeShakeRuntime=true`. `src/index.html` + `src/pages/**` + `src/_layout.html` work immediately.
- With `chocola.config.json`, any subset of keys can be set — e.g. `{ "bundle": { "spa": true } }` to enable SPA without touching `srcDir`/`pagesDir`; `{ "bundle": { "pagesDir": "routes" } }` to rename `src/pages/` → `src/routes/`; `{ "bundle": { "srcDir": "app" } }` to move the whole tree. Missing keys still default. No new required field — absence of `src/pages/` = single-page (plus optional `src/_layout.html`).

#### B. Pages discovery — `compiler/pipeline.js`

Keep `getSrcIndex` for the index page and add `getPages` sibling to `getComponents`:

- `getSrcIndex(srcDir)` (`pipeline.js:42-57`) continues to load `src/index.html` → `pageModule { id:"index.html", kind:"page" }` (now the `/` entry in the collection, not the singleton).
- `export async function getPages(pagesDir)` (`new`):
  - `fs.readdir(pagesDir, { recursive:true })` (or manual walk for Node 18 compat) collecting `*.html` as POSIX `rel` relative to `pagesDir`.
  - For each `rel`, `source = await fs.readFile(path.join(pagesDir, rel), "utf-8")`, `empty` check (push to `emptyPages` if `trim().length===0` — warn via `chalk.yellow` like `module-graph.js:221`), `loadedPages.set(rel.toLowerCase(), source)`, `pagesLib.push(rel)` preserving original casing for emit.
  - Return `{ pagesLib, loadedPages, emptyPages }`. If `pagesDir` missing (`ENOENT`), return `{ pagesLib: [], loadedPages: Map, emptyPages: [] }` — caller falls back to index-only.
  - Ignore `**/_layout.html` from route generation here — handled in §E.

Root layout discovery is separate: probe `src/_layout.html` (canonical `_layout.html` — no `layout.html` plain form) at `paths.src` ahead of `getPages`.

#### C. Module graph — `compiler/module-graph.js`

Replace singleton `PAGE_ID` with multi-page, with index at `src/`:

```js
export class ModuleGraph {
  constructor(rootDir, config, paths) {
    // ...
    this.pages = new Map();      // id (rel POSIX, e.g. "index.html", "pages/about.html", "pages/blog/[id].html") → ChocolaModule
    this.layouts = new Map();    // dir key → ChocolaModule (e.g. "" → src/_layout.html, "pages/blog" → src/pages/blog/_layout.html)
    this.page = null;            // legacy compat getter alias → pages.get("index.html") || first page (src/index.html)
  }
  pageByRoute(route) { return this.routeMap?.get(route) ?? null; }
}
```

`buildModuleGraph`:

1. Load index page: `const indexFiles = await getSrcIndex(paths.src)` — if present, create `ChocolaModule { id:"index.html", kind:"page", sourcePath: path.join(paths.src,"index.html"), source: indexFiles.srcHtmlFile, mtimeMs }`, `graph.pages.set("index.html", mod)`, `graph.addModule(mod)`. If missing and `src/pages/` exists, `/` will 404 (warn to create `src/index.html`).
2. Load layouts: probe `src/_layout.html` at `paths.src` (only `_layout.html` — plain `layout.html` not accepted per this revision) for root layout; then for each directory under `src/pages/` that contains pages, probe `src/pages/blog/_layout.html` etc. for nested layouts — store in `graph.layouts` keyed by dir POSIX (`""` = root from `src/_layout.html`, `"pages/blog"` = nested). Layout modules are `kind:"layout"` but not added to `pages` and not emitted as routes.
3. `const foundPages = await getPages(paths.pages)` — if `pagesLib.length > 0`, for each `rel` in `pagesLib` where `rel !== "_layout.html"` (nested layouts already excluded), create `new ChocolaModule({ id: path.posix.join(config.pagesDir, rel), kind:"page", sourcePath: path.join(paths.pages, rel), source, mtimeMs })`, `graph.pages.set(id, mod)`, `graph.addModule(mod)`.
4. For each `pageModule` in `graph.pages.values()`: `pageModule.deps = scanComponentDeps(createDOM(pageModule.source).document, graph) ∪ scanAssetModules(...)` — reusing existing helpers. Asset deduplication across pages happens via shared `out.ids` during `renderPages` (§D) / emit.
5. Component graph (`compileComponentModule` per component) stays unchanged — pages simply add more roots depending on components.

`buildModuleGraph` still populates `graph.page` alias for back-compat (`graph.page = graph.pages.get("index.html") ?? graph.pages.values().next().value ?? null`) so external callers that reference `graph.page` continue to work for single-page projects.

File watching (`dev/index.js`) already watches `src/`; extend glob to watch `src/index.html`, `src/_layout.html`, and `src/pages/**/*` and invalidate `graph.pages`/`graph.layouts` entry on change (see §G).

#### D. Rendering — `compiler/render.js`

Change signature to **per-page** render while providing a backward-compatible batch wrapper:

```js
// new primitive
export async function renderPageFor(graph, pageId, ctx = {}) { /* pageId = rel e.g. "index.html" or "pages/about.html" */ }

// batch renderer used by emit/server
export async function renderPages(graph, ctx = {}) {
  // renders each graph.pages entry, shares out.ids per batch for dedup
}

// compat: preserve export async function renderPage(graph, ctx)  → renders graph.page (src/index.html)
export async function renderPage(graph, ctx = {}) {
  return renderPageFor(graph, graph.page.id, ctx);
}
```

`renderPageFor(graph, pageId, ctx)` is today's `renderPage` extracted with `page = graph.pages.get(pageId) ?? graph.page` and `page.source`/`page.sourcePath` replacing the singleton `graph.page.*` uses (`render.js:144-159`):

- Resolve layout chain for the page's dir (root `src/_layout.html` + any `src/pages/blog/_layout.html` ancestors) — see §E — then `layoutAppliedSource = applyLayouts(page.source, layoutChain)`.
- `createDOM(layoutAppliedSource)`, `validateAppContainer`, `processPageConditionals(appContainer, page.sourcePath, protectedContent, ctx)`.
- `processAllComponents(appElements, graph.loadedComponents, page.sourcePath, page.source, ctx, graph.config.treeShakeRuntime)` — same args, now page-scoped.
- `generateRuntimeScript(runtimeScript, csrSource, csrClasses)` + `processAssets(doc, graph, outBatch)` where `outBatch` is shared across `renderPages` to deduplicate `sc-*`/`run-*`/`css-*`/`js-*` by content hash, exactly as today's per-page `out.ids` dedup (`pipeline.js:68-105`).
- Return `{ html, hashMap, files, copies }` per page; `renderPages` merges `{ htmlByRoute: Map<route, html>, files: [], copies: [], hashMap }` (union of files).

Head handling: Chocola already preserves `<head>` content from the page source via linkedom `serializeDOM`; per-page `<title>`/`<meta>` thus work with no new API. Layout `<head>` is merged (page wins on conflicts). Add `validateHead` no-op or warn on duplicate.

Route ↔ file mapping helper (`compiler/page-route.js` new):

```js
export function fileToRoute(rel) {
  // rel POSIX e.g. "index.html" → "/", "pages/about.html" → "/about",
  // "pages/blog/index.html" → "/blog", "pages/blog/[id].html" → "/blog/:id",
  // "pages/blog/[...rest].html" → "/blog/:rest*"
  // strip "pages/" prefix, strip ".html", map "/index" suffix to "/", preserve bracket segments verbatim
}
export function routeToFile(route) { /* inverse for emit */ }
export function emitFileForRoute(route) {
  // "/" → "index.html", "/about" → "about/index.html", "/blog" → "blog/index.html"
  // ensures directory index for static hosts (Netlify/Vercel/Cloudflare Pages)
}
```

Dynamic segments `[id]` / `[...rest]`: **Phase 1** leaves them as literal files emitting `dist/blog/[id]/index.html` with placeholder content and a `console.warn` directing to Phase 2 param resolution (`TODO: wire ctx.params`). **Phase 2** resolves params server/build-time via `ctx.params`. File `[...slug].html` maps to `/:slug*`. RFC includes Phase 1 emissive behavior so static hosts still serve a file at that URL path; SSR param extraction follows in §F.

#### E. Declarative layouts — zero-config `src/_layout.html` at `src/` (SvelteKit `+layout`)

Root layout lives at `src/` itself — **only `src/_layout.html`** (plain `src/layout.html` not accepted — `_layout.html` keeps the layout namespace explicit and collision-free with a user `/layout` route):

- If `src/_layout.html` exists, its `<template>` is treated as a wrapper with a single `<slot />` where the page's `<app>` children are inserted. Implementation reuses existing slot replacement (`component-processor.js:601-636` slot logic): `layoutDoc = createDOM(layoutSource)`, `slot = layoutDoc.querySelector("slot")`, replace with `pageAppChildren` fragment before `processAllComponents`. If no `<slot>` is present, the layout's `<template>` is prepended as chrome (warn to add `<slot>`).
- Layout itself participates in component graph (imports, scoped CSS) and its `<head>` is merged (page `<head>` wins on conflicts).
- No JS layout config needed — presence of file = shared chrome. Absence = no wrapping. This mirrors SvelteKit: `src/routes/+layout.svelte` wrapping every `+page` — here `src/_layout.html` wrapping `src/index.html` (`/`) and every `src/pages/**`.
- Nested layouts: `src/pages/blog/_layout.html` wraps pages under `src/pages/blog/**`. `fileToLayoutChain(rel)` returns ancestor layout chain `[root src/_layout.html, ..., parent pages/blog/_layout.html]` so `src/pages/blog/[id].html` gets `root` → `blog` wrapping (innermost closest to page). Ship root-only in Phase 1, enable nested chain when `src/pages/blog/_layout.html` is detected — forward-compatible helper already supports it.
- `src/_layout.html` and per-folder `_layout.html` are excluded from `routeTable` (not routes), stored in `graph.layouts`.

Rationale for `_layout.html` over `layout.html`: `_`-prefix sorts first, clearly signals "not a route" (`/_layout` would be a weird route), and avoids colliding with a user-defined `/layout` page. Matches the original `pages/_layout.html` proposal's convention.

#### F. Server route table — `server/index.js`

Rewrite `buildRouteTable(graph)` (`103-112`) to iterate `graph.pages`:

```js
function buildRouteTable(graph) {
  const table = new Map();
  for (const [id, mod] of graph.pages) {
    const route = fileToRoute(id);   // "index.html"→"/", "pages/about.html"→"/about"
    table.set(route, mod);
    table.set(route.endsWith("/") ? route.slice(0,-1) : route + "/", mod);
    table.set("/" + id, mod);
    if (route !== "/") table.set(route + ".html", mod);
  }
  return table;
}

function matchRoute(routeTable, pathname) {
  if (routeTable.has(pathname)) return { page: routeTable.get(pathname), params:{} };
  // trailing slash variants, then param scan: "/blog/:id" pattern test, extract params
}
```

Per-request handler (`server/index.js:223-306`) becomes:

- `const { page, params } = matchRoute(routeTable, pathname) ?? {}`.
- `let ctx = { ...query, ...params }` then middleware merges (`Object.assign(ctx, result)`) and `opts.ctx` merges as before.
- `const result = await renderPageFor(graph, page.id, ctx)` → `ingest(result)` where virtual files are now union across pages (on-demand per route, not priming all pages at startup — only the requested page's `files` are ingested; static assets like `sc-*` are content-hashed so cross-page dedup is natural).
- `virtualFiles` remains `Map<pathname -> {buffer, etag, mtime, type}>` served via `serveBuffer` with `304`/`gzip` unchanged.
- Static fallback `src/static` remains, but is now subservient to `routeTable` — a request for `/about` hits SSR first, not a raw file. `src/lib/` is never served raw (components only).

Params flow: middleware receives `params` alongside `query/cookies/headers/url/pathname/searchParams` (`server/index.js:258-268`), so auth/validation can act per-route.

#### G. Dev server & HMR — `dev/index.js`

- Watch `src/index.html`, `src/_layout.html`, and `src/pages/**/*.html` plus `src/lib/**/*.html` via existing `fs.watch`/`chokidar`. On page/layout file add/change/unlink:
  - Rebuild `graph.pages`/`graph.layouts` entry (reuse `compileComponentModule` + `scanComponentDeps` for that page).
  - Rebuild `routeTable` (`handler.routeTable = buildRouteTable(graph)`).
  - Send WS HMR payload `{ type:"route-update", route:fileToRoute(rel) }` or full reload if `src/_layout.html` or asset changed (layout wraps every page, so `src/_layout.html` change triggers full app reload, matching existing `index.html`/config full-reload behavior `specs/hmr-and-lightweight-ssr.md:137`).
- Client HMR runtime already distinguishes component vs page updates; extend to swap page HTML fragment for SPA navigations without reload.

#### H. Client SPA navigation — progressive enhancement (zero-config MPA, opt-in via `chocola.config.json`)

**Default (MPA, zero-config)**: no config needed, no runtime change — `<a href="/about">` does a normal navigation, browser fetches SSR HTML. Already works with no `chocola.config.json`.

**Opt-in SPA** (still declarative, enabled via optional `chocola.config.json`): set `{ "bundle": { "spa": true } }` in `chocola.config.json` when you want it (or auto-enable when a tiny router shim is included by `generateRuntimeScript`):

1. Emit a small `nav-<hash>.js` (~1-2kB) appended after `run-*.js` (`runtime-generator.js` fourth descriptor) when `graph.config.spa` is `true` or `graph.pages.size > 1`.
2. Shim does:
   - Global click listener on `a[href]` with same-origin, no `target="_blank"`, no `download`, no `rel="external"`, and `href` matching `routeTable` (embedded as `__CHOCOLA_ROUTES__ = ["/","/about","/blog/:id"]` pre-hashed at build).
   - `e.preventDefault()` → `fetch(href, {headers:{"X-Chocola-Nav":"1"}})` → server responds with HTML (or `result.html` fragment extracted via `DOMParser`'s `<app>` subtree). Body's `<app>` replaces current `document.querySelector("app")`'s innerHTML, `<title>`/`<head>` delta merged, `history.pushState({route}, "", href)` + `scrollTo(0,0)` unless `hash`.
   - On `popstate`, re-fetch prior HTML or restore from `sessionStorage` cache.
   - After swap, re-run `DOMContentLoaded` invocations (`run-*.js` `r_<hash>(el, ctx)` calls) for newly mounted components — reuse existing `ChocolaComponent#mount` idempotent path (`runtime/index.js:214-251`).
3. Server optimizes `X-Chocola-Nav` requests by returning only `<app>` innerHTML + head delta + next `hashMap` JSON (content-type `application/json`) — progressive, but initial implementation can return full HTML and client extracts `<app>`.

This keeps pages declarative and independent — no `<Router>` component, no `navigate("/about")` imperative API required. For imperative navigation, users can call `history.pushState` or the shim-provided `window.__chocolaNavigate(href)` (mirrors `NextRouter.push` but optional).

Config toggle allows teams to ship MPA first (SEO safest) and flip `spa:true` when ready without file moves.

#### I. Emit — `compiler/index.js`

Replace single-write with per-route emit, sharing asset dedup:

```js
export async function emit(graph, options={}) {
  const outDir = graph.paths.outDir;
  await setupOutputDirectory(outDir, graph.config.emptyOutDir);
  const merged = { files: [], copies: [], ids: [], outDir };
  const htmlByRoute = new Map();
  for (const [id, mod] of graph.pages) {
    const res = await renderPageFor(graph, id, options.ctx);
    const route = fileToRoute(id); // "index.html"→"/", "pages/about.html"→"/about"
    const outPath = emitFileForRoute(route); // "index.html" or "about/index.html"
    htmlByRoute.set(route, outPath);
    for (const f of res.files) if (!merged.ids.includes(f.path)) { merged.ids.push(f.path); merged.files.push(f); }
    for (const c of res.copies) merged.copies.push(c);
    res._emitHtmlPath = outPath; res._emitHtml = res.html;
  }
  for (const res of perPageResults) await fs.writeFile(path.join(outDir, res._emitHtmlPath), res._emitHtml);
  // ... writes of merged.files/copies + .chocola/hashes.json as today
}
```

Legacy single-page fallback: when `graph.pages.size===1` (`src/index.html`-only, no `src/pages/`), `emitFileForRoute("/")` is `index.html` — output identical to today, hashed names unchanged. Adding `src/pages/*` simply adds more `dist/*/index.html` entries; `src/index.html` stays `/` at the root.

#### J. Preservation & migration

- No breakage: single-page projects (just `src/index.html`, `src/lib/*`, optional `src/_layout.html`) build exactly as before. `compiler/module-graph.js` alias `graph.page` and `renderPage(graph, ctx)` compat wrapper ensure external callers (`chocola.js` build script, `server/index.js`, `tests/compiler.test.js:68` `graph.page.id`) pass.
- Adding pages is additive: create `src/pages/about.html` and `/about` exists; delete it and it 404s. No config to update.
- Layout is opt-in: create `src/_layout.html` to get root chrome wrapping both `/` and `src/pages/**`; delete it to remove wrapping. Nested `src/pages/blog/_layout.html` opts into per-section chrome.
- Feature flag: `bundle.pagesDir` config (default `"pages"` inside `srcDir`) and `graph.config.spa` toggle for client router.
- Tests updated via alias — add new suites: `tests/pages.test.js` for `fileToRoute`/`emitFileForRoute` (`index.html→/`, `pages/about.html→/about`, `pages/blog/[id].html→/blog/:id`, `src/_layout.html` excluded), multi-page `buildModuleGraph` assertions, `emit` `dist/about/index.html` existence, `server` routeTable matching with params, SPA no-JS fallback (MPA still returns HTML).

#### K. Testing & bench

- Unit: `parser/page-route.test.js` covering `index.html→/`, `pages/about.html→/about`, `pages/about/index.html→/about`, `pages/blog/index.html→/blog`, `pages/blog/[id].html→/blog/:id`, `src/_layout.html` excluded, trailing slash normalization, dynamic precedence (static `/blog/featured` wins over `/blog/:id`).
- Integration: `tests/fixtures/multipage` with `src/index.html`, `src/_layout.html` wrapping Navbar + `<slot>`, `src/pages/about.html`, `src/pages/blog/[id].html`, `src/pages/blog/_layout.html` nested, assertions on `dist/` tree (root `index.html` at top-level plus `dist/about/index.html`), `routeTable` lookups, per-request `ctx.params`, layout `<slot>` insertion, and that `run-*.js` per page is subset of union (payload smaller than monolithic `if`-router page).
- SSR: `tests/server.test.js` extended: `GET /`, `GET /about`, `GET /blog/123` with middleware param injection, `304`/`gzip` for page HTML, `SPA fetch` `X-Chocola-Nav` partial.
- Bench: `bench/index.js` adds `renderPages` sweep 1→200 pages × 50 components/page measuring `buildModuleGraph` + per-page `renderPageFor` vs. today's single monolithic page; expect linear scaling per added page but smaller per-page payload vs. today's `if`-router, reported as `routesCount`, `avgPageRenderMs`, `sharedAssetDedup%`.

## How we teach this

This is an **evolution of what beginners already know** — "make a file, get a route" — with SvelteKit-familiar entry points at `src/`, and it stays **zero-config-first**: no `chocola.config.json` needed to start; add one only when you want to customize:

- `documentation/01-introduction/03-project-structure.md` gains:
  ```
  my-app/
  ├── src/
  │   ├── index.html         ← /            (SvelteKit: src/routes/+page)
  │   ├── _layout.html       ← root layout  (SvelteKit: src/routes/+layout, Chocola: _layout.html)
  │   ├── pages/             ← other routes (zero-config)
  │   │   ├── about.html     → /about
  │   │   ├── blog/
  │   │   │   ├── index.html → /blog
  │   │   │   └── [id].html  → /blog/:id
  │   │   └── contact.html   → /contact
  │   ├── lib/               ← unchanged: components
  │   └── static/            ← unchanged: static assets
  └── ...
  ```
  Callout: "> `src/index.html` is `/` and `src/_layout.html` (if present) wraps every page — just like SvelteKit `+page`/`+layout` at the routes root. Every other route is a file under `src/pages/` — no config."

- `documentation/01-introduction/02-getting-started.md` second step after "Create `src/index.html`" becomes "Make more pages: create `src/pages/about.html` and refresh — your new `/about` route is live. Add `src/_layout.html` with a `<slot />` for a shared navbar/footer that wraps `src/index.html` and all pages. No router code, no config."

 - New guide `documentation/04-routing/01-file-based-routing.md` (SvelteKit voice, Chocola idioms):
  - Declarative table `src/index.html` → `/`, `src/pages/about.html` → `/about`, `src/pages/blog/index.html` → `/blog`, `src/pages/blog/[id].html` → `/blog/:id` (params via `ctx.params.id`), `src/_layout.html` shared shell with `<slot>` (and `src/pages/blog/_layout.html` nested), file rename = route rename, delete = 404.
  - SvelteKit mapping note: `src/index.html` ≈ `src/routes/+page.svelte`, `src/_layout.html` ≈ `src/routes/+layout.svelte`, `src/pages/blog/[id].html` ≈ `src/routes/blog/[id]/+page.svelte` (here isolated under `src/pages/`).
  - Zero-config-first callout: "No `chocola.config.json` needed — conventions work out of the box. Create `chocola.config.json` only to override: `bundle.pagesDir` to rename `pages/`, `bundle.spa: true` for SPA, `bundle.srcDir/outDir/libDir` as today."
  - SPA vs MPA: same-origin `<a href="/about">` just works with no config (MPA); enable SPA by adding `chocola.config.json` `{ "bundle": { "spa": true } }` for instant transitions (progressive enhancement, zero `<Link>` import needed, but `window.__chocolaNavigate` available if you want it).
  - Head per page: put `<title>`/`<meta>` in that page's `<head>` — they merge with `src/_layout.html` head.

- `documentation/06-architecture/01-compiler-flow.md` §§2,4,8 update: graph now holds `pages: Map<route, Module>` (`src/index.html` + `src/pages/**`) plus `layouts` (`src/_layout.html` at root), render path is `renderPageFor(graph, pageId, ctx)` + `renderPages`, emit writes `dist/<route>/index.html`, server `routeTable` built from `fileToRoute` with root `src/_layout.html`.

 - `documentation/07-reference/01-faq.md` add Q&A: "How do I add a new page? Create `src/pages/my-page.html` — that's it, no config. Do I need `chocola.config.json`? No — it works zero-config; create `chocola.config.json` only to customize `pagesDir`/`spa`/`srcDir`. Where's the layout? Create `src/_layout.html` with a `<slot />` at `src/` — like SvelteKit `+layout` at the root. Where's the index? `src/index.html` at `src/` — like `+page` at root."

No API churn for components: `src/lib/*.html` authoring, `<script>` + `function $runtime()` (`documentation/02-components/01-fundamentals.md:95-140`), `bind:*`, `if/mount:if`, CSS scoping all behave identically per page. Users who never create more than `src/index.html` notice no change; SvelteKit users feel at home — index and layout live at `src/`, other pages in `src/pages/`.

## Drawbacks

- **More files emitted**: `dist/` grows from one `index.html` to N `* /index.html` files; static hosts handle this well (directory-index convention) but custom hosting that expects a single file needs its own serving rule — mitigated by `emitFileForRoute` producing `index.html` per directory which most static servers already handle.
- **Two-level discovery mental model**: index/layout at `src/` while other pages are in `src/pages/` is one more rule than "everything in `src/pages/`" or "everything in `src/`". Mitigated by SvelteKit familiarity (root `+page`/`+layout` + nested routes) and by the zero-config payoff: `src/index.html` keeps its historic meaning, and `src/pages/` isolates non-index routes so stray `src/*.html` drafts don't publish.
- **Shared runtime duplication risk**: each page's `run-*.js` is content-hashed globally, so identical runtime chunks dedup via shared `files` union (no 2× payload). But pages with divergent component sets still generate distinct `run-<hash>.js` per page; `DOMContentLoaded` shim must still broadcast per-route. Mitigated by asset dedup and the SPA shim's caching.
- **Route precedence nuance**: file system implies an order (`/blog/featured.html` vs `/blog/[id].html` under `src/pages/blog/`). Without a top-level route manifest, ties are resolved by longest static match — teachable like Next.js/SvelteKit but subtly different from imperative router `addRoute("/blog/:id", handler)`.
- **`with(ctx)` dynamism carries per route**: `compileExpr` + `Proxy(has(){return true})` in `processPageConditionals`/`component-processor` stays per-page; dynamic `ctx.params` still triggers fallback. No worse than today, but per-page server resolution amplifies the "include-all + warn" path for truly dynamic expressions.
- **SSR table size scales with page count**: `routeTable` grows O(pages); for 500 pages still tiny (Map lookups, segment split for params). Largest cost is per-request `matchRoute` param scan — linear in param-routes, but bounded by route count and cached after first hit (LRU if needed).
- **SPA shim weight/complexity**: ~2kB client JS + fetch/swap/history logic is more code to maintain and test than pure MPA; opt-in via `spa:true` keeps default MPA simple, but teams expecting "SPA by default like SvelteKit/Next.js" may need to set one flag.
- **`_layout.html` naming**: `_`-prefix avoids colliding with a `/layout` route but is one more convention to teach vs plain `layout.html`. Consistent with Next.js `_app`/`_layout` and SvelteKit `+layout` "not a route" signal, and kept to a single canonical form (`_layout.html`) per this revision to minimize alias confusion.

## Alternatives

- **Do nothing (single-page `if`-router)**: keep `src/index.html` as the only entry and document `ctx.page` branching. Retains simplicity but permanently caps Chocola at single-URL apps, forces payload union, and abdicates per-route SEO/head/middleware ergonomics. Rejected — community growth will demand multipage regardless.
- **All pages in `src/pages/` including index** (Next.js `pages/`): require `src/pages/index.html` → `/` and `src/pages/_layout.html` for root layout. More isolated (pages can't collide with stray `src/*.html` drafts) but introduces the extra folder for the entry point the user explicitly wanted at `src/` (`src/index.html`) and for the root layout — breaking SvelteKit expectation (`+page`/`+layout` at root) and requiring a migration for existing `src/index.html`.
- **All pages directly in `src/`** (`src/` as routes root): scan `src/**/*.html` excluding `lib`/`static` as routes (`src/about.html` → `/about`). Zero nesting but means every `*.html` outside `lib`/`static` is a route by default — a stray `src/draft.html` would publish. Hybrid `src/index.html` + `src/pages/**` is clearer.
- **SvelteKit-literal `src/routes/` subfolder** (`src/routes/+page.html`/`+layout.html`): fully SvelteKit-faithful but diverges from Chocola's current `src/index.html` location and leaves `src/lib` routing ambiguous. Hybrid keeps `src/` as before while still mapping `src/_layout.html` ↔ `+layout` at root.
- **Explicit router config (central `routes.json` / `chocola.routes.js`)**: `export default { "/": "./src/index.html", "/about": "./src/pages/about.html" }`. More flexible but reintroduces a manifest to keep in sync, breaks file-rename-is-route-rename declaratively, and adds API churn. Considered as `bundle.routes?: Record<string,string>` escape hatch for power users later — not the zero-config default.
- **Next.js `app/` router clone (nested `layout.js` + `page.js` + `loading.js`)**: powerful but requires multiple files per route and a mental model shift from Chocola's single-file component (`<template>` + `<script>` + `<style>`). Deferred — single `src/_layout.html` (+ nested `src/pages/blog/_layout.html`) covers 90% of shared chrome with one file.
- **Full MPA only, no SPA shim**: emit multipage and rely on browser navigations. Simpler and maximally SEO-safe, but loses instant transitions that users associate with SPA. Chosen progressive approach (MPA default, `spa:true` opt-in) captures both: static-first correctness with SPA perf when requested.
- **Build-time only, no server route table**: static `emit` multipage but keep `server/index.js` single-page SSR. Would force dual pipelines and break `createHandler` parity (`server/index.js:183` is shared by dev + prod SSR). Rejected — same `renderPages`/`routeTable` powers both.

Other frameworks: SvelteKit `src/routes/` (`+page`/`+layout`, `[id]`/`[...rest]`, nested `+layout`, client router enhancing `<a>`), Next.js `pages/` (`[id]`/`[...slug]`, `_app.js`/`_document.js`), Nuxt `pages/`, Astro `src/pages/` (`[...slug].astro`). All validate declarative filesystem routing as the right zero-config-first default. This RFC adapts SvelteKit's *placement* (index + root layout at `src/`) to Chocola's `src/pages/` for other routes, with `_layout.html` as the layout primitive.

## Unresolved questions

- Exact emit layout on static hosts: `dist/about.html` vs `dist/about/index.html` vs both (duplicate file with same content)? Proposal uses directory-index `about/index.html` only; should we also write flat `about.html` for raw `file://` preview?
- Dynamic route ergonomics: how does `src/pages/blog/[id].html` access `ctx.params.id` — via `{params.id}` in template (`compileExpr` already handles `ctx` proxy) or via `export let id` auto-bridged from params? Preference: `ctx.params.id` and `{params.id}` first (no prop bridging), add prop bridging later if ergonomic wins are strong.
- Nested layouts beyond root: when to promote `src/pages/blog/_layout.html` per-folder shells beyond root `src/_layout.html`? Ship root-only in Phase 1, enable nested chain when `src/pages/blog/_layout.html` is detected — or include nested resolution now via `fileToLayoutChain`?
- SPA freshness: should `nav-<hash>.js` prefetch `href` on `mouseenter`/`viewport` (Next.js `<Link prefetch>` / SvelteKit `data-sveltekit-preload`) or fetch on click only? Prefetch is faster but more bandwidth — likely opt-in `prefetch: true` config.
- 404/500 pages: `src/_404.html` / `src/_error.html` at `src/` as declarative error boundaries (SvelteKit `+error.svelte` analogy)? Or keep generic `server/index.js:355` `404 Not Found` body until needed?
- Asset scoping: should `sc-*.css` be page-chunked (one per page) or global union (today's single `sc-*.css` per render)? Global union dedup is simplest; per-page chunking could further shrink per-route payload but adds `<link>` bookkeeping per page.
- Should `routeTable` be embedded into client `nav-<hash>.js` as `__CHOCOLA_ROUTES__` at build time or fetched `/_routes.json` at runtime for very large apps (1000+ routes)? Embed is faster for typical sites; lazy fetch scales better for huge.
- Compatibility: should `graph.page` alias remain forever or be deprecated behind a `DEPRECATED_GRAPH_PAGE` warn after one major? Preference: keep alias through next major, warn in `buildModuleGraph` when `pages.size > 1` and caller reads `.page` expecting single.
- Should we also accept `src/layout.html` (plain) and `src/+layout.html` (SvelteKit literal) as aliases for `src/_layout.html`? Proposal is `_layout.html`-only per this revision to avoid alias confusion — open to re-adding `+layout.html` alias if SvelteKit porting friction arises.
