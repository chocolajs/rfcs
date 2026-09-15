- Start Date: 2026-09-06
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# State management

## Summary

Introduce declarative reactivity to Chocola via a `chocola/state` sub-module exposing three primitives — `$cast` (with `.$with()` config), `$react`, and `$bake` — and a template convention that distinguishes reactive (`${foo}`) from static (`{foo}`) bindings. `$cast(...).$with({ recursive, root })` allows fine-grained opt-out: `recursive` controls deep vs shallow nesting (shallow requires manual nested `$cast`), `root` controls whether structural root mutations (adding/removing leafs/items) trigger effects. `$react` is offered in two forms: an imported ` $react([dep1, dep2], () => {})` for explicit multi-dependency effects and a direct instance method `castVar.$react((oldValue) => {})` that persists as the more direct, lower-overhead path for single-variable effects (reduced scope, no array/diff). Elements and components remain statefulness-agnostic by default, with `$cast`/`$bake` directives to override binding statefulness per-tag (inspired by Flutter's stateful/stateless widgets). This RFC integrates `specs/state.md` as the canonical spec and expands it into a full RFC with compiler and teaching implications.

## Motivation

### What is hard today

Managing state in Chocola is hand-made and can break easily. For example, if we want to create a counter, we have to manually update the DOM:

```html
<script>
  let btn;
  let numDisplay;

  let num = 0;

  function $runtime() {
    btn.addEventListener("click", () => {
      num++;
      numDisplay.textContent = num;
    })
  }
</script>

<template>
  <button bind:self="btn"></button>
  <span bind:self="numDisplay"></span>
</template>
```

This puts a lot of heavy thinking on the developer for complex logic and is hard to maintain. Every piece of synchronized UI requires imperative `querySelector` / `bind:self` + manual `textContent` assignment. There is no single source of truth, no declarative dependency tracking, and no compiler help to warn about unused reactive state.

### What becomes possible

```html
<script>
  import { $cast } from 'chocola/state';

  let num = $cast(0);

  function sumNum() {
    num++;
  }
</script>

<template>
  <button on:click={sumNum}>Click me!</button>
  <span>${num}</span>
</template>
```

The compiler knows `num` is reactive (via `$cast`), the template opts into reactivity with `${num}`, and the DOM updates automatically. Snapshotting and side-effects become explicit via `$bake` and `$react` without imperative DOM manipulation. This is a prerequisite for production adoption and for the server-resolved `$runtime` tree-shaking (`text/0001-server-resolved-script-runtime.md`).

## Detailed design

### Technical Background

Reactivity lets developers bind values through their components to make them update synchronously without having to do it manually, making it safe and easy.

Most frameworks do it by default. E.g., Svelte uses runes like `$state` and `$derived` to manage reactivity. Chocola's approach is intentionally **declarative and explicit**: the compiler must know what is reactive and what has to update according to it. This enables:

- Runtime optimization (only reactive bindings subscribe).
- Better DX via compiler diagnostics (unused reactive variables, reactive casting of static variables).
- Clear migration path for SSR/CSR and server-resolved `$runtime` (`compiler/component-processor.js:538-565`, `compiler/runtime-generator.js:18`).

The state lifecycle in Chocola is designed to make track of unused reactive variables and reactive casting of static variables easier by explicitly marking reactivity at definition (`$cast`) and usage (`${}`) sites.

Prior art: Svelte runes (`$state`/`$derived`/`$effect`), Solid signals, Vue `ref`/`reactive`. Chocola's `$cast`/`$react`/`$bake` mirrors the cast-react-bake mental model (batter → mold → baked cake) and Flutter's stateful/stateless widget distinction for template overrides. With `.$with({ recursive, root })` and manual `$cast`/`$bake` of nested leaves, the model also intentionally resembles Rust: explicit, shallow-by-default opt-in ownership where deep cloning/tracking is not implicit — the developer decides per-structure whether nesting and root structural mutations are tracked, trading ergonomics for precise compiler reasoning and minimal proxy overhead.

### Implementation

I propose Chocola provides `$cast`, `$react` and `$bake` as its main way to manage state via the `chocola/state` sub-module. The following is the integrated spec (from `specs/state.md`).

#### `state` Sub-module

##### `$cast()`

When writing the `<script>` of a component, variables can be defined as stateful using the `$cast` function:

```js
import { $cast } from 'chocola/state';

let num = $cast(0);                  // primitive
let user = $cast({ name: 'Juan' });  // object (deep reactive)
let list = $cast([1, 2, 3]);         // array (deep reactive)
```

Casting a variable prompts Chocola to create a structured class that contains its value. By default the primitive is **deep reactive** for objects and arrays — mutations to nested properties trigger updates. The compiler desugars `$cast(initial)` into a reactive primitive (structured class) that holds the value and tracks subscribers. In the client `run-*.js` (`compiler/component-processor.js:495-572`, `compiler/runtime-generator.js:3-23`), the declaration is emitted as `let num = ctx.num ?? $cast(0)` when client-reachable (`text/0001-server-resolved-script-runtime.md` reachability), otherwise evaluated server-side only during `renderPage` (`compiler/render.js:152-158`, `component-processor.js:300-316`).

`$cast` is only valid at top-level `let` in `<script>` (similar to `export let` props `parser/component.js:1-10`). Reassigning the binding (`num = 5`) and mutating deep state (`user.name = 'Ana'`, `list.push(4)`) both trigger reactivity.

For arrays and objects, the casting behavior can be configured by chaining `.$with(config)` directly after `$cast()`. The config object currently exposes `recursive: boolean` (default `true`) and `root: boolean` (default `true`) for root-mutation tracking, extensible with `...` future options:

```js
import { $cast } from 'chocola/state';

// deep reactive (default) — nested mutations trigger
let user = $cast({ name: 'Juan', address: { city: 'Madrid' } });
user.address.city = 'Barcelona'; // triggers

// shallow — allow triggers, leafs must be manually casted
let shallowUser = $cast({ id: 1, name: $cast('Gabi') })
  .$with({ recursive: false });

shallowUser.id = 2; // does NOT trigger (reassigning an uncasted leaf)

shallowUser.name = 'Ana'; // triggers (reassigning a casted leaf)

shallowUser.name = $cast('Ana'); // throws compile error: can't assing $cast() to an already casted leaf

shallowUser.id = $cast(1) // casts an uncasted leaf

shallowUser.id.$bake(); // uncasts a casted leaf

// arrays
let list = $cast([1, 2, 3]); // deep — push triggers

let shallowList = $cast([ $cast(1), 2, 3 ])
  .$with({ recursive: false });
shallowList.push(4); // triggers (shallowList itself), but shallowList[1] = 99 without $cast leaf would be shallow-ignored for deep subscribers

// root-mutation control — if root is false, structural root changes don't trigger
let noRootUser = $cast({ id: 1, name: $cast('Gabi'), age: 30 })
  .$with({ recursive: false, root: false });

noRootUser.id = 99; // mutating existing uncasted leaf — NOT trigger (still shallow)
noRootUser.name = 'Ana'; // mutating casted leaf — triggers via leaf's own subscription, even though root is disabled
noRootUser.extra = 'new'; // adding a new leaf at root — does NOT trigger (root: false)
delete noRootUser.age; // removing a leaf at root — does NOT trigger
noRootUser = { id: 2, name: $cast('Leo') }; // reassigning entire root binding — still triggers (binding assignment is separate from root mutation trap)

let noRootList = $cast([ $cast(1), 2 ])
  .$with({ recursive: false, root: false });
noRootList.push(3); // does NOT trigger (root structural mutation suppressed)
noRootList.pop(); // does NOT trigger
noRootList[0] = $cast(99); // reassigning a casted indexed leaf — triggers via leaf
noRootList[1] = 42; // reassigning uncasted indexed leaf — NOT trigger
```

Semantics of `.$with()`:

- Must be chained immediately after `$cast(initial)` at the same top-level `let` declaration: `let x = $cast(...).$with({ recursive: false })` or `let x = $cast(...).$with({ recursive: false, root: false })`. Chaining after any other expression or reassigning is a compile error.
- `recursive: true` (default if `.$with()` omitted) — Chocola recursively wraps all nested objects/arrays with reactive proxies (deep reactive). Equivalent to `.$with({ recursive: true })`.
- `recursive: false` — Chocola creates a shallow primitive that tracks only the root value/reference. Nested objects/arrays remain plain JS unless they are themselves individually `$cast`-ed at creation (e.g., `name: $cast('Gabi')`). This reduces proxy depth, memory, and subscription scope for large or static-heavy structures.
- `root: true` (default) — structural mutations at the root (adding/removing/reordering leafs for objects, `push`/`pop`/`splice`/`shift`/`unshift` for arrays, `delete` for objects) notify the root's subscriber list and thus trigger `${root}`, `$react([root], ...)` and `root.$react(...)`. 
- `root: false` — structural root mutations are **suppressed** and do **not** trigger root effects. Adding `x.extra = 'hi'`, deleting `delete x.age`, or `arr.push(1)` on a `.$with({ recursive: false, root: false })` target is silent at the root level. Leaf-level reactivity remains intact: mutating a nested value that is itself individually `$cast`-ed (e.g., `name: $cast('Gabi')`) still triggers via that leaf's own subscription, even when `root: false`. Reassigning the entire binding (`x = $cast({...})`) is a binding-level write and still triggers irrespective of `root`.
- Combination: `recursive` and `root` are orthogonal. Typical presets: `{ recursive: true, root: true }` (default deep), `{ recursive: false, root: true }` (shallow but root structural changes still notify), `{ recursive: false, root: false }` (fully isolated leaves — only `$cast`-ed leaf writes trigger). `root: false` without `recursive: false` is allowed but less useful (deep leaves would still trigger via deep proxy; only root add/remove suppressed).
- The config object is compile-time only; it is not retained at runtime beyond selecting the primitive factory (`deepCast` vs `shallowCast` vs `shallowNoRoot`). Future keys (e.g., `compare: 'reference' | 'deep'`, `freeze: boolean`) are reserved via `...` extensibility but out of scope for this RFC.
- `.$with()` is only valid for object/array initializers; using it on `let num = $cast(0).$with({ recursive: false })` or `let num = $cast(0).$with({ root: false })` is a compile warning/no-op (primitives are already leaf values and have no structural root).

##### `$react()`

`$react` is available in two complementary forms — an imported effect for explicit multi-dependency tracking, and a direct instance method on a casted variable for the single-dependency fast path (reduced scope, better performance).

**1. Imported form — explicit dependencies array:**

```js
import { $cast, $react } from 'chocola/state';

let dep1 = $cast(0);
let dep2 = $cast(1);

$react([dep1, dep2], (oldValues) => {
  console.log(dep1, oldValues.dep2); // runs when dep1 or dep2 updates
});

// single dependency via imported form is also valid
let num = $cast(0);
$react([num], () => {
  console.log(num); // runs when num updates
});
```

**2. Instance method — single-variable fast path:**

```js
import { $cast } from 'chocola/state';

let num = $cast(0);

num.$react((oldValue) => {
  console.log(oldValue); // runs when num updates, receives previous snapshot
});
```

Semantics:

- **Imported signature**: `$react(deps: Array<$cast>, effect: (oldValues: {...depsValues}) => void)`. `deps` is an explicit array of casted variables to track; `effect` is an arrow function that receives a generated map of the dependencies previous snapshots.
- **Instance signature**: `castVar.$react(effect: (oldValue) => void)`. Available only on casted variables; `oldValue` is the snapshot before mutation (obtained via internal `$bake`). More direct and more performant for single-var effects: no array allocation, no dependency diffing, subscribes directly to that primitive's subscriber list with reduced scope.
- Both forms are invoked synchronously after any dependency mutates, batched per microtask if multiple mutations occur in the same tick (to avoid duplicate DOM patches and effects firing twice when both `dep1` and `dep2` change together).
- Dependencies / receivers must be casted variables; passing an uncasted variable is a compile error. `$react([])` with an empty array is a no-op with a compiler warning.
- The two forms coexist and are interchangeable for the single-var case: `num.$react(fn)` is equivalent to `$react([num], fn)` with lower overhead. Prefer the instance method when reacting to exactly one variable, and the imported form when reacting to two or more.
- In SSR, `$react` effects do not run server-side (they are client-only effects, registered inside `ChocolaComponent#init` `runtime/index.js:214-251`).

##### `$bake()`

To save a snapshot of a casted variable, use the `$bake()` function:

```js
import { $cast, $bake, $react } from 'chocola/state';

let num = $cast(0);
let hist = [];

$react([num], () => {
    hist.push($bake(num)); // push the current value of num
    // `hist.push(num);` would push the stateful variable reference instead
    if (hist.length > 5) console.log(hist);
});
```

`$bake(value)` returns a plain JS primitive/value (deep clone for objects/arrays) that is no longer reactive. It is the explicit opt-out from reactivity at the expression level. Server-side, `$bake` is identity (already plain `ctx` value via `compileExpr`).

The same effect can be written with the instance fast path when only one variable is tracked:

```js
import { $cast, $bake } from 'chocola/state';

let num = $cast(0);
let hist = [];

num.$react(() => {
    hist.push($bake(num)); // equivalent to $react([num], () => hist.push($bake(num)))
});
```

#### Templates

##### Reactive bindings `${foo}`

To work with stateful variables in templates, they must be bound as `${foo}` instead of `{foo}`. This way, the Chocola compiler will be certain what's reactive and what's not.

```html
<script>
  import { $cast } from 'chocola/state';

  let num = $cast(0);

  function sumNum() {
    num++;
  }
</script>

<template>
  <button on:click={sumNum}>Click me!</button>

  <!-- updates with num -->
  <span>${num}<span>

  <!-- will not update -->
  <span>{num}<span>
</template>
```

Compiler handling (`compiler/component-processor.js:452-464` `compileExpr`, `dom-processor.js` `interpolateNode`):

- `{expr}` — evaluated once server-side via `compileExpr(expr)(ctxProxy)` with `with(ctx)` proxy (`parser/utils.js:5-16`), result interpolated as static text/attribute.
- `${expr}` — compiled to a reactive subscription. The parser (`protectCurlyBraces` `utils.js:20-44` must distinguish `${` from `{`) creates a binding that subscribes to identifiers found in `expr`. At runtime, the generated `run-*.js` patch function re-evaluates `expr` and updates the text node/attribute when any `$cast` dependency notifies.

This explicit distinction enables the server-resolved Script Resolver (`text/0001-server-resolved-script-runtime.md` §B Reachability) to treat `${}` deps, `$react([...deps], ...)` deps, and `castVar.$react(...)` receivers as client-reachable and warn when `${num}` references a non-`$cast` variable (reactive casting of static) or when `$cast` variable is never used reactively (unused reactive).

##### Elements and components statefulness

HTML elements and Chocola components are stateful-related agnostic. You can use both `${foo}` and `{foo}` in the same tag.

Chocola provides `$bake` and `$cast` directives to make all bindings inside a tag (be it a component or an element) override its statefulness. If you're familiar with Flutter, think of it as stateful and stateless widgets.

```html
<script>
import { $cast, $bake, $react } from 'chocola/state';

let num = $cast(0);
let history = [];

function sumNum() {
  num++;
  const numSnap = $bake(num);
  history.push(numSnap);
}

$react([num], () => {
  if (num > 5) console.log(history);
});
</script>

<template>
  <button on:click={sumNum}>Click me</button>

  <!-- makes all bindings stateful -->
  <!-- overrides: <span>${num}</span> -->
  <span $cast>{num}</span>

  <span>Original value:</span>

  <!-- removes statefulness from all bindings -->
  <!-- overrides: <span>0</span> -->
  <span $bake>${num}</span>
</template>
```

Semantics:

- `<tag $cast>` — all `{expr}` inside become reactive as if written `${expr}`. Useful for bulk stateful sections without rewriting each binding.
- `<tag $bake>` — all `${expr}` inside become static as if written `{expr}`. Captures a one-time snapshot at render.
- Directives are compile-time only; they do not emit DOM attributes. They compose per-element (not inherited by children unless explicitly propagated).
- Precedence: directive overrides sigil. `$cast` on a tag containing `${num}` is redundant but valid; `$bake` on `${num}` forces static.

#### Mental Model

The analogy is simple:

- `$cast`: When you want to make a cake, you pour the batter into a mold to cast its shape. It's still malleable, and the different toppings you add remain separate (the batter with its toppings represents the generated Chocola primitive).
- `$react`: While the cake is baking, you keep an eye on it to make sure everything is going well. If something happens, you react accordingly, if necessary.
- `$bake`: When you bake the batter, this mixture (the Chocola primitive) turns into a single solid cake (the JS primitive) that is no longer malleable.

#### Compiler & Runtime Integration

- **Parsing**: `parser/component.js:50-144` `extractTopLevelVariables` and `parser/script.js` (proposed in `text/0001-`) must recognize `import { $cast, $bake, $react } from 'chocola/state'`, `let x = $cast(...)` and `let x = $cast(...).$with({ recursive: boolean, root: boolean, ... })` patterns via AST (`acorn`) rather than regex, to handle destructuring, chained `.$with()`, and deep/shallow/root reactivity. The parser validates that `.$with()` is chained immediately after `$cast()` at top-level `let` and that its argument is a static object literal with known keys `recursive`/`root`.
- **Reachability**: `$cast` declarations referenced only via `{}` are server-only; those referenced via `${}`, via `$react([...deps], fn)` dependency array, via `castVar.$react(fn)` receiver, or via `$bake` inside `$runtime` are client-reachable and emitted as `let x = ctx.x ?? $cast(init)` (or `let x = ctx.x ?? $cast(init).$with(config)` when shallow/no-root) with hydration guard (`component-processor.js:547-549`). The resolver collects identifiers from the `$react` dependency array AST and from `castVar.$react` call-site receivers (not from effect body alone). For `recursive: false`, reachability is not transitive into uncasted nested leaves — only explicitly `$cast`-ed leaves are considered client-reachable. For `root: false`, root structural ops (`push`/`pop`/`splice`/`delete`/property add) are not considered root-reachable; only leaf-level `$cast` writes count.
- **Reactivity runtime**: `runtime/index.js:173-312` `ChocolaComponent` will hold a reactive primitive class (similar to Svelte's `State` store) with `#value`, `get`, `set`, `subscribe`, `$react` (instance method for single-var fast path), `$with` (config selector), and `$bake` (deep clone). `$cast(init)` defaults to `deepCast` (recursive proxy, `root: true`); `$cast(init).$with({ recursive: false })` selects `shallowCast` (root-only proxy, nested plain unless individually `$cast`-ed); `$cast(init).$with({ recursive: false, root: false })` selects `shallowNoRoot` (neither root structural mutations nor uncasted leaves trigger; only `$cast`-ed leaves notify). The proxy `set` trap short-circuits when `root: false` and operation is `add`/`delete` at root depth; array mutators (`push`/`pop`/`splice`/`shift`/`unshift`/`sort`) are wrapped to no-op-notify when `root: false`. The standalone `$react(deps, fn)` helper iterates `deps` and subscribes to each primitive's subscriber list with batched invocation; the instance method `castVar.$react(fn)` subscribes directly with reduced scope and avoids array/diff overhead. Text node patching reuses `interpolateNode` subscriptions and respects shallow vs deep vs root-disabled scope.
- **Diagnostics**: `warnUnusedDeclaration` (`compiler/utils.js:17-22`, `component-processor.js:170-233`) extended: warn on `$cast` never used in `${}` or as a `$react` dependency / `.$react` receiver, warn on `$react([])` with empty/non-`$cast` deps and on `nonCastVar.$react`, warn on `${nonCastVar}`, and warn on `.$with()` misuses (non-object/array target, non-literal config, chained not after `$cast`, unknown config keys, contradictory `{ recursive: true, root: false }`, or `recursive: false`/`root: false` where root/leaf mutation is observed but suppressed and `${}`/`$react` would never fire).

## How we teach this

This continues the use of the `$` sigil for Chocola features and lends state management terminology from Flutter for stateful and stateless elements/components and variables.

This would imply creating a new docs section for explaining how reactivity works in Chocola. Most users will find this familiar since it's a concept present in most commercial frameworks. Specifically:

- New page `documentation/04-state/01-reactivity.md` covering `$cast`/`$cast(...).$with({ recursive, root })` (deep vs shallow vs root-disabled), `$react`/`$bake`, `${}` vs `{}`, and `$cast`/`$bake` directives with the cake analogy, including a matrix table for `{ recursive, root }` combinations.
- `documentation/02-components/01-fundamentals.md:95-140` updated to note top-level `let` is static unless casted; `documentation/03-templates` updated to document `${}`.
- `documentation/05-runtime/01-runtime.md` clarified: `$runtime` is where `$react` subscriptions are registered — both imported `$react([...deps], effect)` and instance `castVar.$react(effect)` (single-var fast path); `documentation/06-architecture/01-compiler-flow.md` steps 3-5 note reactive binding subscription injection.
- Teaching emphasizes explicitness: "if you want reactivity, cast it and bind with `${}`; otherwise it is static and server-evaluated."

No `PULL_REQUEST_TEMPLATE.md` churn beyond changelog entry referencing RFC.

## Drawbacks

- This feature may be hard to design and implement. Requires new parser handling for `${}` vs `{}`, reactive primitive class, subscription batching, and directive overrides.
- **Parser weight**: `${}` disambiguation in `protectCurlyBraces` and `linkedom` DOM parse must not treat `${` as attribute/template syntax incorrectly.
- **Deep reactivity cost**: deep reactive objects/arrays need proxy or structured cloning; large state trees may pay per-mutation overhead. Mitigated by `.$with({ recursive: false })` shallow and `.$with({ root: false })` root-disabled modes — but both add manual `$cast` bookkeeping and a larger API surface to learn and misuse (silent suppression of root mutations).
- **Learning curve**: two binding syntaxes (`{}` vs `${}`) add concept overhead versus frameworks that are reactive by default. Mitigated by compiler warnings and Flutter-like directive escape hatches.
- **`with(ctx)` dynamism**: `compileExpr` + Proxy (`component-processor.js:320-323`) makes static analysis of `${expr}` dependencies conservative; dynamic `ctx["x"+y]` may force broader subscriptions + warning.

## Alternatives

- Not implementing reactivity means probably having almost no adoption.
- **Reactive by default** (like Svelte without runes): every `{expr}` is reactive, no `${}` needed. Simpler DX but loses explicitness, makes unused-reactive warnings impossible, and ships more subscriptions than needed (conflicts with server-resolved tree-shaking).
- **Manual DOM updates** (status quo via `bind:self`): zero compiler complexity but imperative, error-prone, and non-portable for complex state — the exact pain this RFC solves.
- **Signals API variant** (`createSignal`/`createEffect`): e.g., `let [num, setNum] = createSignal(0)`. More familiar to Solid users but diverges from Chocola's `$` sigil convention and cake mental model. `$cast`/`$react`/`$bake` keeps naming consistent with existing `$runtime` and Flutter inspiration.

Other frameworks: Svelte compiles `*.svelte` with `$state`/`$derived` via Rollup; Solid uses `createSignal`; Vue uses `ref`/`reactive` — all lend precedent for compiler-assisted reactivity this RFC proposes in explicit form.

## Unresolved questions

- What does a Chocola stateful variable primitive look like? (structured class shape, proxy vs getter/setter, array method patching, `.$with()` dispatch between `deepCast`/`shallowCast`/`shallowNoRoot`)
- How deep can reactivity be in a stateful variable? Resolved via `.$with({ recursive, root })` — deep by default, shallow when `recursive: false` requiring manual nested `$cast`, root-structural suppression when `root: false`. Confirm performance and snapshot semantics for `$bake` under all three modes, and what additional config keys (`compare`, `freeze`, etc.) should be reserved via `...`.
- How does it integrate in the lifecycle of an app (SSR/CSR)? (SSR renders static snapshot via `renderPage`; CSR hydrates with `ctx` + subscribes — confirm `$react` does not run server-side and `${}` hydrates correctly)
- How is management solved within async promises? (e.g., `let data = $cast(null); fetch(...).then(d => data = d)` — does promise resolution trigger batched updates; how to handle `await` inside `$react`)
- Should `$cast`/`$bake` directives inherit to children or be per-element only? Current spec says per-tag override, not inherited — confirm for nested component slots.
- Interaction with server-resolved `$runtime` tree-shaking: should `${}` reachability count as client-reachable independent of `$runtime` references? Proposal is yes, but needs `parser/script.js` worklist to include template reactive bindings.
