# svelte-aly

Svelte's runes, blocks, and motion for
[Alloy](https://github.com/alloy-luau/alloy), after
[Svelte](https://svelte.dev).

A component is a `.alx` file that starts with `<script>`. The script holds
the state, and the markup after it shows the state. A state is a value that
changes, and the markup that reads it follows each change.

svelte-aly has two parts:

- **The package**, `alloy/svelte-aly`: a reactive core, the `.alx` factory,
  the blocks, and springs and tweens with Apple's parameters. It works alone,
  and it depends on no other package.
- **The ingot**, `svelte-aly` in
  [alloy-luau/ingots](https://github.com/alloy-luau/ingots): optional. It
  compiles Svelte's syntax to the package: `$state`, `$derived`, `$effect`,
  `$props`, `{#if}`, and `{#each}`.

svelte-aly needs Alloy 0.1.7 or later, which reads `<script>` in a `.alx`
file, and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/svelte-aly/how_to)
  is a tour in numbered steps, from `001_components.alx` to
  `040_without_the_ingot.alx`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/svelte-aly/examples)
  holds two small games: the counter from Svelte's tutorial, and a shop that
  springs open.

## Install

Add the package to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
svelte-aly = { name = "alloy/svelte-aly", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`, and point the `.alx` markup at
the package's `create`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]

[alx.factory]
backend = "table"
create = "create"
interpolate = "wrap"
```

For Svelte's syntax, add the ingot too, then run `alloy ingot install`:

```toml
[ingots]
svelte-aly = { repo = "alloy-luau/ingots" }
```

## Quick start

With the ingot, a component reads as it does in Svelte:

```alx
<script>
  let count = $state(0)
  let doubled = $derived(count * 2)
  let items = $state([ 'sword', 'shield' ])

  $effect(function()
    print('count changed:', count)
  end)

  function add()
    items = items:concat([ `item {#items + 1}` ])
  end
</script>

<ScreenGui ResetOnSpawn={false}>
  <Frame Size={UDim2.fromScale(1, 1)} BackgroundTransparency={1}>
    <UIListLayout SortOrder={Enum.SortOrder.LayoutOrder} />
    <TextButton Size={new UDim2(0, 260, 0, 40)} Activated={function() count += 1 end}>
      Clicked {count} times (doubled: {doubled})
    </TextButton>

    {#if count > 5}
      <TextLabel Size={new UDim2(0, 260, 0, 30)}>That's a lot of clicks</TextLabel>
    {/if}

    {#each items as item (item)}
      <TextLabel Size={new UDim2(0, 260, 0, 30)}>{item}</TextLabel>
    {/each}

    <TextButton Size={new UDim2(0, 260, 0, 40)} LayoutOrder={1000} Activated={add}>
      Add item
    </TextButton>
  </Frame>
</ScreenGui>
```

A client script shows it:

```aly
import Players from '@game/Players'
import { mount } from '@packages/svelte-aly'
import Counter from './Counter'

mount(Counter, { target = Players.LocalPlayer->'PlayerGui'! })
```

## Component files

A `.alx` file that starts with `<script>` is a component file. The compiler
makes the script the body of a function of `props`, and the markup after
`</script>` the nodes that the function gives. The module returns the
function, so another file imports it and uses it as a tag: `<Counter />`.

- The imports at the top of the script stay at the top of the module. The
  code after them runs once for each copy of the component.
- After `</script>` the file is markup, so a comment there is `<!-- -->`.
- With the ingot, the script needs no import of the package or of `create`,
  and each `function name()` of the script is local.

## Runes

The runes need the ingot. Each one becomes a call of the package.

| Rune | Becomes |
| --- | --- |
| `let n = $state(value)` | `local n = state(value)`; each read and write of `n` is `n.value` |
| `let d = $derived(expr)` | `const d = derived(function() return expr end)` |
| `$derived.by(fn)` | `derived(fn)` |
| `$effect(fn)` | `effect(fn)`; a function that `fn` gives back is its cleanup |
| `let { a, b } = $props()` | the props of the component; each read is `get(a)` |

- A state that code writes is `local`, any other is `const`. An annotation
  names the state type: `let items: Item[] = $state([])`.
- A markup hole that reads a rune becomes a function, so the factory binds
  it. On a component tag it becomes a derived value, so the child follows it.
- A change in place, such as `items:push(x)` or `table.insert(items, x)`, runs
  no effect. Assign the state again after it, `items = items`, or assign a new
  table. The `svelte-aly/mutation` lint warns.
- Runes work in a `.aly` file too, for state that many components share.

## Blocks

`{#if}` and `{#each}` need the ingot.

```alx
{#if health > 60}
  <TextLabel>Healthy</TextLabel>
{:else if health > 0}
  <TextLabel>Hurt: {health}</TextLabel>
{:else}
  <TextLabel>Down</TextLabel>
{/if}

{#each items as item, i (item.id)}
  <TextLabel>{i}. {item.name}</TextLabel>
{:else}
  <TextLabel>The bag is empty</TextLabel>
{/each}
```

- `{#if}` builds its nodes again only when the branch changes.
- `{#each}` with a key keeps the nodes of an item while its key stays in the
  list, and `i` follows its place. With no key, the nodes belong to a place.
  Each GuiObject of an item gets its place as `LayoutOrder`.
- A branch and an item each have an owner. When one goes, its effects stop
  and its nodes go.

## Springs

A spring moves a value to its target, the way Apple's springs move a view.
`new Spring(value)` holds its own target; `Spring.of(fn)` follows what `fn`
reads. A template binds a spring as it binds a state:

```alx
<script>
  import { Spring, Springs } from '@packages/svelte-aly'

  let open = $state(false)

  const size = Spring.of(function()
    return if open then new UDim2(0, 320, 0, 200) else new UDim2(0, 160, 0, 48)
  end, Springs.bouncy(0.45))
</script>

<TextButton Size={size} Activated={function() open = not open end}>Menu</TextButton>
```

The options take one of three forms, as Apple's springs do:

| Form | Fields | Apple |
| --- | --- | --- |
| Perceptual | `duration` (0.5), `bounce` (0) | `Spring(duration:bounce:)` |
| Response | `response`, `damping_ratio` | `.spring(response:dampingFraction:)` |
| Physical | `mass` (1), `stiffness` (100), `damping` (10) | `Spring(mass:stiffness:damping:)` |

- `Springs.smooth(duration?, extra_bounce?)`, `Springs.snappy(...)`, and
  `Springs.bouncy(...)` are Apple's presets, with a bounce of 0, 0.15, and
  0.3. `Springs.interactive()` is for a value that follows a drag.
- `.current` reads the value now, and `.target` the goal; write `.target` to
  move it. `.resting` is true once the spring rests.
- `spring:set(value, { instant = true })` jumps, `spring:impulse(v)` pushes,
  and `spring:velocity()` reads the velocity.
- A spring moves numbers, Vector2, Vector3, UDim, UDim2, Color3, CFrame, and
  Rect. A Color3 moves in Oklab, so a blend keeps its lightness.
- The spring solves the damped oscillator exactly at each frame. It moves the
  same at any frame rate, and a new target keeps the velocity of the motion
  before it.

## Tweens

A tween moves a value to its target along a curve, in a set time:

```aly
import { Easing, Tween } from '@packages/svelte-aly'

local color = new Tween(Color3.fromRGB(60, 160, 90), { duration = 0.4, easing = Easing.cubic_out })
color.target = Color3.fromRGB(220, 60, 60)
```

- `duration` is 0.25 seconds and `easing` is `Easing.ease_in_out` by default.
  `delay` waits before the move.
- `Tween.of(fn)` follows what `fn` reads. `tween:set(value, options)` moves
  once with other options.
- `Easing` has Svelte's curves in snake case, such as `Easing.cubic_out` and
  `Easing.elastic_in_out`; Roblox's styles, `Easing.style('Quad', 'Out')`;
  Core Animation's `ease_in`, `ease_out`, and `ease_in_out`; and
  `Easing.cubic_bezier(x1, y1, x2, y2)`.

## Without the ingot

The package works alone. Call the functions in place of the runes, use
`.value`, and call the blocks in a hole:

```alx
<script>
  import { create, derived, if_block, state } from '@packages/svelte-aly'

  local count = state(0)
  const doubled = derived(function() return count.value * 2 end)
</script>

<TextButton Size={new UDim2(0, 260, 0, 40)} Activated={function() count.value += 1 end}>
  Clicked {count} times (doubled: {doubled})
</TextButton>

{if_block(function() return count.value > 5 end, function()
  return <TextLabel Size={new UDim2(0, 260, 0, 30)}>That's a lot of clicks</TextLabel>
end)}
```

- A state or a derived value in a hole binds as it stands: `{count}`.
- A hole that computes from `.value` needs a function:
  `{function() return count.value + 1 end}`.
- `if_block` takes pairs: a condition function, then a build function. `true`
  stands for `{:else}`.
- `each_block(items, key, render, fallback)`. A nil key keys by place.

## The ingot's lints

| Lint | Level | What it reports |
| --- | --- | --- |
| `shadowed_rune` | deny | A second binding with the name of a rune. |
| `derived_write` | deny | A write to a `$derived` value. |
| `mutation` | warn | A change in place to the table of a state. |
| `props_outside_script` | deny | `$props()` outside a component file. |
| `bad_each` | deny | An `{#each}` the ingot cannot read. |
| `unclosed_block` | deny | A block with no close, or a close with no block. |
| `unsupported_block` | deny | `{#await}`, `{#key}`, `{#snippet}`, `{@html}`, and the other blocks it does not compile. |
| `style_needs_silk` | warn | A `<style>` in a project without the Silk ingot. |

`<style>` works with the [Silk](https://github.com/alloy-luau/ingots) ingot,
which turns it into a StyleSheet.

## API

| Name | What it is |
| --- | --- |
| `state(value)` | A `State`: read and write `.value`. |
| `derived(fn)` | A `Derived`: `.value` reads what `fn` gives. |
| `effect(fn)` | Runs `fn` now and after each change. A function that `fn` gives back is its cleanup. Gives the function that stops it. |
| `untrack(fn)` | Runs `fn` with no links. |
| `get(value)` | Reads a prop: a state gives its value, anything else gives itself. |
| `is_state(value)` | Whether a value is a state, a derived value, a spring, or a tween. |
| `on_destroy(fn)` | Runs `fn` when the current owner goes. |
| `create(class)` | The `.alx` factory. |
| `if_block(...)` | The block of `{#if}`. |
| `each_block(items, key, render, fallback)` | The block of `{#each}`. |
| `mount(component, { target, props })` | Shows a component under `target`. Gives an `App`. |
| `unmount(app)` | Stops the effects of a component and destroys its nodes. |
| `new Spring(value, options?)`, `Spring.of(fn, options?)` | A `Spring`. |
| `Springs.smooth`, `.snappy`, `.bouncy`, `.interactive` | Apple's spring presets. |
| `new Tween(value, options?)`, `Tween.of(fn, options?)` | A `Tween`. |
| `Easing` | The curves of a tween. |
| `lerp(a, b, t)` | A blend of two values of one kind. |
| `step(dt)`, `manual_frames()` | Move the frame loop by hand. |
| `State<T>`, `Derived<T>`, `Spring<T>`, `Tween<T>` | The types of the values. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They
run on lest, with Alloy's test shim for the Roblox API.

svelte-aly lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `svelte-aly` folder. Run
the commands above from `svelte-aly`. `embr publish` from `svelte-aly`
publishes it alone.

`ember.toml` gives the colours of the block tags as `[highlight]` rules,
for the code blocks of this README on the Ember site. Publish with embr
1.0.0-rc.4 or later. An older embr drops the rules, and the site then
shows the block tags with no colour of their own.

The `how_to` and `examples` files import `@packages/svelte-aly`, and most of
them use the ingot. To check them, copy them into a game that installs the
package and the ingot, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/svelte-aly/LICENSE).
