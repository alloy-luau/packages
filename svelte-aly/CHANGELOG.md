# Changelog

## 0.1.0

The first release.

- State: `state`, `derived`, `effect`, `untrack`, `get`, and `on_destroy`, on
  a reactive core of svelte-aly's own. svelte-aly depends on no other package.
  A state reads and writes `.value`, and turns into the text of its value.
- Templates: `create`, the `.alx` factory, with `if_block` and `each_block`
  for Svelte's `{#if}` and `{#each}`, and `mount` and `unmount`.
- Motion: `Spring` and `Tween`, after `svelte/motion`, with `.current`,
  `.target`, and `Spring.of` and `Tween.of`. The springs take Apple's
  parameters and presets in `Springs`. `Easing` has Svelte's curves, Roblox's
  styles, and `cubic_bezier`. `lerp` blends numbers, vectors, UDims, colors in
  Oklab, CFrames, and Rects.
- The optional `svelte-aly` ingot compiles `$state`, `$derived`, `$effect`,
  `$props`, `{#if}`, and `{#each}` to these calls.
