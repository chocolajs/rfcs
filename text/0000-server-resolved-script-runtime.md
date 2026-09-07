- Start Date: 2026-09-06
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# Server-resolved `<script>` with runtime tree-shaking

## Summary

Today the compiler inlines *almost all* top-level `<script>` declarations into the client-side `$runtime` function that is shipped as `run-*.js`. Specifically, `processComponentElement()` (`compiler/component-processor.js:538-565`) and `generateCSRClass()` (`compiler/component-processor.js:99-126`) extract `$runtime` via `parser/component.js:12-48` and then inject **every** top-level `let`/`const` (`extractTopLevelVariables`, `parser/component.js:50-144`) and **every** top-level `function` (`extractTopLevelFunctions`, `parser/component.js:189-268`) plus all `bind:*` bindings and `export let` defaults — regardless of whether they are used inside `$runtime`. `import` lines are stripped unconditionally (`compiler/component-processor.js:262-271`, `601-618`).

This RFC proposes that `<script>` be **resolved server-side**: evaluate and execute server-only declarations during `buildModuleGraph`/`renderPage`, compute reachability from `$runtime`, and inject **only the transitive closure of declarations that `$runtime` actually needs** into the function sent to the client. This includes bundling only client-reachable imports (Chocola components vs JS assets vs unused) and initializing only client-reachable variables/functions.

## Motivation

### What is hard today

A component `<script>` mixes three concerns:

1. **Props** (`export let name = "World"` — `tests/fixtures/basic/src/lib/greeting.html:2`)
2. **Server/template state** (`let items = [1,2,3]` rendered as `{items.length}` or used in `if`/`mount:if`)
3. **Client interactivity** (used inside `function $runtime(){ btn.addEventListener(...) }` — `tests/fixtures/basic/src/lib/counter.html:5-10`)

The current injector does not distinguish them:

```html
<!-- motivation: over-bundled example -->
<script>
  export let title = "Card";
  let serverItems = expensiveFetchSync();   // only for {serverItems.length} in template
  let clientCount = 0;                      // needed in $runtime
  function formatForTemplate(d) { return d.toISOString(); } // server only
  function increment() { clientCount++; }                   // client only
  import Action from "./Action.html";        // imported but maybe unused client-side
  function $runtime() {
    increment();
  }
</script>
<template>
  <div>{serverItems.length} {formatForTemplate(new Date())} {clientCount}</div>
  <Action />
</template>
```

*Today* `run-*.js` (`compiler/runtime-generator.js:18` `DOMContentLoaded` wrapper) ships `serverItems = ctx.serverItems??expensiveFetchSync()` + `formatForTemplate` body + `Action` CSR class (`compiler/component-processor.js:84`, `266`) — none of which `$runtime` needs. The same applies to the CSR class path (`generateCSRClass:101-126`) where `propsParts` (`L134-137`) and `childMappings` are also eager.

Concrete costs:

- **Payload**: every extra `let` initializer and `function` body is hashed into `run-<hash>.js` / `run-<hash>.js` for CSR classes and downloaded by the browser. For a page with 20 components each carrying 2 unused helpers, this is a measurable regression the `bench/` harness would surface.
- **Parse/compile time**: `with(ctx)` functions (`parser/utils.js:5-16`, duplicated `runtime/index.js:6-19`) pay per-byte.
- **Correctness confusion**: `let serverItems = fetch(...)` executed client-side may throw (no server env) or double-fetch. Users currently work around by manually splitting files or not using top-level helpers — but docs (`documentation/05-runtime/01-runtime.md:52`, `documentation/02-components/01-fundamentals.md:95-140`) encourage them.
- **Import semantics**: the regex `/import\s+(\w+)\s+from\s+['"]([^'"]+)['"]/g` only matches `import X from "..."` and strips *all* matches (`return ""` even when `!cx.loadedComponents.has(importedCompName)`). Named imports (`import {x} from "./lib.js"`), `import *`, side-effect `import "./polyfill.js"` and non-component JS assets are mis-handled and would still not be tree-shaken.

There are no GitHub issues yet filed, but this is a prerequisite for the planned state reactivity (`specs/state.md`, `text/0000-state-management.md` RFC) and HMR (`specs/hmr-and-lightweight-ssr.md:136`) where precise client chunk boundaries matter.

### What becomes possible

```html
<script>
  let items = [1,2,3];           // used only as {items.length} → evaluated server-side, not shipped
  let count = 0;                 // used in $runtime → shipped as `let count = ctx.count??0`
  function inc(){ count++; }
  function unusedHelper(){}      // dropped
  function $runtime(){ inc(); btn.addEventListener("click", inc); }
</script>
```

Desired client bundle after resolution:

```js
function r_abcdabcd(self, ctx){
  let count = ctx.count??0;
  function inc(){ count++; }
  btn = self.querySelector('[data-chbind-b0]');
  inc(); btn.addEventListener("click", inc);
}
```

with `items` and `unusedHelper` absent and their initializers already applied to `ctx` during `renderPage` (`compiler/render.js:152-158`).

## Detailed design

### Technical Background

**Compilation pipeline** (`documentation/06-architecture/01-compiler-flow.md`):

1. `buildModuleGraph(rootDir)` (`compiler/module-graph.js:155-210`) loads `chocola.config.json` (`compiler/config.js:4-14`), reads `src/lib/*.html` (`compiler/pipeline.js:6-40`), creates `ChocolaModule{kind:"component"}` (`module-graph.js:17-37`). `compileComponentModule` (`L79-113`) parses `protectCurlyBraces(source)` via `linkedom`, extracts `script/template/styles`, `extractPropsDefaults` (`parser/component.js:1-10`), `extractTopLevelFunctions/Variables`, hashes `cssId`/`fnId`, collects `deps` from `import` regex and template `querySelectorAll("*")`.
2. `renderPage(graph, ctx)` (`compiler/render.js:136-181`) creates DOM (`dom-processor.js:8-13`), runs `processPageConditionals` (`L24-117`) with `compileExpr`, calls `processAllComponents` (`component-processor.js:601-636`), which inits `ProcessContext` (`L15-28` `{runtimeChunks, runtimeMap, csrClasses, staticCtxRegistry}`), pre-scans imports for CSR (`L606-618`), then `processComponentElement` per `getAppElements` (`dom-processor.js:23-25`).
3. `processComponentElement` (`L235-599`): cycle check, `protectCurlyBraces` + `parseHTML`, strip `import` lines (`L262-271`), merge `ctx` from `extractCtxFromEl` (`parser/context.js:4-49`, evaluates `{expr}` via `compileExpr` with `with(ctx)` Proxy) + `globalCtx` (`render.js:152` per-request SSR) + defaults (`L288-298` `compileExpr(defaultValue)`) + top-var/function evaluation for template (`L300-316` `eval`), slot replacement, child loop with `if/mount:if/elif/else`/`bind:*`/`void` and recursive `processComponentElement`, attribute interpolation (`L452-464` `compileExpr(expr,true)(ctxProxy)`), `applyConditionalToElement`, `interpolateNode`. Then **runtime chunk generation** (`L495-572`): `chid` + `ctxDef` + `declared` set, `undeclaredTopVars` → `injectCode`, `bindings` → `querySelector('[data-chbind-bX]')`, `topFuncs` → inject, `runtime.replace(/\$runtime\([^)]*\)\s*\{/, injectCode)` (`L561`), `runtime.replace("$runtime()", fnId+"(self, ctx)")` (`L563`), `runtimeChunks.push(runtime)` + invocation `fnId(querySelector('[chid="..."]'), JSON.stringify(ctx))` (`L571`). Dedup via `runtimeMap` (`L537-566`). CSR path `generateCSRClass` (`L45-147`) mirrors injection but builds `class X extends ChocolaComponent` (`runtime/index.js:173-312`) with `template`, `hash`, `props`, `runtime`, `children`.
4. `generateRuntimeScript` (`compiler/runtime-generator.js:3-23`) emits up to three `run-<hash>.js`: base class (export stripped), CSR classes, `DOMContentLoaded` invocations, appended via `dom-processor.js:57-61` `appendRuntimeScript` / `render.js:171-174`.

**Key properties to preserve**:

- `compileExpr` memoization (`parser/utils.js:5-16`) and `with(ctx)` proxy semantics for `{expr}` in attributes/text/conditionals.
- `self`/`ctx` injection params (`documentation/05-runtime/01-runtime.md:126-131`).
- Deterministic `fnId = "r_"+deterministicHash(compName,8)` (`compiler/utils.js:112-114`) and `compId = "chid-"+deterministicHash(compName+":"+idx,8)` (`component-processor.js:500-501`).
- Isomorphic `ChocolaComponent#init` (`runtime/index.js:214-251`) `document.createElement("template").content`, `processDirectives`/`interpolateAttributes`/`collectBindings`/`#collectListeners`.

Prior art: Svelte/Solid/Vue SFC compilers perform AST-based `<script>` analysis, split `setup` server/client, and tree-shake via Rollup-esque reachability. Chocola can adopt a lighter, no-bundler precursor: per-component reachability from `$runtime` before falling back to a full bundler.

### Implementation

#### Overview

Introduce a **Script Resolver** phase between `extract*` and `injectCode` that, per `ChocolaModule`:

1. Parse `script` to ESTree AST (candidate: `acorn` + `acorn-jsx` is ~15kB, already used transitively by `linkedom`’s deps; alternative `meriyah`).
2. Classify top-level declarations: `ExportLet`, `Let/Const`, `ImportDeclaration`, `FunctionDeclaration`, `FunctionDeclaration:$runtime`.
3. Compute **client-reachable set**: traverse `$runtime` body AST, collect `Identifier` references (excluding `self`, `ctx`, globals), transitively expand through declarations until fixed point. Treat `bind:*` vars as reachable if referenced inside `$runtime` (otherwise keep only if no `$runtime` — server-only components still warn but ship nothing).
4. Evaluate **server-only** declarations via existing `compileExpr` / `eval` path and *exclude* from `injectCode`.
5. Emit client `injectCode` solely from reachable set, preserving source via `script.slice(start,end)` to avoid codegen.

#### Step-by-step

**A. AST extraction (replace/augment regex helpers)**

- Add `parser/script.js` exposing `parseScript(script) → { ast, props, topVars, topFuncs, imports, runtimeNode }`. Internally use `acorn.parse(script, {ecmaVersion:2023, sourceType:"module"})`. Keep `parser/component.js` shims deprecated but exported for back-compat.
- Handle comma-declarators, `const {a,b}=obj`, `let [x,...rest]=arr`, `import {x} from "./lib.js"`, `import * as ns from "./ns.js"`, `import "./side.js"` robustly. Update `compiler/module-graph.js:94-100` and `component-processor.js:262-271` to consume `ast.imports` instead of `path.basename(regex)` trick — resolve relative `importPath` against `module.sourcePath` via `path.resolve`/`path.posix.join`, then `graph.component()` / `graph.moduleById()` lookup.
- Note: `utils.js:20-44` `protectCurlyBraces` must run *before* DOM parse but *after* AST parse (or AST parse should operate on original `script.innerHTML`, not protected).

**B. Reachability**

- If no `runtimeNode` → `needed = ∅` (or `needed = bindVars` if template-less CSR needs them — see unresolved). Current code still injects helpers into CSR runtime even when `runtime==null`; new behavior would skip.
- Else: `neededVars = new Set()`, `neededFuncs = new Set()`, `neededImports = new Set()`. Seed with identifiers in `runtimeNode.body` (skip `self`, `ctx`, `document`, `window`, globals allowlist). For each `id`, if it matches a top-var/function/import binding, add and recursively scan that declaration’s initializer/body AST for further identifiers (worklist). For imports, check if import binding is reachable; if so, mark that `ChocolaModule` dep as client-needed, else ignore.
- Special cases: `bind:*` variables (`L438-448` `varName`) are considered provided by `data-chbind` injection; they are included only if referenced in `$runtime`. Props (`export let`) are included only if referenced (but defaults still needed for `ctx` server evaluation). `ctx.*` property accesses count as `ctx` use, not declaration.
- String/quasi evaluation: if `$runtime` does `ctx["dyn"+x]` or `with(ctx)` style (rare), conservative fallback = include all. Initial implementation can warn and include all when `compileExpr` patterns are dynamic.

**C. Server resolution**

- Keep existing evaluation order `L288-316`: props defaults → top vars → funcs into `ctx` via `compileExpr(value)` / `(0,eval)`. Change: *always* evaluate all declarations server-side (needed or not) so template `interpolateNode`/`applyConditionalToElement` sees values. For needed declarations, also produce hydration guard string `let name = ctx.name??(<src>)` (or `const`) exactly as before (`L547-549`, `L110-112`) but filtered. For unneeded, emit nothing client-side.
- Preserve `staticCtxRegistry` (`L276-285`) per-request SSR with `globalCtx` (from `server/index.js:247-250` `query` + middleware).

**D. Injection rewrite**

- In `processComponentElement:L541-562` and `generateCSRClass:L101-126`:

  ```js
  // before: for (const v of topVars) / topFuncs
  // after:  for (const v of neededVars) / neededFuncs
  //         for (const imp of neededImports) generateCSRClass(imp, cx)
  ```

- For `generateCSRClass`, `propsParts` (`L130-138`) should include only reachable vars if runtime exists; otherwise keep empty (or all for CSR-only instantiation via `new X().mount`). `childMappings` (`L74-87`) should include only tags reachable from `template` *and* runtime children usage? For now keep template scan but note unresolved.

- Preserve dedup via `runtimeMap` (`L537-539`): `fnId` is still per `compName`; second instance reuses function. Since `needed` is computed per `compName` (not per instance ctx), caching is sound. If future per-instance specialization (e.g., `mount:if` removed branches) is desired, key by `needed` hash.

**E. Module graph & assets**

- `compiler/module-graph.js:79-113` `compileComponentModule` stores `module.imports = parsed.imports` and `module.neededClientImports` after resolver. `page.deps` remains template deps; client `run-*` deps become minimal.
- Non-component JS imports (e.g., `import { debounce } from "./utils.js"`) — if reachable, need bundling. Phase 1 of this RFC: **warn and drop** (or inline as `import` in `run-*.js` with `<script type="module">`). Phase 2 would integrate `esbuild`/`rollup` or current `pipeline.js:93-123` `processScript` hashing for assets (already does `css-*`/`js-*` hashing). Unreachable JS imports are dropped silently.
- Tests: `tests/compiler.test.js:65-92` graph expectations update: `card.html` depends on `lib/action.html` via both tag and import — reachable case retains; a new component `unused-import.html` with an untracked import should *not* produce a `deps` entry in client `hashMap` but still count in graph if template uses it.

**F. Preservation & migration**

- No public API change: `function $runtime(){}` signature stays; `self`/`ctx` injection unchanged (`L563` `fnId(self, ctx)`). Users see smaller `run-*.js` transparently.
- Add `warnUnusedDeclaration` (`compiler/utils.js:17-22`, `component-processor.js:170-233`) remains, but its `countIdentifiers` (`L152-158`) is superseded by AST reachability — can keep as diagnostic until resolver stabilizes, then align thresholds.
- Feature flag: `chocola.config.json` `{ "compiler": { "treeShakeRuntime": false } }` to revert to current eager injection for one minor.

**G. Testing & bench**

- Unit: `parser/script.test.js` for edge cases (destructuring, comments containing `function $runtime`, `//` skipping, nested braces in template literals).
- Integration: `tests/fixtures/basic` added `unused.html` with `let a=1, b=2, function h1, h2, $runtime uses only b/h2` — assert `run-*.js` content lacks `a`, `h1`, and size delta.
- Bench: extend `bench/index.js` to report `runtimeScript.length` before/after for 100-component synthetic page — expected 20-60% reduction when helpers are abundant (quantified in PR).

### Alternatives considered within design

- **Regex-only filtering** (current `BARE_IDENTIFIER_RE` `L149`): brittle against `a.b`, `ctx.a`, string contents, arrow functions.
- **Bundler delegation** (esbuild pre-bundle `<script>`): correct but heavyweight; SSR still needs `compileExpr` eval for `{expr}`. Deferred to Phase 2.
- **Manual annotations** (`// @client`): low effort but poor DX and breaks isomorphism.

## How we teach this

This is an **internal compiler optimization**, not a new concept. Existing teaching remains:

- `documentation/02-components/01-fundamentals.md:95-140` ("top-level `let/const` become part of context") stays true server-side; clarify that only *client-reachable* state is rehydrated. Add a note:
  > Anything you declare at the top level is available in your template and is evaluated server-side. Only values transitively used inside `$runtime` are shipped to the browser.

- `documentation/05-runtime/01-runtime.md` add a short "What gets shipped" subsection showing a server-only vs client-needed example and how to verify via `dist/run-*.js`.

- `documentation/06-architecture/01-compiler-flow.md:101-104` steps 3-5 update from "injects prop variable declarations, top-level variable declarations, and helper function definitions" to "injects only client-reachable declarations".

No `PULL_REQUEST_TEMPLATE.md` docs churn beyond changelog entry referencing RFC. Migration is automatic; no codemod.

## Drawbacks

- **Parser weight**: adding `acorn` (~15kB) to `compiler/*` (Node-only) is fine for build, but adds dep audit surface. Kept server-only; not shipped to `runtime/index.js`.
- **`with(ctx)` dynamism**: `compileExpr` + `new Proxy(ctx,{has(){return true}})` (`component-processor.js:320-323`) makes static reachability *sound but conservative*. Dynamic `eval`, `ctx["x"+y]`, `this[...]`, or `Function` constructor may force fallback to "include all" + warning.
- **Spec risk**: `function $runtime(self, ctx)` explicit params today (`tests/fixtures/basic/src/lib/action.html:3`) — AST must respect `params` but injection still rewrites to `function r_hash(self, ctx)` (`L563`, `L125`). Reachability on params-qualified uses needs care.
- **Import resolution**: `path.basename(importPath).toLowerCase()` today is forgiving but case-insensitive; AST resolver must preserve that while handling `../`/`./lib/` prefixes correctly across `config.libDir` (`compiler/config.js`).
- **Caching complexity**: `runtimeMap` per `compName` deduplicates `runtimeChunks`; reachable set must be deterministic per `compName` or cache key expands.
- **Debugging**: dropped declarations mean `dist/run-*.js` less closely mirrors `<script>` source; sourcemaps/line numbers (`utils.js:59-85` `findElementLine`) should map remaining injected lines back to original `__sourceFile`.

## Alternatives

- **Do nothing (status quo)**: keep eager injection; rely on gzip to hide waste and on `warnUnusedDeclaration` (`L170-233`) for manual cleanup. Impact: payload stays, HMR/SSR specs remain fragile, self-inflicted double-fetch bugs.
- **Manual `@client` / `@server` pragmas**: `// @client` over declarations to force inclusion. More explicit control but pollutes `<script>` with bundler-like directives; alternatives section of `state` RFC already dismissed similar pragmas.
- **Full esbuild integration**: parse every `<script>` via `esbuild` with `bundle:true`, `treeShaking:true`, `format:"iife"` targeting `$runtime`. Gives correct JS import handling but requires broadening `compiler/index.js:88-103` `compile()` to async bundle per component, handling `paths.src`/`outDir` assets differently, and SSR needs separate no-bundle evaluation. Considered as Phase 2 after this RFC lands.
- **Svelte-style separate `onMount`**: split server and client scripts (`<script server>` / `<script client>`). More invasive API change; rejected in `documentation/03-templates/05-bind.md`-style `bind:*` continuity.

Other frameworks: Svelte compiles `n$.js` blocks with reachability via Rollup; Solid uses `"use server"` directive; Vue SFC `<script setup>` does compile-time macro elimination — all lend precedent for AST-based elision this RFC proposes in minimal form.

## Unresolved questions

- Should `generateCSRClass` tree-shake `childMappings` based on `$runtime` usage (e.g., `new Action().mount`) versus template presence? Today CSR class includes all template child tags (`L78-87`); if template child is server-only (`mount:if={false}`), shipping its class is wasteful.
- How to handle **side-effectful initializers** (`let x = Math.random()` or `let y = localStorage.getItem(...)`): server evaluation may give different values than client hydration. Current `ctx.x??(expr)` guards preserve client re-evaluation; should we warn when server-only elision would change semantics?
- Do we need **per-instance** specialized runtimes (when `if` prunes branches per ctx) vs per-component shared `fnId`? Sharing is smaller but slightly over-includes.
- Exact **import bundling** for JS assets: inline `import` vs `processScript` `js-<hash>.js` + `<script src>` linking? How does `runtime/index.js` `ChocolaComponent#mount` interact with `type="module"` runtime chunks?
- Fallback granularity when reachability is undecidable (e.g., `eval("count")`): include all + warning, or error? Preference for warn + include-all in config `strict: false`.
- Should resolver live in `parser/script.js` reused by both `compiler/module-graph.js` and `server/index.js:183-375` `createHandler` per-request rendering, with LRU cache keyed on `script+template` hash?

