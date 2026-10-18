- Start Date: 2026-10-18
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# Import-based component graph with `.chjs` components and a pure compiler

## Summary

Today the Chocola compiler discovers every `.html` component under `src/lib` (`compiler/pipeline.js:6-42`), builds a global component map keyed by lowercased filename (`compiler/module-graph.js:215-231`), and resolves component usage in templates by turning the tag name into `<tag>.html` and looking it up (`compiler/module-graph.js:146-152`, `compiler/component-processor.js:300`). Imports are matched by basename (`compiler/module-graph.js:116-141`, `compiler/component-processor.js:342-357`), `srcDir`/`libDir` are baked into config and paths (`compiler/config.js:6-40`), the entry is a discovered `src/index.html` page that must contain an `<app>` (`compiler/pipeline.js:44-59`, `compiler/dom-processor.js:15-25`), and the compiler writes the final site, scoped CSS, runtime chunks, assets and `.chocola/hashes.json` to disk (`compiler/index.js:12-44`).

This RFC proposes to turn the compiler into a **pure, host-agnostic transform**:

- The compiler compiles **one root `.chjs` component and its imports**, nothing else. Component discovery is deleted.
- Components are **`.chjs` files** (same `<script>`/`<template>`/`<style>` blocks). `.html` component support is removed entirely.
- A component used in a template **must be imported**. A tag that is neither a static HTML/custom element nor an imported component is a **compile error**.
- `srcDir` and `libDir` disappear. The entry is a **required absolute path** to the root `.chjs`.
- The compiler **never writes files** and **never resolves project paths**: it returns the compiled fragment, scoped CSS and runtime descriptors in memory.
- The compiler **never reads config**: config becomes an optional plain object argument, and hosts load `chocola.config.json` themselves.
- Imports resolve **relative to the importing file**; module identity is expressed **relative to the root component**.
- Component usage supports both **`<MyComponent />`** and **`<MyComponent></MyComponent>`** (self-closing is optional). Child projection keeps `<MyComponent>children</MyComponent>`.

Document assembly, entry enforcement (`app.html`), `<chocolakit:body>` injection, routing, and JS/asset bundling become **host responsibilities** (ChocolaKit, Vite plugins, custom tooling).

## Motivation

### What is hard today

**1. Implicit global component namespace.** Any `.html` file dropped in `src/lib` becomes a globally-available tag. Two files with the same basename in different folders collide, because the map is keyed by lowercased filename (`compiler/pipeline.js:28`). There is no way to alias a component, and moving a file silently changes (or breaks) its tag name.

```html
<!-- src/lib/Card.html -->
<template><div class="card"><slot></slot></div></template>
```

```html
<!-- src/index.html: which Card is this? Whichever file the scanner found. -->
<app><Card></Card></app>
```

**2. Imports are decorative.** `import Action from "./Action.html"` is parsed (`compiler/module-graph.js:116-141`) but only its **basename** is used (`compiler/module-graph.js:123`). It does not actually bind the tag, does not scope the namespace, and cannot be aliased. A component can be used without being imported at all, so there is no compile-time guarantee that everything is wired.

**3. Project-structure coupling.** `srcDir` and `libDir` are core config keys (`compiler/config.js:18-29`), `resolvePaths` projects them (`compiler/config.js:34-40`), and `processStylesheet`/`processIcons`/`processScript`/`copyStaticDir` hard-code `rootDir/srcDir` (`compiler/pipeline.js:61-135`). This prevents using Chocola as a pure compiler behind Vite/ChocolaKit, and forces a `src/` layout even when the host owns the entry.

**4. Writing and config I/O in core.** `compile()` calls `emit()` which creates/cleans `outDir`, writes `index.html`, CSS/JS files, copies icons/static, and writes `.chocola/hashes.json` (`compiler/index.js:12-44`). `loadConfig` reads `chocola.config.json` from disk (`compiler/config.js:4-14`). Hosts that already manage output and config cannot compose the pipeline without side effects.

**5. A page concept that hosts should own.** The core requires `src/index.html` with an `<app>` (`compiler/pipeline.js:44-59`, `compiler/dom-processor.js:15-25`) and runs page-level conditional logic (`compiler/render.js:24-117`). Hosts like ChocolaKit instead want to compile a root component and inject its fragment into their own document (e.g. a `<chocolakit:body>` tag), with their own routing.

**6. HTML parser vs. component syntax.** In HTML, a self-closing slash on a non-void element is ignored, so `parseHTML("<A /><B />")` makes `B` a **child of `A`**. Chocola currently sidesteps this because components always use explicit open/close tags, but adopting self-closing usage requires the compiler to normalize templates before they reach the HTML parser.

### What becomes possible

Imports become the single source of truth for template component usage:

```html
<!-- app.chjs (root component) -->
<script>
  import Greeting from "./lib/greeting.chjs";
  import Card from "./lib/card.chjs";
  import Action from "./lib/action.chjs" as Button;   // aliasing (future)
</script>
<template>
  <div>
    <Greeting name="World" />        <!-- optional self-closing -->
    <Greeting name="Also"></Greeting>
    <Card title="Hi">
      <span>slot content</span>      <!-- child projection -->
    </Card>
  </div>
</template>
<style>
  div { display: block; }
</style>
```

```html
<!-- lib/card.chjs: imports resolve relative to THIS file -->
<script>
  import Action from "../action.chjs";
</script>
<template>
  <div class="card"><h2>{title}</h2><slot /><Action /></div>
</template>
```

- Using an unimported tag is a compile error with a file/line, like Svelte.
- Two components named `card.chjs` in different folders no longer collide.
- A host can compile any root component and receive a fragment + runtime, with no filesystem or config side effects.
- Building an app is "give me a root component; I follow its imports."

## Detailed design

### Technical Background

**Current pipeline** (`documentation/06-architecture/01-compiler-flow.md`):

1. `loadConfig(rootDir)` reads `chocola.config.json`, defaulting `srcDir="src"`, `libDir="lib"`, `outDir="dist"` (`compiler/config.js:4-32`); `resolvePaths` projects them to absolute paths (`compiler/config.js:34-40`).
2. `buildModuleGraph(rootDir)` (`compiler/module-graph.js:198-248`) loads `src/index.html` as the page (`compiler/pipeline.js:44-59`), scans every `.html` file under `src/lib` (`compiler/pipeline.js:6-42`), and creates a `ChocolaModule` per file keyed by lowercased basename. `compileComponentModule` (`compiler/module-graph.js:80-156`) parses `protectCurlyBraces(source)` with linkedom, extracts sections, hashes `cssId`/`fnId`, then records deps from `import` statements (by basename) and from template tag scans (`tag + ".html"`).
3. `renderPage(graph, ctx)` (`compiler/render.js:136-183`) creates the page DOM, requires `<app>` (`compiler/dom-processor.js:15-21`), runs page conditionals, expands components via `processAllComponents` (`compiler/component-processor.js:837-903`) which looks tags up in the global `loadedComponents` map (`compiler/component-processor.js:300`), then processes page assets (`compiler/render.js:119-134`).
4. `emit(graph)` writes `index.html`, hashed CSS/JS, copies, and `.chocola/hashes.json` (`compiler/index.js:12-44`).

**Key mechanisms to preserve**: scoped CSS hashing (`deterministicHash`, `compiler/utils.js:89-96`), stable runtime ids (`runtimeFunctionId`, `compiler/utils.js:98-100`), `$runtime` extraction and client reachability tree-shaking (`parser/script.js:336-482`), `compileExpr`/`protectCurlyBraces` semantics (`parser/utils.js`, `utils.js:7-31`), slot projection, conditional chains (`if`/`mount:if`/`elif`/`else`, `<void>`), and the `ChocolaComponent` runtime (`runtime/index.js`).

**HTML parsing constraint.** linkedom follows the HTML parsing algorithm: for non-void elements the trailing `/>` is ignored, so `<A /><B />` nests `B` inside `A`. Any self-closing component syntax must therefore be normalized to explicit open/close pairs **before** `parseHTML`.

**Prior art.** Svelte requires `import X from "./X.svelte"` and errors on unknown components; Astro/Vue SFCs are single-file components with an extension that signals the compiler; Rollup/Vite own bundling, output, and document assembly. This RFC aligns Chocola's core with that separation.

### Implementation

#### Overview

- **Entry:** `buildModuleGraph(rootComponentPath, { config } = {})` where `rootComponentPath` is a required absolute path to a `.chjs` file (or a path to the root component directory plus name), and `config` is an optional plain object.
- **Graph:** rooted at the root component; components discovered by following `.chjs` imports; a per-module `importMap` (lowercased import local → module id) drives template tag resolution; unknown tags throw.
- **Render:** `render(graph, ctx)` returns `{ html, styles, runtime, hashMap }` in memory; no writes, no document assembly.
- **Config:** pure `normalizeConfig(config)`; no file I/O, no path projection.
- **Templates:** a normalization pass expands self-closing component tags to explicit pairs before parsing; explicit pairs remain valid.

#### A. Public API and purity (`compiler/index.js`, `compiler/config.js`)

```js
// compiler/index.js
export async function buildModuleGraph(rootComponentPath, { config } = {});
export async function render(graph, ctx = {});
export async function compile(rootComponentPath, { config, ctx } = {}); // pure convenience
```

`render` returns:

```js
{
  html,       // root component compiled to an HTML fragment
  styles,     // scoped CSS string (may be "")
  runtime,    // [{ name: "run-<hash>.js", content }] base class + CSR classes + DOMContentLoaded chunk
  hashMap     // { "lib/card.chjs": "<cssId>" } (debug aid; hosts may persist)
}
```

- Remove `emit`, `app`, `setupOutputDirectory`, `writeHTMLOutput`, `.chocola/hashes.json` writing, `copyStaticDir` invocation, `appendStylesheetLink`, `appendRuntimeScript`, `serializeDOM`-based document writing.
- `rootComponentPath` must be absolute; otherwise throw a clear error.
- `compiler/config.js` becomes:

```js
export function normalizeConfig(config = {}) {
  return { treeShakeRuntime: true, ...config };
}
```

  Only compile-behavior flags live here. No `srcDir`, `libDir`, `outDir`, no `chocola.config.json` reading. Hosts load their own config and pass it in.

#### B. `.chjs` components (remove `.html` components)

- Component files use the `.chjs` extension; internal format is unchanged: `<script>` (optional), `<template>` (required), `<style>` (optional).
- Replace every `.html` component assumption: import detection (`compiler/module-graph.js:117`, `compiler/component-processor.js:343`), basename-derived names (`path.basename(...).replace(".html", ...)`), tag-name lookup (`tag + ".html"`), and docs/fixtures/bench.
- Non-`.chjs` imports remain in `module.imports` but are ignored by the graph — bundling JS/assets is a host concern.

#### C. Import-based graph (`compiler/module-graph.js`, `compiler/pipeline.js`)

- `buildModuleGraph(rootComponentPath, { config })`:
  1. Validate the absolute root path and read it; `baseDir = path.dirname(rootComponentPath)`.
  2. `graph.root = loadModule(rootComponentPath)`.
  3. Breadth-first over unparsed modules:
     - Parse `<script>` with `parseScript` (`parser/script.js:62-204`).
     - For each import whose source ends with `.chjs`: resolve it **relative to the importing file** (`path.resolve(path.dirname(module.sourcePath), imp.source)`); `checkFile` → throw `Cannot resolve import "<src>" from <moduleId>`; `loadModule(abs)`; add to `module.deps`; for each specifier set `module.importMap.set(spec.local.toLowerCase(), child.id)` (default and named specifiers). Side-effect imports add a dep without a tag entry.
     - Scan `<template>`: for each element, if its lowercased tag is in `importMap`, add the dep; if `isStaticTag(tag)` leave it; otherwise **throw** `Unknown component <foo>. Did you forget to import it?` with `moduleId:line` (reuse `findElementLine`, `compiler/utils.js:45-71`).
  4. No page loading (`getSrcIndex` removed), no directory scan (`getComponents` removed), no asset modules.

- `ChocolaModule`:
  - `id`: **root-relative posix path** from `baseDir` (e.g. `app.chjs`, `lib/card.chjs`).
  - `sourcePath`: absolute path.
  - `isRoot`: boolean.
  - `importMap: Map<tagLower, moduleId>`, `deps: Set<moduleId>`.
  - Keep parsed `script`/`template`/`styles`/`imports`/`compProps`/`topFuncs`/`topVars`/`cssId`/`fnId`.
- Hashing by `module.id` (`deterministicHash(module.id, 8)`, `runtimeFunctionId(module.id)`) so components with equal basenames no longer collide.
- New `compiler/tags.js`: `KNOWN_HTML_TAGS` (HTML/SVG/MathML plus `template`, `script`, `style`, `slot`, `void`) and `isStaticTag(tag)` = in the list or hyphenated (custom element).
- `compiler/pipeline.js`: delete `getComponents`, `getSrcIndex`, `processStylesheet`, `processIcons`, `processScript`, `copyStaticDir`; keep read-only `readMyFile`/`checkFile` (`compiler/fs.js`).

#### D. Dynamic vs. static tags, aliasing, cycles

- **Tag ↔ import binding.** The tag matches an import's **local name**, case-insensitively (linkedom lowercases tag names). `import Btn from "./lib/card.chjs"` lets the template use `<Btn />`.
- **Static tags.** Standard HTML/SVG/MathML elements and hyphenated custom elements are emitted as-is.
- **Cycles.** Import cycles are allowed at graph level; rendering cycle detection follows module ids (`renderChain`), as today.
- **Aliasing / named imports.** Multiple specifiers from one `.chjs` map several tag names to the same module; this enables future import-aliasing syntax. Namespace imports (`import * as ns`) do not produce a tag binding and are out of scope for the first iteration.

#### E. Self-closing and explicit tags (template normalization)

Self-closing usage is **optional**; both forms are supported:

```html
<MyComponent />             <!-- no children -->
<MyComponent></MyComponent> <!-- equivalent, no children -->
<MyComponent>children</MyComponent> <!-- child projection -->
```

Because linkedom ignores `/>` on non-void elements, add `expandSelfClosingComponents(templateSource, importLocals)`:

- Scan the raw template, **skipping `<script>`/`<style>` regions** (same split approach as `protectCurlyBraces`, `utils.js:7-20`).
- For a tag whose name matches an imported component local (case-preserving at the source level), rewrite `<Local ... />` → `<Local ...></Local>`. Explicit pairs are left untouched.
- Also normalize `<slot />` → `<slot></slot>`.
- Use a quote-aware scan so attribute values containing `/` or `>` are not mis-split.
- Run **before** `protectCurlyBraces` + `parseHTML`, once per module, after the script has been parsed to obtain `importMap`.

Centralize this in `parseModuleSource(module, importMap)` returning expanded `script`/`template`/`styles`, and have both graph building and `component-processor` consume it (instead of re-parsing raw `source`), so every stage sees the same normalized template.

Custom elements written self-closing (`<my-widget />`) share the HTML parse limitation but are out of scope (host concern); they require explicit close tags.

#### F. Rendering (`compiler/component-processor.js`, `compiler/render.js`, `compiler/dom-processor.js`, `compiler/runtime-generator.js`)

- Replace the global `loadedComponents` map with module-scoped resolution: `ownerModule.importMap` + `cx.graph.moduleById`. Every `cx.loadedComponents.has/get(...)` (`compiler/component-processor.js:71,108,301,348,366,617,842-880`) is rewritten.
- Extract the per-instance render body of `processComponentElement` into `renderInstance(module, usageElement | null, cx, renderChain, sourceFile, sourceContent, globalCtx)`, shared by the root component (`usageElement = null`) and nested components. `renderChain` tracks module ids.
- `generateCSRClass(module, cx, explicitClassName)` resolves nested imports through `module.importMap` and uses `.chjs` names.
- `render(graph, ctx)` renders `graph.root`, collecting scoped styles (`compiler/component-processor.js`), runtime chunks and CSR classes, then returns `{ html, styles, runtime, hashMap }`. Remove `validateAppContainer`, `getAppElements`, `processPageConditionals`, page asset processing, and all document-writing helpers.
- `compiler/dom-processor.js` keeps only fragment parse/serialize helpers. `compiler/runtime-generator.js` keeps its logic but returns `{ name, content }` descriptors instead of implying writes.

#### G. Host responsibilities (out of core)

- Choosing/validating the entry (`app.html`), owning `chocola.config.json`, and passing it as `config`.
- Assembling the final document (e.g. injecting the fragment into `<chocolakit:body>`), routing, and mapping intercepted components to pages.
- Bundling non-`.chjs` JS imports and assets, and persisting returned styles/runtime.

### Migration

- Rename components from `.html` to `.chjs`.
- Add `import X from "./X.chjs"` in every file that uses `<X>` (resolve paths relative to the importing file).
- Replace `buildModuleGraph(rootDir)` / `app.build(rootDir)` / `compile(rootDir)` calls with the root component path (`buildModuleGraph(entryAbsPath, { config })`, etc.).
- Stop relying on `emit`, `outDir`, static-dir copying, and config discovery; consume `{ html, styles, runtime }` and handle output in the host.
- Remove `srcDir`/`libDir` from configs.
- Replace the `<app>` page with a root component (or a host-managed document).

## How we teach this

This is a **mental-model shift** and must be taught as such, following Svelte's precedent ("import your components; the compiler follows imports"):

- **Components guide** (`documentation/02-components/02-usage.md`): "Import a component before using it. The import's local name is the tag name; imports resolve relative to the file that contains them. Self-closing (`<MyComponent />`) and explicit (`<MyComponent></MyComponent>`) are both fine; children project into the component's `<slot>`."
- **Getting started / FAQ** (`documentation/01-introduction/02-getting-started.md`, `documentation/07-reference/01-faq.md`): replace `src/index.html`/`<app>` guidance with "the host passes a root `.chjs` component; the core returns a fragment."
- **Architecture** (`documentation/06-architecture/01-compiler-flow.md`): rewrite stages — no config stage/IO, no discovery, `.chjs`, import-driven graph, template normalization, pure output, host duties.
- **Compiler errors** are the primary teacher: `Unknown component <Foo> in lib/bar.chjs:12. Did you forget to import it?`

Migration is mechanical (extension rename + import insertion); no codemod is strictly required, but a simple codemod that adds imports based on tag usage is possible.

## Drawbacks

- **Breaking for everyone**: every project must rename files and add imports; every host must adopt the new API and own document/output/config.
- **Loss of zero-config convenience**: no directory scanning means small demos must write imports (matching modern frameworks, but a change).
- **Strictness**: unknown-tag errors can be noisy for typos and for projects relying on custom elements without hyphens.
- **Template normalization cost/complexity**: an extra source pass before HTML parsing; correctness depends on robust quote/region handling.
- **Tag casing information loss**: after linkedom parsing, tags are lowercase, so errors report lowercase names, and matching component locals is case-insensitive.
- **JS imports become host work**: core no longer bundles or rewrites non-`.chjs` imports, so bare `npm`-style components and JS utilities require host support.

## Alternatives

- **Keep discovery, add imports**: preserve `getComponents` and let imports be optional aliases. Rejected: keeps the global namespace, collision and structure coupling the issue targets.
- **Resolve imports by basename (status quo)**: rejected: prevents aliasing, scatters namespaces, and does not guarantee declared components are imported.
- **Root-relative import resolution for all files**: simpler mental model, but confirmed to be relative-to-importing-file instead; per-file resolution matches ESM and host expectations.
- **Require self-closing for childless components** (error on `<X></X>`): rejected in favor of supporting both forms for ergonomics and migration ease.
- **Custom template parser instead of normalizing for linkedom**: more control, but a large rewrite of template handling for little gain; normalization is local and testable.
- **Keep a page/`<app>` concept in core**: rejected; hosts own documents and routing.
- **Keep `emit`/file writes**: rejected; core should have no side effects so hosts can compose it.

Prior art: Svelte (required imports, `.svelte` extension, compile errors for unknown components), Vue/Astro (SFC extension, import-based usage), Rollup/Vite (bundling/output/document ownership by the host).

## Unresolved questions

- Should the entry be `buildModuleGraph(rootComponentPath)` (path first) or `buildModuleGraph(rootDir, { entry })`? The former is proposed; a config/override `entry` may still be useful for hosts.
- How should **namespace imports** (`import * as ns from "./x.chjs"`) map to tags, if at all?
- Should hyphenated custom elements self-closing syntax also be normalized, or left to hosts?
- Do we still need `hashMap` in the return value, or should scoped CSS/runtime descriptors be self-contained?
- How are **bare/package imports** (`import X from "chocola-router"`) surfaced to hosts, and should core pass them through in a manifest?
- Should `.chjs` also be accepted as the script-only form, or reserved for full SFC components?
- Error policy for near-miss tags (e.g. suggest the closest imported name), and whether unknown-tag strictness is configurable.
- Interaction with the in-progress server-resolved `<script>` RFC (`text/0001-server-resolved-script-runtime.md`): import reachability/tree-shaking must be recomputed against the module graph rather than basenames.
