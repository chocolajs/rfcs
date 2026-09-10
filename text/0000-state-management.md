- Start Date: 2026-09-06
- RFC PR: (leave this empty)
- Chocola Issue: (leave this empty)

# State management

## Summary

A core feature of web frameworks is reactivity. To be production-ready, Chocola will have to reach this milestone for possible massive adoption.

## Motivation

Managing state in Chocola is hand-made and can brake easily. For example, if we want to create a counter, we'll have to manually update the DOM:

```html
<script>
  let btn;
  let numDisplay;

  let num = 0;

  function $runtime() {
    btn.addEventLister("click", () => {
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

This puts a lot of heavy thinking for complex logic and is hard to mantain.

## Detailed design

### Technical Background

Reactivity lets developers to bind values through their components to make them update synchronously without having to do it manually, making it safe and easy.

Most frameworks do it by default. E.g., Svelte uses runes like `$state` and `$derived` to manage reactivity.

### Implementation

I propose Chocola provides `$cast`, `$react` and `$bake` as its main way to manage state.

See [state](https://github.com/chocolajs/chocola/blob/main/specs/state.md) specs for detailed information.

## How we teach this

This continues the use of the `$` sigil for Chocola features and lends state managmente terminology from Flutter for stateful and stateless elements/components and variables.

This would imply creating a new docs section for explaining how reactivity works in Chocola. Most users will find this familiar since it's a concept present in most commercial frameworks.

## Drawbacks

This feature may be hard to design and implement.

## Alternatives

Not implementing reactivity means probably having almost no adoption.

## Unresolved questions

- What does a Chocola stateful variable primitive looks like?
- How deep can reactivity be in a stateful variable?
- How does it integrate in the lifecycle of an app (SSR/CSR)?
- How is managment solved within async promises?