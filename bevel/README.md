# Bevel

Components, services, injectors, and signals for
[Alloy](https://github.com/alloy-luau/alloy), after
[Angular](https://angular.dev).

A component is a `struct` and an `impl` marked `@component`. A service is a
`struct` marked `@injectable`. The attribute contracts make the compiler check
each one: a component needs `new` and `template`, and a service needs a `new()`
with no parameters. Signals hold the state, and a template binds to them, so
the view follows each change.

Bevel has its own reactive core and its own `.alx` factory. It depends on no
other package.

Bevel needs Alloy 0.1.4 or later and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/bevel/how_to) is a
  tour of Bevel in numbered steps, from `001_hello_world.alx` to
  `030_control_flow.alx`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/bevel/examples)
  holds small games: a todo list, a shared counter, and two forms with a state
  each.

## Install

Add Bevel to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
bevel = { name = "alloy/bevel", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`, and point the `.alx` markup at
Bevel's `create`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]

[alx.factory]
backend = "table"
create = "create"
interpolate = "wrap"
```

## Quick start

A service holds the state:

```aly
import { injectable, signal, type WritableSignal } from '@packages/bevel'

--- One instance for the whole game. Any component gets it with inject().
@injectable
export struct Counter
  read count: WritableSignal<number>
end

impl Counter
  function new() -> Counter
    return new Counter { count = signal(0) }
  end

  --- Adds one to the count.
  function bump(self) -> ()
    self.count:update(function(n) return n + 1 end)
  end
end
```

A component shows it:

```alx
import { component, create, inject, mount } from '@packages/bevel'
import { Counter } from './Counter'

struct CounterButtonComponent
  read counter: Counter
end

@component
impl CounterButtonComponent
  function new(_props: {}) -> CounterButtonComponent
    return new CounterButtonComponent { counter = inject(Counter) }
  end

  function template(self) -> Instance
    return (
      <TextButton Size={new UDim2(0, 200, 0, 50)} Activated={function() self.counter:bump() end}>
        {self.counter.count()} clicks
      </TextButton>
    )
  end
end

return mount(CounterButtonComponent)
```

A client script starts the app, as Angular's `bootstrapApplication` does:

```aly
import Players from '@game/Players'
import { bootstrap } from '@packages/bevel'
import CounterButton from './CounterButton'

bootstrap(CounterButton, Players.LocalPlayer->'PlayerGui'!)
```

`bootstrap` gives a function that takes the app down again.

## Signals

A signal holds a value that changes:

```aly
import { computed, effect, signal } from '@packages/bevel'

const count = signal(0)
const double = computed(function() return count() * 2 end)

effect(function(on_cleanup)
  print(`{count()} doubled is {double()}`)
  on_cleanup(function() print('before the next run') end)
end)

count:set(4)
count:update(function(n) return n + 1 end)
```

- `count()` reads a signal. Inside a `computed` or an `effect`, the read links
  the reader to the signal.
- `count:set(value)` writes it. An equal value changes nothing and runs
  nothing.
- `count:update(fn)` writes a value from the old one.
- `count:readonly()` gives a signal that reads `count` and cannot write it.
- `computed(fn)` runs `fn` again only when something reads it and a signal
  that `fn` read has changed.
- `effect(fn)` runs `fn` now, and again after each change of a signal that
  `fn` read. `fn` gets `on_cleanup`, which keeps a function for the next run
  and for the end. `effect` gives an `EffectRef`, and `ref.destroy()` stops it.
- `untracked(fn)` runs `fn`, and its reads link to nothing.
- `batch(fn)` runs the effects that the writes in `fn` mark once, at the end.

`Signal<T>` is the type of a signal that code reads, and `WritableSignal<T>`
the type of one that code also writes. A `WritableSignal<T>` fits where a
`Signal<T>` goes.

An effect that keeps writing a signal that it reads stops with an error after
10000 runs in one write.

## Components

`@component` sits on the `impl` of a struct. The contract asks for two public
functions:

- `new(props)` takes the inputs of the tag and builds the component.
- `template(self)` returns the `Instance` that the tag shows.

An input is a field of the props. An output is a callback in the props:

```aly
interface TodoItemProps
  todo: Todo
  toggled: () -> ()
end
```

A component may also write `on_init(self)`, which runs after `new` and before
`template`, and `on_destroy(self)`, which runs when the view goes. They are
Angular's `ngOnInit` and `ngOnDestroy`.

`mount(Class)` turns the class into a tag: a function from props to an
`Instance`. Return it as the module's value, and another `.alx` file writes the
module as a tag. Each mount builds the component in an owner of its own. When
the view goes, the owner goes, and with it each effect that the component made.

`on_destroy(fn)` registers a cleanup on the current owner from any code inside
a component, as Angular's `DestroyRef.onDestroy` does.

## Templates

A tag lowers to `create('Frame')({ ... })`. `create` makes the Instance and
reads each key of the table:

- A plain value sets the property.
- A signal, or a function, binds the property. The property follows the value.
- A function on an event, such as `Activated`, connects to the event.
- Each child goes under the Instance. A child function builds again when the
  signals that it reads change, and what it built before goes.
- `Parent` comes last.

Text with holes in it binds the `Text`: `<TextLabel>{count()} left</TextLabel>`.

The control flow:

| Angular | Bevel |
| --- | --- |
| `@for (item of items(); track item.id)` | `each(items, function(item, index) ... end, function(item) return item.id end)` |
| `@if (open()) { } @else { }` | `when(open, render, otherwise)` |
| `[(ngModel)]="draft"` | `Text={model(draft)}` |

```alx
{each(self.store.todos, function(todo)
  return <TodoItem todo={todo} toggled={function() self.store:toggle(todo.id) end} />
end, function(todo) return todo.id end)}

{when(function() return #self.store.todos() == 0 end, function()
  return <TextLabel>Nothing to do</TextLabel>
end)}
```

`each` keeps one view for each key while the key stays in the list. `index()`
reads the place of the item, and a view that is a GuiObject gets its place as
`LayoutOrder`. `when` builds again only when the truth of its condition
changes.

## Services and inject

`@injectable` sits on a struct. The contract asks its impl for `new()` with no
parameters, because the injector calls `new()` itself.

`inject(Class)` returns the instance of the nearest `provide` that holds the
class. With none, it returns the root instance. The root injector builds that
instance on the first call and keeps it.

In a component's `new`, the field gives the type:
`new ShopComponent { wallet = inject(Wallet) }`. In a local, write the type of
the result: `const wallet = inject<<Wallet>>(Wallet)`. Luau cannot read the
type of an instance from its struct, so the result is `unknown` without it.

`inject` stops with an error when the class has no `@injectable`.

## Scoped services and provide

`@injectable('form')`, or any scope other than `'root'`, keeps a service out of
the root injector. Such a service needs a `provide` above it, and `inject`
stops with an error without one.

`provide(classes, build)` builds a new instance of each class, then runs
`build`. Each `inject` in what `build` makes reads those instances first. An
inner `provide` reads through to the one above it, so the lookup walks up the
tree, as Angular's element injectors do. A service that writes
`on_destroy(self)` hears when the `provide` goes.

```alx
function template(self) -> Instance
  return provide([ FormState ], function()
    return <Form />
  end)
end
```

## API

| Name | What it is |
| --- | --- |
| `signal(value)` | A `WritableSignal`: call it to read, `:set` and `:update` to write. |
| `computed(fn)` | A `Signal` that `fn` derives from other signals. |
| `effect(fn)` | Runs `fn` now and after each change. Gives an `EffectRef`. |
| `untracked(fn)` | Runs `fn` with no links. |
| `batch(fn)` | Runs the marked effects once, at the end of `fn`. |
| `read(value)` | Reads a signal, or calls a function. |
| `root(fn)` | Runs `fn` in an owner of its own. `fn` gets the dispose function. |
| `on_destroy(fn)` | Runs `fn` when the current owner goes. |
| `create(class)` | The `.alx` factory. |
| `each(items, render, track?)` | One view for each item. |
| `when(condition, render, otherwise?)` | A view while the condition holds. |
| `model(signal)` | Binds a property both ways. |
| `@component` | Marks a component impl. Requires `new` and `template(self)`. |
| `mount(Class)` | Turns a component class into a tag. |
| `bootstrap(tag, target)` | Builds the root component under `target`. |
| `@injectable(scope = 'root')` | Marks a service struct. Requires `new()`. |
| `inject(Class)` | The instance of the nearest `provide`, else the root instance. |
| `provide(classes, build)` | Runs `build` with a new instance of each class. |
| `Signal<T>`, `WritableSignal<T>`, `Readable<T>` | The types of signals, and of what Bevel reads. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API.

Bevel lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `bevel` folder. Each
package there is a member of one Ember workspace and keeps its own version.
Run the commands above from `bevel`. `embr publish` from `bevel` publishes
Bevel alone.

The `how_to` and `examples` files import `@packages/bevel`. To check them, copy
them into a game that installs Bevel, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/bevel/LICENSE).
