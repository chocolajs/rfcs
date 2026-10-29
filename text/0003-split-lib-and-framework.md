- Start Date: 2026-09-20
- RFC PR: [#7](https://github.com/chocolajs/rfcs/pull/7)
- Chocola Issue: chocolajs/chocola#185

# Splitting Chocola into Library and Framework

## Summary

Chocola is currently a single monolithic package that bundles the compiler, dev server, SSR capabilities, CLI, and all tooling into one installable unit. This RFC proposes a **split into three packages**: `@chocola/core`, the compiler library; `@chocola/vite-plugin`, the official Vite integration; and `chocola`, the full-stack framework that bundles the compiler, dev server, SSR, and CLI together.

The distinction mirrors a familiar pattern in the JavaScript ecosystem: **React** is both a library (`react`) and the ecosystem/framework people think of when they say "I'm using React." Chocola follows the same model — `@chocola/core` is the library that makes components work anywhere, `@chocola/vite-plugin` is the official bridge to Vite, and `chocola` is the framework that brings the complete toolchain. This means a developer can say "I'm using Chocola" whether they're using just the compiler inside Vite or the all-in-one stack.

`@chocola/core` enables bundler-agnostic component compilation and adoption as a library. `@chocola/vite-plugin` provides the official Vite integration. `chocola` preserves the original experience — `chocola dev`, `chocola build`, SSR, and CLI — for teams that want the complete, all-in-one framework. The brand stays unified. The toolchain stays consistent. The choice is yours.

## Motivation

### What is hard today

Chocola's current structure is a monolith. The `package.json` exports `compiler`, `dev`, and `server` as subpath imports, but they all live in a single repository and are distributed as a single npm package. This creates several problems:

1. **Bundler lock-in.** To use Chocola components, a project must use Chocola's own dev server or build pipeline. There is no way to drop a Chocola component into a Vite project and have it just work. Developers who prefer Vite's DX, plugin ecosystem, or HMR must choose between Vite and Chocola — they can't have both.

2. **Adoption friction.** For Chocola to gain traction beyond its own community, it needs to be easy to try. A project that already uses Vite should be able to add Chocola components with a single package install and a plugin registration — not a full framework migration. Every additional tool a user must download (the native dev server, CLI, SSR machinery) is a barrier to entry.

3. **Ecosystem mismatch.** Every major framework has embraced the plugin model: Vue has `@vitejs/plugin-vue`, Svelte has `@sveltejs/vite-plugin-svelte`, Solid has `vite-plugin-solid`. These plugins let frameworks coexist with Vite's bundling, HMR, and tooling. Chocola has no equivalent, which makes it invisible in the Vite ecosystem.

4. **Maintenance overhead.** The monolith forces all contributors to work within a single release cycle. A bug fix in the compiler shouldn't require a server release, and a CLI improvement shouldn't block a core update. The monolith also means that users who only need the compiler (e.g., for a custom build pipeline) pay the weight of the server, dev tooling, and CLI in their dependency tree.

5. **Brand positioning.** Chocola currently occupies a specific niche — "the sweetest way to build the web" with its own CLI, server, and SSR. But this positioning alienates developers who want to use Chocola components as a *library*, not a *framework*. A split lets Chocola be both: the library that works everywhere, and the framework that handles everything. Just like React.

### What becomes possible

- **Vite integration.** A project using Vite can install `@chocola/core` and `@chocola/vite-plugin`, register the plugin in `vite.config.js`, then start writing Chocola components — no `chocola dev`, no `chocola build`. Vite handles bundling, HMR, and dev server; `@chocola/core` handles components.

- **One brand, multiple experiences.** Users say "I'm using Chocola" whether they're using `@chocola/core` inside Vite, `@chocola/vite-plugin` as a bridge, or the full `chocola` framework with its CLI and dev server. The brand is consistent, just as React is "React" whether you're using the core library or the full React ecosystem. This eliminates the confusing "which Chocola do I choose?" problem.

- **Independent scaling.** `@chocola/core`, `@chocola/vite-plugin`, and `chocola` can be versioned and released independently. A compiler fix doesn't require a framework release, and vice versa. Contributors can focus on one package without needing to understand the entire codebase.

- **The CLI stays familiar.** The `chocola` CLI remains the primary way to run Chocola projects. No one needs to learn a new command. Existing workflows — `chocola dev`, `chocola build`, `chocola serve` — continue unchanged.

- **Third-party ecosystem.** With `@chocola/core` cleanly separated, other tooling authors can build plugins for it — webpack plugins, Rollup plugins, esbuild plugins, and alternatives to `@chocola/vite-plugin` — without forking the Chocola repository.

- **Gradual migration.** Existing Chocola projects continue to work with `chocola` unchanged. New projects can start with `@chocola/core` and `@chocola/vite-plugin`, then add the full framework later if needed. No big bang migration required.

## Detailed design

### Package structure

This splits into three packages:

| Package | Contents | Purpose |
|---|---|---|
| `@chocola/core` | Compiler, parser, runtime, shared utilities | The heart of Chocola — compiles `.html` components to runnable JS. Framework-agnostic. Usable with any bundler. |
| `@chocola/vite-plugin` | Official Vite plugin | Vite integration — component transformation, HMR, build output. Connects `@chocola/core` to Vite's ecosystem. |
| `chocola` | Full-stack framework: `@chocola/core` + dev server + SSR server + CLI | The complete Chocola experience. This is what users install for `chocola dev`, `chocola build`, and SSR. |

### Technical Background

**Current state.** The monolithic `chocola` package (`package.json` version `2.0.0-next.11`) distributes:

- `compiler/` — The build pipeline (`module-graph.js`, `component-processor.js`, `render.js`, `pipeline.js`, `runtime-generator.js`, `config.js`, `utils.js`, `dom-processor.js`, `fs.js`, `chalk.js`, `index.js`).
- `dev/` — Dev server with HMR (`dev/index.js`).
- `server/` — SSR server (`server/index.js`).
- `runtime/` — Client-side runtime (`runtime/index.js`).
- `parser/` — HTML/component parser (`parser/component.js`).
- `bin/` — CLI entry point (`bin/chocola.js`).
- `utils.js` — Shared utilities.

All are shipped as a single npm package with subpath exports (`chocola/compiler`, `chocola/dev`, `chocola/server`). The `bin` field exposes `chocola` and `chjs` commands.

**What changes.** The monolith becomes three packages:

- `@chocola/core` takes `compiler/`, `parser/`, `runtime/`, and `utils.js`. It exports the compiler API (`compile`, `buildModuleGraph`, etc.) and nothing else. It has no Vite integration — that lives in `@chocola/vite-plugin`. This is the minimal package needed to compile Chocola components, usable in any environment that can run Node.js.

- `@chocola/vite-plugin` is a separate package in its own repository (`github.com/chocolajs/vite-plugin`). It wraps `@chocola/core`'s compilation pipeline and integrates it into Vite's build and dev cycles. It handles component transformation, HMR, and build output generation as a Vite plugin. It depends on `@chocola/core` and has `vite` as a peer dependency.

- `chocola` takes the server, the CLI, and the dev server — plus `@chocola/core` as a dependency. It is the spiritual successor to the current monolithic `chocola` package. The `bin` field exposes `chocola` and `chjs` commands. The exports map preserves the current `./compiler`, `./dev`, and `./server` subpaths; `./compiler` re-exports `@chocola/core` for programmatic use.

**Repository split.** `@chocola/core` lives in its own repository at `github.com/chocolajs/core`. `@chocola/vite-plugin` lives in its own repository at `github.com/chocolajs/vite-plugin`. The `chocola` package (the framework) stays in the main `github.com/chocolajs/chocola` repo, which depends on `@chocola/core` via npm.

`@chocola/core` publishes to npm as `@chocola/core`. `@chocola/vite-plugin` publishes to npm as `@chocola/vite-plugin`. The `chocola` repo consumes `@chocola/core` as a dependency. Cross-repo development is managed via npm versioning and CI that runs tests against published versions. This mirrors how React separates `react` and `react-dom` into their own repos while maintaining coordinated releases.

New contributors who want to work on the compiler go to `chocolajs/core`; those who want to work on the Vite plugin go to `chocolajs/vite-plugin`; those who want to work on the framework experience go to `chocolajs/chocola`. Each repo is focused and self-contained.

### Implementation

#### Package boundaries

**`@chocola/core`** — The compiler library. Contains everything in the current `compiler/`, `parser/`, and `runtime/` directories, plus `utils.js`. This is the minimal package needed to compile Chocola components. It has no dependencies on Node.js-specific APIs beyond build-time file system operations. It should be usable in any environment that can run Node.js — including bundlers via their plugin APIs.

Key exports:
- `compile(rootDir)` — Build components for production.
- `buildModuleGraph(rootDir)` — Load and parse the project graph.
- Programmatic `app.build()` API (preserved from current `compiler/index.js`).

**`chocola`** — The full-stack framework package. Depends on `@chocola/core` and adds:
- Dev server with HMR (`dev.server()`)
- SSR production server (`serve()`, `createHandler()`)
- CLI tool (`bin/chocola.js`) with `build`, `dev`, and `serve` commands
- Middleware support
- All existing `chocola.config.json` configuration

This is the package users install for the complete Chocola experience. It is a drop-in replacement for the current monolithic `chocola` package. Existing projects continue to work without modification.

#### Repository structure

```
github.com/chocolajs/core          → @chocola/core (compiler library)
  ├── compiler/
  ├── parser/
  ├── runtime/
  └── utils.js

github.com/chocolajs/vite-plugin    → @chocola/vite-plugin (official Vite plugin)
  ├── src/
  ├── package.json                  → depends on @chocola/core, vite (peer)
  └── ...

github.com/chocolajs/chocola        → chocola (full-stack framework)
  ├── dev/
  ├── server/
  ├── bin/
  └── package.json                  → depends on @chocola/core
```

#### Dependency graph

```
chocola (github.com/chocolajs/chocola)
  └── @chocola/core (github.com/chocolajs/core)
      └── acorn, linkedom

@chocola/vite-plugin (github.com/chocolajs/vite-plugin)
  ├── @chocola/core
  └── vite (peer)
```

Three repos. One ecosystem. New contributors go to `chocolajs/core` for the compiler, `chocolajs/vite-plugin` for the Vite integration, `chocolajs/chocola` for the framework. Each repo is focused and self-contained.

#### How the Vite plugin works

The `@chocola/vite-plugin` package wraps `@chocola/core`'s compilation pipeline and integrates it into Vite's build and dev cycles as a standard Vite plugin:

1. **`transform` hook** — When Vite encounters a `.html` file with Chocola components, the plugin invokes `@chocola/core` to produce the JavaScript output (runtime chunk, component class, template).
2. **`resolveId` hook** — Maps Chocola component imports (`import Action from "./Action.html"`) to virtual module IDs that the plugin can resolve.
3. **`load` hook** — Returns the compiled output for virtual modules.
4. **`configureServer` hook** — For dev mode, the plugin integrates with Vite's dev server to provide HMR updates when components change. The plugin uses Vite's native HMR mechanism instead of Chocola's custom one.
5. **`generateBundle` hook** — During `vite build`, the plugin ensures the Chocola runtime chunks are properly emitted and bundled.

## How we teach this

### The React analogy

Just as React is both a library (`react`) and the framework ecosystem people associate with it, Chocola is both `@chocola/core` and `chocola`. A developer using `@chocola/core` inside Vite is still "using Chocola." A team running `chocola dev` is still "using Chocola." The brand is unified — only the toolchain changes based on the user's needs.

This matters because it eliminates a common friction point in modular ecosystems: "Am I using the library or the framework?" With React, nobody asks this question. The answer is always "React." Chocola should be the same.

### For new users

- **"I use Vite"** → Install `@chocola/core` and `@chocola/vite-plugin`. Register the plugin in `vite.config.js`. Start writing components. You're using Chocola.
- **"I want the full Chocola experience"** → Install `chocola`. Get everything — compiler, dev server, SSR, CLI — out of the box. `chocola dev`, `chocola build`, done. You're using Chocola.
- **"I need custom tooling"** → Install `@chocola/core` directly and build whatever integration you need (webpack plugin, Rollup plugin, esbuild plugin, etc.). You're using Chocola.

### Marketing positioning

- **`@chocola/core`** is the library. It's the compiler. It's what makes Chocola components work anywhere — Vite, webpack, Rollup, or any bundler with a plugin API. Market it as "Chocola Components" or "The Chocola Compiler."
- **`@chocola/vite-plugin`** is the official Vite bridge. It connects Chocola components to Vite's ecosystem. Market it as "Chocola for Vite" — discoverable by Vite users searching for frameworks.
- **`chocola`** is the framework. It's the complete toolchain — compiler, dev server, SSR, CLI, everything. Market it as "Chocola" or "The Chocola Framework." This is the brand that carries the "sweetest way to build the web" identity.
- **The brand is one.** All three packages share the Chocola name, logo, and identity. There is no competing brand — there's just Chocola, used the way you need it.

### Migration path

Existing Chocola projects continue to work unchanged with `chocola`, which is essentially the old monolithic `chocola` package repackaged. The `chocola` CLI remains available. The `chocola.config.json` and all existing project structures are preserved. There is no migration required for current users.

New users are introduced to the package that matches their workflow — Vite users get `@chocola/core` + `@chocola/vite-plugin`, full-stack users get `chocola`. Both are "using Chocola."

## Drawbacks

### Maintenance complexity

- **Three packages instead of one.** Three `package.json` files, version numbers, CI pipelines, and changelogs to track. A change in `@chocola/core` may require coordinated releases in `@chocola/vite-plugin` and `chocola`. Cross-package testing is needed to ensure the framework and the Vite plugin integrate correctly with the library.
- **Dependency management.** `chocola` depends on `@chocola/core`. Version conflicts between releases must be managed carefully.
- **Breaking changes propagate.** If `@chocola/core` changes its public API, both `@chocola/vite-plugin` and `chocola` need to be updated and tested before release. The monolith had this problem too, but it was invisible — everything shipped together. Now it's explicit.

### Marketing tradeoffs

- **Brand ambiguity.** Three packages with the same name could confuse users: "Is `@chocola/core` Chocola? Is `chocola` the real thing? What is `@chocola/vite-plugin`?" The React analogy helps here, but it requires consistent messaging. Documentation and marketing must clearly establish that all three packages are Chocola. This may be resolved organically by users giving packages explicit nicknames, by some other means, or by designing a new marketing strategy in the future.
- **Discovery challenges.** `@chocola/core` and `@chocola/vite-plugin` both appear in npm search results for "Chocola." Users searching for "Chocola framework" may find `chocola` instead of `@chocola/core`, or vice versa. Clear naming conventions and cross-referencing are essential.

### Technical tradeoffs

- **Loss of control.** When developers use `@chocola/vite-plugin` (or any third-party integration) instead of Chocola’s own dev server, the Chocola project loses direct control over the dev experience for those users.

## Alternatives

### Status quo — stay monolithic

Keep the current `chocola` package as-is. No split, no plugin, no additional complexity. The monolith is simple, easy to understand, and works today. The downside is continued bundler lock-in and slower adoption outside the Chocola ecosystem. This is the baseline option and requires zero additional work.

### Full five-package split

The original proposal with five packages (`@chocola/core`, `@chocola/server`, `@chocola/cli`, `@chocola/stacks`, `@chocola/vite-plugin`). This provides maximum flexibility but doubles or triples the maintenance surface compared to the three-package approach. Each package is a separate marketing entity, which fragments the brand. The `@chocola/stacks` package becomes a thin wrapper that adds little value beyond convenience. More packages mean more decisions for users and more coordination for maintainers.

### Community plugin model

Don't create an official Vite plugin. Instead, publish `@chocola/core` and let the community build plugins. This reduces maintenance scope but risks a fragmented plugin ecosystem with varying quality. The Vite plugin may never reach the quality of the native dev server without dedicated maintenance.

## Unresolved questions

- **Repository structure.** Decided. `@chocola/core` lives in its own repo (`github.com/chocolajs/core`), making it easy for new contributors to work with the compiler independently. `chocola` stays in `github.com/chocolajs/chocola`. Cross-repo coordination uses npm versioning and CI that tests against published versions, mirroring how React separates `react` and `react-dom`.
- **Versioning strategy.** Decided. Each package will follow its own SemVer, allowing independent releases with their own dependencies, not forcing the latest `@chocola/core` release on both `chocola` and `@chocola/vite-plugin`.
- **Backwards compatibility.** Decided. Since `chocola` is currently iterating through `next` versions before its v2, it is expected that substantial changes may be implemented. Iteration will continue as usual, and the `2.0.0` release will meet this RFC's specs.
- **Vite plugin maturity.** What exactly does `@chocola/vite-plugin` need to support from day one? Minimum viable: compile `.html` components, HMR for components, `vite build` output. Stretch: SSR integration, middleware support. The scope directly impacts the engineering effort. The plugin lives in its own repo (`github.com/chocolajs/vite-plugin`), so it can be developed and released independently of the core compiler and the framework.
- **Dependency strategy.** Decided. `@chocola/core` will have both `acorn` and `linkedom` dependencies.
- **Documentation.** Will each package have its own documentation, or will there be a unified site? The current `chocolajs.dev` site would need restructuring to cover both the library (`@chocola/core`) and the framework (`chocola`) experiences. A Svelte-like approach, where svelte.dev explicitly contains both docs for Svelte and SvelteKit, may be of use.
- **Testing matrix.** Decided. SemVer and releases will be fully independent, meaning each package will be tested against the released versions of their specified dependencies.