- Start Date: 2026-09-20
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# Modularization: Splitting Chocola into Packages

## Summary

Chocola is currently a single monolithic package that bundles the compiler, dev server, SSR capabilities, CLI, and all tooling into one installable unit. This RFC proposes splitting Chocola into separate publishable packages — `@chocola/core`, `@chocola/server`, `@chocola/cli`, `@chocola/stacks`, and `@chocola/vite-plugin` — so that the compiler can be consumed independently by projects using other build tools and frameworks, starting with Vite.

The goal is to make Chocola a first-class citizen in the broader JavaScript ecosystem, not just a self-contained framework with its own CLI and server. Users should be able to use Chocola components inside Vite, Astro, or any bundler that supports custom plugins, without being forced to adopt Chocola's full toolchain.

## Motivation

### What is hard today

Chocola's current structure is a monolith. The `package.json` exports `compiler`, `dev`, and `server` as sub-path imports, but they all live in a single repository and are distributed as a single npm package. This creates several problems:

1. **Bundler lock-in.** To use Chocola components, a project must use Chocola's own dev server or build pipeline. There is no way to drop a Chocola component into a Vite project and have it Just Work. Developers who prefer Vite's DX, plugin ecosystem, or HMR must choose between Vite and Chocola — they can't have both.

2. **Adoption friction.** For Chocola to gain traction beyond its own community, it needs to be easy to try. A project that already uses Vite should be able to add Chocola components with a single package install and a plugin registration — not a full framework migration. Every additional tool a user must download (the native dev server, CLI, SSR machinery) is a barrier to entry.

3. **Ecosystem mismatch.** Every major framework has embraced the plugin model: Vue has `@vitejs/plugin-vue`, Svelte has `@sveltejs/vite-plugin-svelte`, Solid has `vite-plugin-solid`. These plugins let frameworks coexist with Vite's bundling, HMR, and tooling. Chocola has no equivalent, which makes it invisible in the Vite ecosystem.

4. **Maintenance overhead.** The monolith forces all contributors to work within a single release cycle. A bug fix in the compiler shouldn't require a server release, and a CLI improvement shouldn't block a core update. The monolith also means that users who only need the compiler (e.g., for a custom build pipeline) pay the weight of the server, dev tooling, and CLI in their dependency tree.

5. **Marketing and positioning.** Chocola currently occupies a specific niche — "the sweetest way to build the web" with its own CLI, server, and SSR. But this positioning alienates developers who want to use Chocola components as a *library*, not a *framework*. A modular structure allows Chocola to be both: a component framework with its own full-stack metaframework (`@chocola/stacks`) *and* a library that works with Vite (`@chocola/vite-plugin`).

### What becomes possible

- **Vite integration.** A project using Vite can install `@chocola/core` and `@chocola/vite-plugin`, register the plugin in `vite.config.js`, and start writing Chocola components — no `chocola dev`, no `chocola build`. Vite handles bundling, HMR, and dev server; Chocola handles components.

- **Gradual adoption.** A team can start by using Chocola components inside their existing Vite project (`@chocola/core` + `@chocola/vite-plugin`). Later, if they want SSR, dev server features, or the full Chocola-native experience, they can add `@chocola/server` or migrate entirely to `@chocola/stacks`. No big-bang migration required.

- **Independent scaling.** Each package can be versioned, tested, and released independently. A breaking change in the CLI doesn't block a compiler patch. Contributors can focus on a single package without needing to understand the entire codebase.

- **Third-party plugin ecosystem.** If `@chocola/core` is cleanly separated, other tooling authors can build plugins for it — Webpack plugins, Rollup plugins, esbuild plugins — without forking the Chocola repository.

- **New branding surface.** Each package is a separate npm package with its own documentation, README, and GitHub presence. `@chocola/vite-plugin` can be discovered by Vite users searching for frameworks. `@chocola/stacks` can be marketed as "the full Chocola experience". This creates multiple entry points for different audiences.

## Detailed design

### Package structure

The proposed split into five packages:

| Package | Contents | Purpose |
|---|---|---|
| `@chocola/core` | Compiler, parser, runtime, module-graph | The heart of Chocola — compiles `.html` components to runnable JS. Framework-agnostic. |
| `@chocola/server` | Dev server, SSR server, `createHandler` | Full-stack capabilities: dev server with HMR, SSR rendering. Used by `@chocola/stacks` and advanced users. |
| `@chocola/cli` | `chocola` / `chjs` binary | Command-line tool for building, developing, and serving Chocola projects. |
| `@chocola/stacks` | Full metaframework bundling `@chocola/core` + `@chocola/server` + `@chocola/cli` | The original Chocola experience — a complete, autonomous framework with no third-party dependencies. |
| `@chocola/vite-plugin` | Vite plugin wrapping `@chocola/core` | Official Vite integration. Lets Vite handle bundling/HMR/dev server while Chocola handles components. |

### Technical Background

**Current state.** The monolithic `chocola` package (`package.json` version `2.0.0-next.11`) distributes:

- `compiler/` — The build pipeline: `module-graph.js`, `component-processor.js`, `render.js`, `pipeline.js`, `runtime-generator.js`, `config.js`, `utils.js`, `dom-processor.js`, `fs.js`, `chalk.js`, `index.js`.
- `dev/` — Dev server with HMR (`dev/index.js`).
- `server/` — SSR server (`server/index.js`).
- `runtime/` — Client-side runtime (`runtime/index.js`).
- `parser/` — HTML/component parser (`parser/component.js`).
- `bin/` — CLI entry point (`bin/chocola.js`).
- `utils.js` — Shared utilities.

All are shipped as a single npm package with sub-path exports (`chocola/compiler`, `chocola/dev`, `chocola/server`). The `bin` field exposes `chocola` and `chjs` commands.

**What changes.** Each package above becomes its own npm package with its own `package.json`, repository (or monorepo workspace), and publish pipeline. The `compiler/` directory becomes `@chocola/core`, `dev/` + `server/` become `@chocola/server`, `bin/` becomes `@chocola/cli`, and a new `vite-plugin/` directory becomes `@chocola/vite-plugin`.

**Monorepo approach.** The current repo at `github.com/chocolajs/chocola` can become a monorepo root with workspaces. Each package lives in its own directory (`packages/core`, `packages/server`, `packages/cli`, `packages/stacks`, `packages/vite-plugin`). This preserves the single-repository development workflow while enabling independent versioning and publishing.

Alternatively, each package could live in its own repository under the `chocolajs` GitHub organization (e.g., `chocolajs/core`, `chocolajs/vite-plugin`), which gives each package full independence but requires coordinated releases and cross-repo dependency management.

### Implementation

#### Package boundaries

**`@chocola/core`** — The compiler package. Contains everything in the current `compiler/`, `parser/`, and `runtime/` directories, plus `utils.js`. This is the minimal package needed to compile Chocola components. It has no dependencies on Node.js-specific APIs (no `http`, `fs.watch`, etc.) beyond the build-time file system operations. It should be usable in any environment that can run Node.js — including bundlers via their plugin APIs.

Key exports:
- `compile(rootDir)` — Build components for production.
- `buildModuleGraph(rootDir)` — Load and parse the project graph.
- Programmatic `app.build()` API (preserved from current `compiler/index.js`).

**`@chocola/server`** — The server package. Contains the current `dev/` and `server/` functionality. Provides:
- Dev server with HMR (`dev.server()`).
- SSR production server (`serve()`, `createHandler()`).
- Middleware support.

This package depends on `@chocola/core` for compilation. It is the package used by `@chocola/stacks` and by advanced users who want the full server-side experience without the CLI.

**`@chocola/cli`** — The command-line package. Contains the `bin/chocola.js` entry point and all CLI argument parsing logic. It delegates to `@chocola/core` and `@chocola/server` for the actual work. This is the package users install globally (`npm i -g @chocola/cli`) or use via `npx`.

**`@chocola/stacks`** — The metaframework package. A convenience package that depends on `@chocola/core`, `@chocola/server`, and `@chocola/cli` and re-exports them as a unified API. This is the "batteries-included" Chocola experience — the spiritual successor to the current monolithic `chocola` package. Users who want the full framework install `@chocola/stacks` and get everything. No third-party tools required; the server, dev server, and CLI are all bundled.

**`@chocola/vite-plugin`** — The Vite plugin package. Contains a Vite plugin (`chocolaVite()`) that wraps `@chocola/core`'s compilation pipeline and integrates it into Vite's build and dev cycles. It handles:
- Transforming `.html` Chocola components during Vite's module graph build.
- Providing HMR for Chocola components during `vite dev`.
- Generating the client runtime chunks during `vite build`.
- Server-side rendering hooks if `vite-plugin-ssr` or similar is used.

This package depends on `@chocola/core` and `vite` as peers.

#### How the Vite plugin works

The `@chocola/vite-plugin` translates Chocola's compilation steps into Vite's plugin hooks:

1. **`transform` hook** — When Vite encounters a `.html` file with Chocola components, the plugin invokes `@chocola/core`'s `compile()` pipeline to produce the JavaScript output (runtime chunk, component class, template).
2. **`resolveId` hook** — Maps Chocola component imports (`import Action from "./Action.html"`) to virtual module IDs that the plugin can resolve.
3. **`load` hook** — Returns the compiled output for virtual modules.
4. **`configureServer` hook** — For dev mode, the plugin integrates with Vite's dev server to provide HMR updates when components change. The plugin uses Vite's native HMR mechanism instead of Chocola's custom one.
5. **`generateBundle` hook** — During `vite build`, the plugin ensures the Chocola runtime chunks are properly emitted and bundled.

The end result: a Vite project that treats Chocola `.html` files as first-class citizens, with all of Vite's existing features (CSS bundling, JS minification, asset optimization) working alongside Chocola components.

#### Dependency graph

```
@chocola/stacks
  ├── @chocola/core
  ├── @chocola/server
  └── @chocola/cli

@chocola/vite-plugin
  ├── @chocola/core
  └── vite (peer)

@chocola/server
  └── @chocola/core

@chocola/cli
  ├── @chocola/core
  └── @chocola/server (optional)
```

## How we teach this

### For new users

The primary entry point depends on the user's stack:

- **"I use Vite"** → Install `@chocola/core` and `@chocola/vite-plugin`. Register the plugin in `vite.config.js`. Start writing components. This is the easiest path for developers already in the Vite ecosystem.
- **"I want the full Chocola experience"** → Install `@chocola/stacks`. Get everything — compiler, dev server, SSR, CLI — out of the box. This is the spiritual successor to the current `chocola` package.
- **"I need custom tooling"** → Install `@chocola/core` directly and build whatever integration you need (Webpack plugin, Rollup plugin, esbuild plugin, etc.).

### Marketing positioning

- **`@chocola/core`** is the foundation. It's the compiler. It's what makes Chocola components work anywhere.
- **`@chocola/stacks`** is the full framework. It's Chocola as you know it — autonomous, complete, no third-party dependencies. Marketing it as "The Chocola Framework" or "Chocola Stacks."
- **`@chocola/vite-plugin`** is the bridge. It brings Chocola to Vite. Marketing it as "Chocola for Vite" or "Chocola Components — Ready for Vite."
- **`@chocola/server`** is the full-stack layer. For teams that need SSR and dev server but build their own frontend integration.
- **`@chocola/cli`** is the command-line tool. For the `@chocola/stacks` experience or for programmatic use.

### Migration path

Existing Chocola projects continue to work unchanged with `@chocola/stacks`, which is essentially the old `chocola` package repackaged. The `chocola` CLI remains available via `@chocola/cli`. The `chocola.config.json` and all existing project structures are preserved. There is no migration required for current users.

New users are introduced to the package that matches their workflow — Vite users get the plugin, full-stack users get stacks, library consumers get core.

## Drawbacks

### Maintenance complexity

- **Multiple packages to maintain.** Five packages means five `package.json` files, five version numbers, five CI pipelines (or more complex monorepo CI), and five changelogs to track. A change in `@chocola/core` may require coordinated releases in `@chocola/server`, `@chocola/cli`, `@chocola/stacks`, and `@chocola/vite-plugin`.
- **Cross-package testing.** Integration tests that span multiple packages (e.g., `@chocola/vite-plugin` using `@chocola/core`) need careful setup. The monolith's single test suite becomes multiple test suites that must work together.
- **Dependency management.** Each package introduces its own dependency tree. `@chocola/core` might depend on `acorn` and `linkedom`, while `@chocola/vite-plugin` depends on `vite` and `@chocola/core`. Version conflicts between packages (e.g., different versions of the same transitive dependency) become possible and must be managed, especially in a monorepo with workspaces.
- **Breaking changes propagate.** If `@chocola/core` changes its public API, every dependent package needs to be updated and tested before release. The monolith had this problem too, but it was invisible — everything shipped together. Now it's explicit.

### Marketing tradeoffs

- **Brand fragmentation.** Five packages mean five names to market. While `@chocola/stacks` can carry the "Chocola" brand, `@chocola/vite-plugin` is a more niche product. It may be harder to build brand recognition around a plugin than around a framework.
- **Discovery challenges.** `@chocola/vite-plugin` competes with established frameworks in the Vite ecosystem. Users searching for "Vite frameworks" may not discover Chocola at all, since it appears as a plugin rather than a standalone framework.
- **Positioning confusion.** Without a clear narrative, users may not understand the relationship between the packages. "Is `@chocola/stacks` the real Chocola? Is `@chocola/core` just a library? Why would I use `@chocola/cli` instead of `npx @chocola/stacks`?" Clear documentation and marketing are essential.
- **The monolith's simplicity is a feature.** The current monolith is easy to understand: `npm install chocola`, `chocola dev`, done. The modular approach introduces a decision point ("which package do I need?") that some users may find overwhelming, especially beginners.

### Technical tradeoffs

- **Plugin maturity.** A Vite plugin that replicates all features of Chocola's native dev server (HMR, SSR, hot reload for components) is a significant engineering effort. The plugin may not match the native experience immediately, especially for advanced features like `mount:if`, `bind:*`, and server-side `ctx` evaluation.
- **Loss of control.** When Chocola's dev server is replaced by Vite's, Chocola loses direct control over the dev experience. HMR behavior, error overlays, and console output are determined by Vite, not Chocola. This could lead to a degraded developer experience in edge cases.
- **Bundle size overhead.** `@chocola/vite-plugin` adds Vite as a dependency (peer). For users who only need the compiler and don't use Vite, this is irrelevant — but for the Vite plugin path, the combined `vite` + `@chocola/core` bundle may be larger than the monolithic `chocola` package's output.
- **Double maintenance of `@chocola/stacks`.** `@chocola/stacks` is essentially a thin wrapper around the other packages. It adds little value beyond convenience, but it still needs to be maintained, versioned, and tested. It could become a "dead weight" package if users migrate to the individual packages.

## Alternatives

### Status quo — stay monolithic

Keep the current `chocola` package as-is. No split, no plugin, no additional complexity. The monolith is simple, easy to understand, and works today. The downside is continued bundler lock-in and slower adoption outside the Chocola ecosystem. This is the baseline option and requires zero additional work.

### Partial split — compiler + everything else

Instead of five packages, split into just two: `@chocola/core` (the compiler) and `@chocola/framework` (everything else). This halves the maintenance burden while still enabling Vite integration via a community or official plugin. The downside is that users who want the full Chocola experience still need to install two packages, and the "full stack" story is less clear.

### Community plugin model

Don't create an official `@chocola/vite-plugin`. Instead, publish `@chocola/core` and let the community build plugins. This reduces maintenance scope but risks a fragmented plugin ecosystem with varying quality. The Vite plugin may never reach the quality of the native dev server without dedicated maintenance.

### Monorepo with workspace aliases

Keep everything in one repo but use npm workspaces with path aliases (`"chocola": "file:packages/core"`). This preserves the monorepo development workflow while enabling independent publishing. The current repo already has a monorepo-like structure; this approach formalizes it. The downside is that users who install `chocola` still get the monolith unless they explicitly opt into individual packages.

## Unresolved questions

- **Repository structure.** Should this be a single monorepo with workspaces, or separate repositories under `chocolajs/`? A monorepo simplifies cross-package development and testing; separate repos give each package full autonomy. The current repo structure (`compiler/`, `dev/`, `server/`, `runtime/`, `parser/`) suggests a monorepo migration is natural, but the decision is not made.
- **Package naming.** T`@chocola/core`, `@chocola/server`, `@chocola/cli`, `@chocola/stacks`, `@chocola/vite-plugin` are suggested as placeholders. The actual scope of each package may change: should `@chocola/server` include both dev and SSR, or should they be separate? Should the CLI be part of `@chocola/stacks` instead of its own package?
- **Versioning strategy.** Will all packages use the same version number (lockstep) or independent semver? Lockstep simplifies compatibility but defeats the purpose of independent releases. Independent semver requires careful coordination, especially for `@chocola/stacks` which depends on the other packages.
- **Backwards compatibility guarantees.** How long will the monolithic `chocola` package continue to exist and receive updates? Will `@chocola/stacks` eventually replace it entirely, or will they coexist? A clear deprecation timeline is needed.
- **Vite plugin scope.** What exactly does the `@chocola/vite-plugin` need to support from day one? Minimum viable: compile `.html` components, HMR for components, `vite build` output. Stretch: SSR integration, middleware support, `vite preview`. The scope directly impacts the engineering effort.
- **Peer dependency strategy.** Should `@chocola/core` have any peer dependencies? Currently it depends on `acorn` and `linkedom`. Should these be bundled or peer? If peer, users must install them; if bundled, `@chocola/core`'s size increases.
- **Documentation split.** Will each package have its own documentation site, or will there be a unified `chocola.dev` with sections for each package? The current `chocolajs.dev` site would need significant restructure.
- **Testing matrix.** How will cross-package integration tests be structured? `@chocola/vite-plugin` tests need a working Vite environment + `@chocola/core`. `@chocola/stacks` tests need all packages installed. The current single test suite (`tests/**/*.test.js`) needs to be reorganized.
