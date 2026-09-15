- Start Date: 2026-09-14
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# Portable CLI with opt-in config and zero-boilerplate

## Summary

Introduce a first-class, globally-installable CLI (`chocola`, alias `chjs`) that makes Chocola **portable and zero-boilerplate** by default. The CLI ships via `package.json#bin` (`chocola` + `chjs`; `choco` rejected due to Chocolatey) and provides three commands — `chocola build`, `chocola dev` (with `--open`), and `chocola serve` — that directly replace the user-authored init scripts (`chocola.js`, `chocola.server.js`, `server.js`) described in `documentation/01-introduction/03-project-structure.md:106-152` and `documentation/01-introduction/02-getting-started.md:47-92`. Configuration (`chocola.config.json`) becomes fully **opt-in** via `getConfig(rootDir, { silent:true })` / `loadConfig(rootDir, { silent:true })`: when absent the CLI falls back silently to defaults (`srcDir: "src"`, `outDir: "dist"`, `libDir: "lib"`, `dev.port: 3000`, `server.port: 8080`, etc.). Precedence is `CLI flags > env PORT (serve) > config file > built-ins`; `serve` binds to `0.0.0.0` when `PORT` is set in containers. The CLI reuses `compiler/index.js:18-40` `logBanner()` for help and delegates to the project-local `chocola` installation when present (Vite-style). Existing programmatic APIs (`app.build(__dirname)`, `dev.server(__dirname)`, `serve(__dirname)` / `createHandler(__dirname)`) are preserved but become unnecessary for most users. `chocola init`/`create` scaffolding is out of scope and will be a follow-up implementation.

## Motivation

### What is hard today

A new Chocola project currently requires **three user-written JS entry points** that are identical across projects except for `__dirname` resolution:

```js
// chocola.js
import { app } from "chocola/compiler";
import path from "path";
import { fileURLToPath } from "url";
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
app.build(__dirname);
```

```js
// chocola.server.js
import { dev } from "chocola/dev";
import path from "path";
import { fileURLToPath } from "url";
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
dev.server(__dirname);
```

```js
// server.js
import { serve } from "chocola/server";
import path from "path";
import { fileURLToPath } from "url";
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
serve(__dirname);
```

Plus `chocola.config.json` (`documentation/01-introduction/03-project-structure.md:72-91`) which is required to silence per-field `WARNING!` messages emitted by `compiler/config.js:20-31`, `dev/index.js:30-38`, `server/index.js:398-405`, `utils.js:50-61`.

Pain points:

1. **Boilerplate / non-portable.** Every tutorial, starter, and CI pipeline must copy/paste these files. Forking a `src/` folder alone is insufficient to build; the project is not portable. Beginners stumble on `fileURLToPath`/`__dirname` ESM ceremony (`documentation/01-introduction/02-getting-started.md:47-92` calls it out as step 4/5).
2. **Global impossibility.** There is no `bin` entry in `package.json:35-43` (`exports` only exposes `./compiler`, `./dev`, `./server`). `npx chocola build`, `npm exec chocola dev`, or `npm i -g chocola` cannot work. Users cannot prototype without writing files: `mkdir tmp && npx chocola dev` should Just Work.
3. **Config is de-facto required.** `utils.js:getConfig()` treats `ENOENT` as `__chocolaMissingConfigFile` and `loadConfig()` falls back to defaults (`compiler/config.js:10-12`), but `queueConfigWarning`/`flushConfigWarnings` (`utils.js:13-38`) plus per-block warnings (`utils.js:46-61`, `compiler/config.js:20-31`) nudge users to create a config even when defaults are fine. `dev/index.js:25` and `server/index.js:394` similarly warn when blocks are missing. Zero-boilerplate means **no warnings when config is absent** in CLI mode — defaults are intentional.
4. **Script drift / duplicate logic.** `dev/index.js:serve()` and `server/index.js:serve()` each re-parse config, resolve paths via `resolvePaths()` (`compiler/config.js:43-49`), watch `srcDir` (`dev/index.js:61`), handle `hostname`/`port` precedence, and invoke `compile()`/`createHandler()`. If a user edits `chocola.config.json` they must remember to restart the right script. A unified CLI centralizes this logic.
5. **Ecosystem mismatch.** Vite (`vite build/dev`), Astro (`astro build/dev`), Next (`next build/dev/start`), SvelteKit (`vite dev/build`) all ship a single binary that works with or without a config file. Chocola requiring three bespoke `.js` files is an outlier for `ROADMAP.md:42` "Global portable CLI" future item.

Concrete use cases unlocked:

- `npx chocola@latest build` in CI without committing `chocola.js`.
- `npm i -g chocola && chocola dev my-app --port 4000 --open` for quick demos, workshops, or `create-chocola` scaffolds that ship only `src/index.html`.
- `chocola serve --port $PORT` on Render/Fly/Netlify where the start command is a single binary, no `server.js` needed.
- `npx chocola build ./examples/minimal` from a monorepo root without `chdir`.

No GitHub issue tracks this yet, but it is the explicit blocker for `documentation/01-introduction/02-getting-started.md` step reduction and for `ROADMAP.md:42` Future releases.

### What becomes possible

```
my-app/
├── src/
│   ├── lib/Button.html
│   └── index.html
└── package.json          # no chocola.js, no chocola.server.js, no server.js, no chocola.config.json required

$ chocola dev            # -> dev/index.js:serve(process.cwd()) with defaults, http://localhost:3000
$ chocola build          # -> compiler/index.js:compile(process.cwd()) -> dist/
$ chocola serve --port 8080  # -> server/index.js:serve(process.cwd(), { port: 8080 })
```

With opt-in config, an advanced project keeps full control:

```json
// chocola.config.json (optional)
{
  "bundle": { "outDir": "build", "emptyOutDir": false },
  "dev": { "port": 5173 },
  "server": { "middleware": "./middleware.js" }
}
```

```sh
chocola dev --port 4000   # flag wins over config (4000), config wins over default (5173 vs 3000)
chocola build --outDir ./tmp/build --srcDir ./src
chocola serve ./my-app --host 0.0.0.0
```

## Detailed design

### Technical Background

**Current execution model**

- `compiler/index.js:89-133` exports `compile(rootDir)` and `app.build(rootDir)`. `compile` calls `buildModuleGraph(rootDir)` (`compiler/module-graph.js:155`) → `loadConfig(rootDir)` → `resolvePaths()` → `renderPage()` → `emit()`. `utils.js:getConfig()` reads `chocola.config.json` from `rootDir`.
- `dev/index.js:12-162` exports `serve(rootDir)` and `dev.server(rootDir)`. It loads both `getConfig` and `loadConfig`, resolves `dev.hostname/port` with warnings (`dev/index.js:30-39`), calls `compile(rootDir)`, watches `paths.src` via `fs.watch(..., {recursive:true})`, and serves `dist/` with an injected HMR poller.
- `server/index.js:187-421` exports `createHandler(rootDir, opts)`, `createServer`, `serve`. `createHandler` builds the module graph, loads middleware (`loadMiddleware:85-105`), primes virtual files (`ingest:204-215`), and returns an `http` handler that per-request merges `query` + `cookies` + middleware `ctx` and calls `renderPage(graph, ctx)`. `serve` wraps `createHandler` with `http.createServer(handler).listen(port, host)`. Both load `getConfig`/`loadConfig` and honor `fullConfig.server.middleware` plus `opts.middleware`.

**Packaging**

- `package.json` is `"type":"module"` (`package.json:24`), `exports` maps to ESM entries, no `bin`. `utils.js` and `compiler/config.js` already handle `ENOENT` gracefully, so zero-config works if warnings are suppressed.

Prior art: Vite/Astro CLIs use `cac`/`commander` with subcommands, `--host`/`--port`/`--outDir` flags, `process.cwd()` default root, `loadConfig` with `c12`/`find-up`. They all support `npx <tool> dev` with no config. Chocola should mirror that DX but keep its existing `with(ctx)` rendering and deterministic hashing untouched.

### Implementation

#### Overview

Add `bin/chocola.js` (ESM, `#!/usr/bin/env node`) plus `package.json#bin` entry. Use a tiny arg parser (inline ~60 LOC or `mri`-sized, no heavy `commander` dependency) to dispatch `build|dev|serve`. Each subcommand resolves `rootDir` (positional ` [root]` default `process.cwd()`), merges **defaults < config file < CLI flags**, then delegates to the existing functions. No rendering/compiler changes.

#### 1. Packaging

```json
// package.json additions
{
  "bin": {
    "chocola": "./bin/chocola.js",
    "chjs": "./bin/chocola.js"
  },
  "files": ["bin/", "compiler/", "dev/", "server/", "runtime/", "parser/", "utils.js"]
}
```

- `bin/chocola.js` starts with `#!/usr/bin/env node` and `import { ... }` (ESM, consistent with `package.json:type:module`).
- Executable bit set (`chmod +x`) for POSIX; npm generates `.cmd`/`.ps1` shims on Windows automatically. Both `chocola` and `chjs` point to the same entry; `chjs` is the short alias (`choco` is intentionally avoided due to collision with Chocolatey on Windows).
- Keep `exports` unchanged for programmatic usage.

#### 2. CLI surface

```
Usage: chocola <command> [root] [options]

Commands:
  build   Build for production (static)
  dev     Start dev server with HMR
  serve   Start SSR production server

Options (common):
  -c, --config <path>   Path to config file (default: <root>/chocola.config.json, optional)
  -h, --help            Show help
  -v, --version         Show version (from package.json)

build options:
      --srcDir <dir>        Source directory (default: src)
      --outDir <dir>        Output directory (default: dist)
      --libDir <dir>        Components subdir inside srcDir (default: lib)
      --no-emptyOutDir      Do not clean outDir before build

dev options:
      --host <hostname>     Hostname (default: localhost, alias --hostname)
      --port <number>       Port (default: 3000)
      --open                Open browser after start

serve options:
      --host <hostname>     Hostname (default: localhost; see PORT/container note)
      --port <number>       Port (default: 8080; env PORT overrides when flag absent)
      --middleware <path>   Path to middleware file (relative to root)
```

- `root` positional: `chocola build ./my-app` → `path.resolve(process.cwd(), "./my-app")`. If omitted, `process.cwd()`. If `root` is a file path, use its dirname (helpful for `chocola build ./my-app/src/index.html` typo tolerance — warn).
- All flags are kebab-case with camelCase aliases internally (`--src-dir` → `srcDir`).
- `--help` per subcommand: `chocola build --help`, `chocola --help`.
- Exit codes: `0` success, `1` build/render error, `2` bad args. `dev`/`serve` keep running until `SIGINT`/`SIGTERM` (graceful `server.close()`).

Global vs local portability and local delegation:

- Works via `npx chocola build`, `pnpm dlx chocola dev`, `bunx chocola serve`, `npm i -g chocola` (global `bin` link), and `npm run` scripts (`"build": "chocola build"`). No project-local `chocola.js` needed. `chjs` is a drop-in alias everywhere (`chjs build`, `npx chjs dev`).
- **Local delegation (Vite-style):** When the CLI is invoked (especially from a global install), it resolves `rootDir` and attempts to delegate to the **project-local** Chocola installation first. Resolution order:
  1. Try `import.meta.resolve("chocola/compiler", pathToFileURL(path.join(rootDir, "package.json")).href)` / `createRequire(rootDir).resolve("chocola/package.json")` — if found and version satisfies, dynamically `import()` compiler/dev/server from that local path and execute with local code.
  2. Fallback to the CLI's own bundled `compiler/` if no local install exists (enables `npx chocola dev` in an empty folder) or if resolution fails.
  This guarantees `npm i -g chocola@2.1` + project-local `chocola@2.0-next.11` runs the project's pinned version, not the global one. `npx chocola@2.0` already fetches the matching version, but delegation covers the `npm i -g` case without extra config. Log a one-line `Using local chocola vX.Y.Z from <root>/node_modules` at debug level when delegation occurs.

#### 3. Config resolution (opt-in)

Precedence: **CLI flags > config file > built-ins**.

Flow per command (`bin/chocola.js` pseudocode):

```js
import path from "path";
import { readFile } from "fs/promises";
import { getConfig, isMissingConfigFile } from "../utils.js";
import { loadConfig, resolvePaths } from "../compiler/config.js";

async function resolveConfig(rootDir, cliOverrides, configPathOpt) {
  const customPath = configPathOpt ? path.resolve(rootDir, configPathOpt) : null;
  const fullConfig = customPath
    ? JSON.parse(await readFile(customPath, "utf-8"))
    : await getConfig(rootDir, { silent: true }); // <-- silent: true for CLI zero-boilerplate
  const isMissing = customPath ? false : isMissingConfigFile(fullConfig);
  const base = await loadConfig(rootDir, { silent: true, customPath }); // propagated
  // CLI overrides: e.g., --outDir, --port
  if (cliOverrides.outDir) base.outDir = cliOverrides.outDir;
  if (cliOverrides.srcDir) base.srcDir = cliOverrides.srcDir;
  if (cliOverrides.libDir) base.libDir = cliOverrides.libDir;
  if (cliOverrides.emptyOutDir != null) base.emptyOutDir = cliOverrides.emptyOutDir;
  // dev/server namespaces — PORT env handled below for serve
  const effectiveDev = { hostname: "localhost", port: 3000, ...(!isMissing && fullConfig.dev || {}), ...cliOverrides.dev };
  const effectiveServer = { hostname: "localhost", port: 8080, middleware: null, ...(!isMissing && fullConfig.server || {}), ...cliOverrides.server };
  // container PORT env override (serve only): flag > env PORT > config > default
  if (!cliOverrides.server?.port && process.env.PORT) {
    const envPort = parseInt(process.env.PORT, 10);
    if (Number.isFinite(envPort)) effectiveServer.port = envPort;
  }
  // when PORT env is set and no explicit host flag/config, default host to 0.0.0.0 for containers
  if (!cliOverrides.server?.hostname && !fullConfig.server?.hostname && !fullConfig.server?.host && process.env.PORT) {
    effectiveServer.hostname = "0.0.0.0";
  }
  return { fullConfig, base, effectiveDev, effectiveServer, isMissing };
}
```

- `utils.js:getConfig(rootDir, { silent: true })` is added by this RFC. When `silent:true`, `getConfig` does **not** enqueue the "chocola.config.json not found: using default configuration" warning (`utils.js:65-72`) and skips per-block warnings (`utils.js:46-61` `warnedBlockBundle/Dev/Server`). `loadConfig(rootDir, { silent:true })` similarly skips `compiler/config.js:20-31` `warnedBundleFields` and `dev/index.js:30-39` / `server/index.js:398-405` warnings when the caller opts in. All warnings are already gated through `queueConfigWarning`; `silent` simply prevents enqueuing in the first place (no need to clear buffers after the fact). Direct `getConfig`/`loadConfig` calls outside the CLI (programmatic `app.build`) keep current behavior (`silent` defaults to `false`) so existing projects still see warnings.
- If `chocola.config.json` **exists** but a block is missing, `silent` does not suppress — the file was found, so missing-block warnings still fire (useful for advanced users; they already opted into config). `isMissingConfigFile` remains the signal.
- `--config` allows monorepo or `chocola build --config ./configs/chocola.prod.json`. When `--config` is explicit and the file is missing, throw `ENOENT` (do not silence) so typos are surfaced.
- Precedence final: **CLI flags > env PORT (serve) > config file > built-ins**; `dev` does not read `PORT` (only `serve` does, to avoid surprising local dev).

#### 4. Command delegation

```js
// bin/chocola.js dispatch
const [cmd, rootArg, ...rest] = parse(process.argv);
const rootDir = path.resolve(rootArg || process.cwd());
switch (cmd) {
  case "build": {
    const { base } = await resolveConfig(rootDir, cli);
    // reuse compiler/index.js internals but with overridden config
    // Option A: call existing `compile(rootDir)` after temporarily patching getConfig/loadConfig via cliOverrides
    // Option B: call lower-level `buildModuleGraph` + `emit` with explicit config (preferred — no monkey-patch)
    await compile(rootDir, { overrides: cli }); // or new entry compileWithOverrides
    break;
  }
  case "dev": {
    const { effectiveDev } = await resolveConfig(rootDir, cli);
    await devServe(rootDir, { port: effectiveDev.port, hostname: effectiveDev.hostname, overrides: cli });
    break;
  }
  case "serve": {
    const { effectiveServer } = await resolveConfig(rootDir, cli);
    await serverServe(rootDir, { port: effectiveServer.port, hostname: effectiveServer.hostname, middleware: effectiveServer.middleware });
    break;
  }
}
```

Concrete mapping:

- `build`: calls `compiler/index.js:compile(rootDir)` — or `emit(await buildModuleGraph(rootDir, overrides))`. To avoid duplicating config logic, add an optional `overrides` param to `loadConfig(rootDir, { silent:true, overrides })` and thread through `buildModuleGraph`/`renderPage`. Alternatively, the CLI writes a temporary in-memory config overlay and calls existing `compile` (simplest: CLI sets `process.env.CHOCOLA_CLI_OVERRIDES` and `loadConfig` reads it — but explicit param is cleaner). Initial PR can implement by **passing CLI values directly to `loadConfig` merge step** without touching compiler internals for `dev`/`server` port overrides (those are already `opts` in `server/index.js:388-393` which respects `opts.port/hostname` over config).
- `dev`: calls `dev/index.js:serve(rootDir, { port, hostname, open })` — extend to accept `open` boolean. When `--open` is set, after `server.listen` succeeds the CLI does a dynamic `import("open")` or `await import("node:child_process")` fallback to `open http://hostname:port` (lazy import so `open` is optional; if not installed, use `child_process.exec` platform branching `open`/`xdg-open`/`start`). `dev/index.js:12-42` currently loads `fullConfig` internally; add `opts` param that short-circuits config file values. `fs.watch` (`dev/index.js:61`) and HMR injection (`dev/index.js:129-148`) unchanged.
- `serve`: calls `server/index.js:serve(rootDir, { port, hostname, middleware })` — already supports `optsArg.port/hostname/middleware` (`server/index.js:388-394`, `187-197`). CLI normalizes kebab flags to this shape and applies PORT/host container logic above before calling `serve`. `process.env.PORT` is honored only when no `--port` flag is present. For `createHandler` programmatic use, no change.

Signal handling: `dev` and `serve` trap `SIGINT`/`SIGTERM` to `server.close()` and `fs.watch` close, then `process.exit(0)`.

Local delegation detail (`bin/chocola.js` preamble, before arg parsing):

```js
import { createRequire } from "module";
import { pathToFileURL } from "url";
async function delegateToLocal(rootDir) {
  try {
    const req = createRequire(pathToFileURL(path.join(rootDir, "package.json")).href);
    const localPkg = req.resolve("chocola/package.json");
    const localRoot = path.dirname(localPkg);
    // Avoid infinite loop when local === global (same path)
    if (localRoot === path.dirname(path.dirname(import.meta.url))) return null;
    const localBin = path.join(localRoot, "bin/chocola.js");
    // re-exec with local bin preserving argv
    // Option: import local modules directly instead of re-exec:
    const mod = await import(pathToFileURL(path.join(localRoot, "compiler/index.js")).href);
    return mod; // caller uses local compile/dev/server
  } catch { return null; }
}
```

If delegation succeeds, the global CLI forwards to local `compiler/index.js`, `dev/index.js`, `server/index.js` implementations, ensuring config parsing and rendering match the project's declared dependency.

#### 5. Backwards compatibility

- Keep `compiler/index.js:app.build`, `dev/index.js:dev.server`, `server/index.js:serve/createHandler` exports unchanged.
- If `chocola.js`/`chocola.server.js`/`server.js` exist alongside CLI use, they remain runnable (`node chocola.js` still works). CLI does **not** require or execute them. If both exist, CLI takes precedence when user runs `chocola build`; the old files are ignored but not deleted. Document deprecation (soft): mention in `CHANGELOG.md` that init scripts are "legacy; prefer `chocola build/dev/serve`".
- No breaking change to `chocola.config.json` schema; all existing keys honored.
- Projects that already have `chocola.js` can incrementally adopt CLI by removing the file and relying on defaults — no code changes.

#### 6. Arg parsing & DX

Avoid adding `commander`/`yargs` (adds ~50-100kB). Implement ~80 LOC parser handling:

- `--help`, `--version` (`--version` reads `package.json:version`).
- `--no-emptyOutDir` boolean negation.
- `--port`/`--host` coercion (`parseInt`, validate 1-65535).
- `--open` boolean (dev only) — opens `http://<host>:<port>` after listen; implemented via lazy `import("open")` with `child_process` fallback (no hard dep).
- Unknown flag → `console.error` + `process.exit(2)` with hint.
- Color via existing `compiler/chalk.js`.

Help output **reuses** `compiler/index.js:18-40` banner (`logBanner()` gold/white `C H O C O L A` box + `THE SWEETEST WAY TO BUILD THE WEB`). `chocola --help` and `chocola <cmd> --help` first print the banner, then the command-specific usage. This keeps brand identity across `compile()` logs and CLI help. For pipe/grep friendliness, `chocola --help --plain` or `NO_COLOR=1` can suppress ANSI, but default is banner + color (consistent with `logBanner`/`logSuccess`).

#### 7. Testing & validation

- Unit: `tests/cli.test.js` — invoke `bin/chocola.js` via `child_process.spawn` with fixtures `tests/fixtures/basic` and `tests/fixtures/server`. Assert `chocola build` creates `dist/index.html` without `chocola.js` present; `chocola build --outDir tmp` respects flag; `chocola --help`/`chocola build --help` exit 0; `--version` matches `package.json`.
- Integration: run `npx --yes ./bin/chocola.js dev --port 0` and assert `Live server running at` log.
- Config precedence test: `chocola build --config ./alt.json` picks alt; missing config produces no `WARNING!` lines (`configWarningBuffers` cleared).
- Existing `tests/compiler.test.js`, `tests/config.test.js`, `tests/server.test.js` must pass unchanged (no compiler regression).
- Bench: no runtime/perf impact; CLI is Node-only wrapper.

#### 8. Docs & packaging checklist

- `package.json#bin`, `files` include, `npm pack` dry-run shows `bin/chocola.js`.
- `README.md` quickstart updates: replace steps 4/5 with `npx chocola dev` / `chocola build`.
- `documentation/01-introduction/03-project-structure.md` rewritten to list CLI as primary, init scripts as "Legacy (optional) — replaced by `chocola` CLI".
- `documentation/01-introduction/02-getting-started.md` steps 4-5 replaced with `chocola dev`/`chocola build`/`chocola serve`, keep `createHandler` example as programmatic escape hatch.
- `CONTRIBUTING.md:74-75` `npm pack` instruction extended to test CLI (`node ./bin/chocola.js --help`).
- `ROADMAP.md:42` checked off.

## How we teach this

- **Primary path (zero-boilerplate):** New user flow becomes `npm create chocola` (future) or manual `mkdir my-app && npm init -y && npm i chocola && mkdir -p src/lib src/static && echo '<html><body><app>Hello</app></body></html>' > src/index.html && npx chocola dev`. No `chocola.js` mentioned. Teaching emphasizes `chocola <command>` is the only entry point.
- **Opt-in config:** `chocola.config.json` introduced as "only if you need to change defaults" (`bundle.outDir`, `dev.port`, `server.middleware`). `compiler/config.js:43-49` defaults table stays canonical; docs add "CLI flags override this file; if the file is missing, defaults are used silently".
- **Migration for existing projects:** A callout box "Migrating from init scripts" shows `rm chocola.js chocola.server.js && npx chocola build` produces identical `dist/` and `.chocola/hashes.json`. Keep one paragraph that `app.build(__dirname)` etc. still work for embedding Chocola in custom Node tooling (e.g., Vite plugin, Electron).
- **Reference:** New page `documentation/07-reference/04-cli.md` (or `01-cli.md`) documents all commands/flags/exit codes, mirroring `documentation/06-architecture/01-compiler-flow.md` but for operators/CI. Existing `documentation/06-architecture/02-ssr-server.md` cross-links `chocola serve` vs `createHandler`.
- No new terminology beyond `root` (project root directory); aligns with Vite/Next.

## Drawbacks

- **Maintenance surface.** A CLI must be kept in sync with `compiler/config.js`, `dev/index.js`, `server/index.js` flag sets. Adding a new config key requires updating the CLI parser and help text — mitigated by sharing `loadConfig` defaults and the single `resolveConfig` helper.
- **Bin name collision.** `chocola` is owned by this project (`npm view chocola`), but short names risk collision. Mitigated by shipping `chjs` as the **only** alias; `choco` is rejected due to Chocolatey on Windows. Both bins point to the same ESM entry so `chjs --help` is identical to `chocola --help`.
- **Global vs local version skew — mitigated by delegation.** Without delegation, `npm i -g chocola@2.1` + project-local `chocola@2.0-next.11` would run the global version. This RFC **implements Vite-style delegation** (resolve local `chocola/package.json` from `rootDir` and import local `compiler/dev/server` when found), so the skew is resolved by design. Fallback to global when no local install exists preserves `npx chocola dev` in empty folders. The extra `createRequire`/`import` adds ~10 LOC.
- **Silent config absence.** `getConfig(rootDir, { silent:true })` suppresses warnings only when invoked via the CLI; programmatic `app.build` still warns. Bare missing file is silent (desired for zero-boilerplate), but `--config <path>` with `ENOENT` still throws so typos surface. A mis-placed `chocola.config.json` in a parent dir remains undetected (same as today — `getConfig` only looks at `rootDir`, no `find-up`).
- **Windows ESM bin.** `#!/usr/bin/env node` + `package.json:type:module` requires Node >=16 and `.js` extension; Windows `.cmd` shim generation depends on `npm`/`pnpm` correctly handling ESM bins. Tested path: keep bin as `bin/chocola.js` (not extensionless) — npm handles it for both `chocola` and `chjs`.
- **Flag parity creep.** `--open` is now included (lazy `open` import, no hard dep), and `--host 0.0.0.0` container default is handled via `PORT` env for `serve`. Further flags (`--https`, `--watch`) remain deferred to keep scope bounded.

## Alternatives

- **Status quo — keep init scripts.** Users continue to copy `chocola.js`/`chocola.server.js`/`server.js`. `package.json` stays without `bin`. Impact: no portability, no global use, tutorials remain verbose, `ROADMAP.md:42` stays unchecked. This is the "do nothing" baseline.
- **NPM scripts only.** Add `"scripts": {"build":"node chocola.js","dev":"node chocola.server.js","start":"node server.js"}` and tell users to run `npm run build`. Still requires the three files; not portable across projects without copying them; `npx` prototyping still impossible.
- **Scaffolder-only (`create-chocola`).** Ship `npm create chocola` that generates `chocola.js` etc. once. Solves initial creation but not portability (every project still carries boilerplate, updating Chocola requires re-generating files). CLI + opt-in config solves scaffolding *and* steady-state.
- **Config-file-only (no flags).** Provide `chocola build/dev/serve` but require `chocola.config.json` for any customization, no CLI flags. Simpler parser but forces a file for `--port`/`--outDir` overrides — poor for CI/`$PORT` env vars. Precedence model (flags > config) is standard and low-cost, so rejected.
- **Heavy framework (`commander`/`yargs`/`cac`).** Use `commander` for parsing. More features, bigger install, another dep to audit. Rejected for initial version; inline parser + `mri` if needed keeps `package.json:dependencies` (`acorn`, `linkedom`) unbloated. Can adopt `cac` later if subcommand set grows.
- **Cookiecutter / project template repo.** Host a `chocolajs/template` repo users clone. Still boilerplate, not zero-config, and diverges from `npx`-first DX.

## Unresolved questions

- Should `--open` respect a `--host`/`--port` that came from `PORT` env delegation (e.g., `PORT=8080 chjs dev --open` should open `http://localhost:8080` or `http://0.0.0.0:8080`)? Proposal: open `http://localhost:<port>` when host is `0.0.0.0` for browser usability, otherwise `http://<host>:<port>`.
- Add `--https` for `dev`/`serve` in a follow-up, or defer until a real cert story exists? No action for this RFC.

### Resolved (per review)

- `chocola init` / `chocola create` scaffolding (templates, `--template` options, `src/index.html` generation, `git init`, package manager selection) is **out of scope** for this RFC. It will be a new feature with a full RFC covering template design and init options. For this RFC, `chocola dev/build/serve` in an empty directory errors with `src/index.html not found` and hints `mkdir -p src/lib src/static && echo '<html><body><app>Hello</app></body></html>' > src/index.html`.
- **Alias:** `chjs` only; `choco` rejected (Chocolatey collision). Both bins map to `./bin/chocola.js`.
- **`--open`:** Supported in `chocola dev` (lazy `open` import, `child_process` fallback, no hard dependency).
- **`init`/`create`:** Out of scope — separate RFC for templates/init options.
- **Silent warnings:** Implemented as `getConfig(rootDir, { silent: true })` / `loadConfig(rootDir, { silent:true })`; CLI passes `silent:true`, programmatic API defaults `false`.
- **Container `PORT`/`host`:** `chocola serve` honors `process.env.PORT` when no `--port` flag is present (precedence `flag > env PORT > config > default`). When `PORT` is set and no explicit host flag/config exists, host defaults to `0.0.0.0` for container binding.
- **Local delegation:** Global CLI delegates to project-local `chocola` via `createRequire(rootDir).resolve("chocola/package.json")` + dynamic import; falls back to bundled code when no local install.
- **Banner:** Help reuses `compiler/index.js:18-40` `logBanner()` (gold/white box). Plain output available via `NO_COLOR=1` / `--plain`.
