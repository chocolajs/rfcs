- Start Date: 2026-10-09
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# ChocolaKit production build with precompiled server bundle

## Summary

Add a production build step to ChocolaKit that compiles a project once — pages, components, layout, and route table — into a pure JS server bundle, so that SSR server initialization (`createHandler` / `serve`) can load and serve from that bundle instead of recompiling the project from scratch on every startup.

## Motivation

Today ChocolaKit's SSR compiles the project from scratch every time the server is initialized. `createHandler(rootDir)` in `kit/server/index.js` calls `buildMultipageGraph(rootDir)` (filesystem walk, config load, component load, page discovery, layout read/validation) and then renders every route once up front to prime virtual files (`run-*.js`, `css-*.css`, etc.). Only after that can the server accept requests, and per-request `renderRoute(graph, route, ctx)` still runs the full Chocola `renderPage` pipeline against the in-memory graph.

That is the correct behavior for `dev` (file watching, instant edits), but it is wrong for production serving:

- **Slow cold start.** Startup cost scales with page/component count: every `.html` source is read, parsed, validated, and rendered before `listen()` succeeds. Adding pages linearly increases deploy boot time, health-check latency, and autoscale responsiveness.
- **Duplicate work across deploys/restarts.** The same project compiled identically on the last boot is recompiled identically on the next boot. Containers, serverless instances, and Preview deployments pay this cost on every cold start, even though the output is deterministic for a given source tree.
- **Source required at runtime.** The production server needs the full `src/` tree (`src/pages/`, `src/app.html`, `src/lib/`, `chocola.config.json`) plus the Chocola compiler present at serve time. That couples the runtime image to the authoring layout, prevents a clean `build` / `serve` split in CI, and makes integrity checks (what exactly was deployed?) harder.
- **No deployable artifact.** There is no single output to cache, hash, upload to a CDN/host, roll back, or promote from staging to production. Current "build" (`emitMultipage`) emits static HTML + assets, which serves fully static sites but does not help SSR routes that need per-request `ctx` (query, cookies, middleware).
- **Per-request work cannot be reduced without a build boundary.** Middleware ctx merging, head interpolation, and `renderPage` will always run per request, but parsing components, validating layout slots, building the route table, and generating unchanged runtime/CSS assets should happen once at build time, not per boot and partially per request.

Concrete use cases unlocked:

1. `chocola build` in CI produces e.g. `.chocola/server.js` (pure JS, no `.html` parsing at import time); `chocola serve` / `node .chocola/server.js` boots by importing it in milliseconds.
2. Docker image contains only `package.json`, `node_modules` (runtime), and the bundle — no `src/`, no compiler needed at runtime.
3. Rollbacks and content hashing work on the bundle artifact, not on a live source checkout.
4. Large multipage sites (100+ routes) keep fast dev startup while production startup becomes O(bundle import) instead of O(pages × compile + initial render-all).

If this RFC is rejected, the underlying constraint remains: SSR startup does redundant compile work per boot and production serving has no artifact boundary. Any alternative must still answer how cold start, reproducibility, and source-at-runtime are addressed.

## Detailed design

### Technical Background

Current ChocolaKit SSR flow (as observed in `kit/`):

1. `server/index.js:createHandler(rootDir, opts)` loads config (`getConfig` / `loadConfig`), resolves paths, then `routing.js:buildMultipageGraph(rootDir)`:
   - loads shared components via Chocola `getComponents(paths.components)`,
   - discovers pages via `discoverPagesInSrc` (layout mode: `src/app.html` + `src/pages/**/*.html`; legacy mode: standalone `.html` files under `src/`),
   - validates layout (`exactly one <chocolakit:body>`, at most one `<chocolakit:head>`, no `<app>`), validates per-page `<chocolakit:head>` fragments, synthesizes page components (`pageComponentName` / `pageComponentTag`), builds `graph = { modules, components, loadedComponents, pages: Map<route, ChocolaModule>, page, mode, layoutSource, ... }`.
2. `createHandler` builds the route table (`buildRouteTable`: clean URLs plus trailing-slash / `.html` aliases) and pre-renders every route (`renderRoute(graph, route, {})`) to populate the `virtualFiles` map (generated `run-*.js`, `sc-*.css`, `css-*.css`, `js-*.js`).
3. Per request, the handler matches the route, runs middleware to build `ctx`, calls `renderRoute(graph, route, ctx)` → Chocola `renderPage(view, ctx)` (with synthetic layout+page composition in layout mode), re-ingests generated files into `virtualFiles`, and serves HTML or a virtual/static asset with ETag/gzip handling.

Chocola itself (`../chocola`) owns `renderPage`, `compileExpr` / `with(ctx)` evaluation, deterministic hashing, and static emit. ChocolaKit owns routing conventions, layout composition, route table, middleware ctx, and the HTTP server. Any build-bundle design must keep that boundary: Chocola compiles components/pages; Kit composes routes and serves.

Prior art: Next.js `next build` → `.next/server/` bundle + `next start` loading it without recompiling; SvelteKit `vite build` → SSR output + client manifest, served by the adapter's Node handler; Astro `astro build` → pre-rendered pages + SSR bundle; Remix route-module bundling. All separate a Node-only build phase (filesystem access, compilation) from a minimal server phase (import bundle, handle requests). ChocolaKit should follow the same split.

### Implementation

This RFC proposes the shape and requirements, not the final code. A plausible implementation:

**1. New build entry: `compileMultipage` already exists; add `buildServerBundle`.**

- Add `kit/build.js` (or extend `routing.js:emitMultipage`) exporting e.g. `buildServerBundle(rootDir, { outDir })`.
- Input: same `rootDir` + config resolution as today (defaults < `chocola.config.json` < CLI flags, consistent with the portable-CLI RFC `text/0002-portable-cli.md`).
- Output: a self-contained pure JS bundle, e.g. `.chocola/server.mjs` (exact path TBD), plus a client-asset directory (`client/` or reuse `dist/` layout: `run-*.js`, `css-*.css`, static copies, `hashes.json`, per-route pre-rendered HTML where `prerender: true`).
- The bundle must contain, in serialized/precompiled form: resolved config + paths, layout source (already slot-validated), per-page precompiled modules (SFC already normalized via `toComponentSFC` / `splitPageHead`), shared component registry, route table + aliases, and pre-generated deterministic assets (runtime scripts, scoped CSS) that do not depend on per-request `ctx`.

**2. What "pure JS" means.**

- Importing the bundle performs no `fs` reads of `src/`, no glob/walk, no `linkedom` parse of raw `.html`, no layout validation. All of that happened at build time; build errors (duplicate routes, missing `<chocolakit:body>`, invalid `<chocolakit:head>`, page/component collisions) fail `build`, not `serve`.
- The bundle exports a `createHandler`-compatible factory, e.g. `createBundleHandler({ middleware })` or serialized `graph` + `routeTable` that the existing `server/index.js` can consume. Middleware user files stay external (loaded at serve time by path, as today) so secrets/edge config are not baked in unless explicitly opted in.
- Per-request dynamic behavior is preserved: `ctx` from query/cookies/middleware, head interpolation `{expr}`, conditional rendering, and `renderPage(view, ctx)` still run per request — but against precompiled modules, not freshly parsed sources.

**3. `serve` / `createHandler` loading order.**

- At startup, `serve` checks for a bundle first (explicit `--bundle` flag or default `.chocola/server.mjs`): if present and fresh, import it and skip `buildMultipageGraph` + render-all priming. Virtual files are hydrated from the bundle's embedded asset manifest instead of initial renders.
- If no bundle is present (dev, or `serve` run from source), fall back to today's compile-from-source path unchanged. `dev` never uses the bundle; it keeps watching `src/`.
- Add a `--no-bundle` / `--from-source` escape hatch and a staleness check (bundle records source hash / config hash / kit+chocola version; `serve` warns or errors on mismatch — exact policy TBD).

**4. CLI surface (consistent with `0002-portable-cli.md`).**

- `chocola build [root]` gains Kit awareness: in a Kit project (detectable via `src/pages/` or `src/app.html`) it emits both static assets and the server bundle; in a single-page Chocola project behavior is unchanged.
- `chocola serve [root]` prefers the bundle; flags `--port/--host/--middleware` keep current precedence (flags > `PORT` env > config > defaults).
- No new global install requirements; build is Node-only and may reuse the existing compiler pipeline.

**5. Scope boundaries.**

- No changes to component SFC syntax, `$runtime` semantics, `bind:*`, or Chocola's `renderPage` signature beyond what is needed to accept precompiled modules.
- Static-site generation (`emitMultipage` fully static output) stays as-is; the bundle additionally covers SSR routes.
- CSR/hydration payloads (`run-*.js`) are build artifacts either way; this RFC does not change tree-shaking (see `text/0001-server-resolved-script-runtime.md`).

## How we teach this

This is a workflow addition, not a new rendering concept:

- Getting-started docs gain a two-phase story: `chocola dev` (source, watch, recompile) vs `chocola build && chocola serve` (artifact, fast boot). The existing "zero-boilerplate" narrative from `0002-portable-cli.md` is preserved — `serve` without a bundle still works from source, it is just slower and documented as dev/convenience mode.
- New reference page (e.g. `documentation/07-reference/…-build.md`) documents: what the bundle contains, where it is written, `--bundle` / `--from-source` flags, staleness behavior, and what must not be baked in (middleware secrets, `PORT`, per-request ctx).
- Architecture docs (`compiler-flow`-equivalent for Kit) update the SSR diagram from "boot = walk + parse + validate + render-all" to "build = walk + parse + validate + emit bundle; boot = import bundle + hydrate virtual files".
- Migration is additive: existing projects with init scripts / `createHandler(rootDir)` keep working; adopting the build step is opt-in until declared stable, then recommended for production.

## Drawbacks

- **New artifact to keep correct.** Bundle staleness (source edited after build, config changed, Kit/Chocola version drift) must be detected and surfaced; a silently stale bundle serving old pages is worse than a slow boot. Requires hashing and version checks that do not exist today.
- **Build-time vs request-time split bugs.** Anything that accidentally depends on `fs`, `process.cwd()`, build-time `Date`/`Math.random()`, or `src/`-relative paths will behave differently when precompiled. Server-only initializers and per-request `ctx` boundaries need auditing (same class of issue as `0001`'s server/client split).
- **Debugging indirection.** Production stack traces and `__sourceFile` mappings point at bundle code, not `src/pages/*.html`. Sourcemaps or source annotations are needed, otherwise on-call debugging regresses.
- **Middleware/config edge cases.** Middleware loaded by path at serve time vs baked in at build time; `server.middleware` relative-path resolution against `rootDir` vs bundle dir; `PORT`/`host` container logic must stay serve-time. Each needs an explicit rule.
- **Two paths to maintain.** `serve`-from-source and `serve`-from-bundle must both work during transition, doubling test matrix (dev, serve-source, build+serve-bundle, static emit) until the source path can become dev-only.

## Alternatives

- **Do nothing (status quo).** Keep per-boot `buildMultipageGraph` + render-all priming. Cost: cold starts grow with site size, no deployable artifact, source + compiler required at runtime. Acceptable for small sites/demos; increasingly painful for production multipage apps.
- **Cache the graph in memory / on disk opaquely.** Persist `buildMultipageGraph` output (e.g. JSON snapshot in `.chocola/`) and reload if mtimes match, without a designed bundle format. Simpler, but produces an implicit fragile cache (invalidation bugs, version skew, non-portable absolute paths) rather than an explicit portable artifact. Could be a stepping stone, not the end state.
- **Lazy per-route compilation.** Skip render-all priming; compile/render each route on first request and cache. Improves boot time but trades it for first-request latency spikes, keeps source-at-runtime, and still recompiles per deploy. Complementary (bundle serving can also lazy-hydrate), not a substitute.
- **Static-only (pre-render everything).** Extend `emitMultipage` and drop SSR for dynamic routes. Removes the problem by removing the feature: any route needing per-request `ctx` (auth, query, cookies, middleware) stops working. Rejected — SSR with middleware ctx is Kit's purpose.
- **Full bundler delegation (Vite/Rollup/esbuild SSR bundle).** Hand the whole server build to an external bundler with tree-shaking and code-splitting. Most powerful long-term, but heavyweight: new dependency, ESM/CJS interop, `with(ctx)` dynamic-expression constraints, and asset-manifest integration all need solving. This RFC's precompile-to-JS-bundle is deliberately smaller; a full bundler can come later as an optimization of the bundle format.

## Unresolved questions

- Exact bundle path/format: single `.chocola/server.mjs` vs split `server.mjs` + `manifest.json` + `client/` assets? ESM-only or dual ESM/CJS?
- What is embedded vs referenced: are `run-*.js` / `css-*.css` bytes inlined in the bundle or written alongside with hashed filenames and a manifest?
- Staleness policy: warn, error, or silently rebuild when bundle hash mismatches source/config/version?
- Prerender granularity: per-route `prerender: true/false` opt-in/out, or all-routes-primed as today?
- Middleware handling: external-by-default with `--bundle-middleware` opt-in, or always external?
- `createHandler(rootDir)` programmatic API: new signature (e.g. `createHandler({ bundle })`), or auto-detect bundle under `rootDir`?
- Versioning: minimum Node version for bundle import; Windows path handling for embedded manifest paths.
- Benchmarks: cold-start before/after on synthetic 10/100/500-page fixtures; per-request latency parity check to prove the bundle changes boot, not render semantics.

