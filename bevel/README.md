# Bevel

Components, services, and injectors for [Alloy](https://github.com/alloy-luau/alloy), after
[Angular](https://angular.dev), on [Vide](https://github.com/centau/vide).

A component is a `struct` and an `impl` marked `@component`. A service is a `struct` marked
`@injectable`. The attribute contracts make the compiler check each one: a component needs `new`
and `template`, and a service needs a `new()` with no parameters. `inject()` gives a component its
services, and `provide()` gives a part of the tree its own instances.

Bevel needs Alloy 0.1.4 or later and embr 0.8.1 or later.

## Contents

- [Install](#install)
- [Quick start](#quick-start)
- [Components](#components)
- [Services and inject](#services-and-inject)
- [Scoped services and provide](#scoped-services-and-provide)
- [Signals](#signals)
- [API](#api)
- [Development](#development)

## Install

Add Bevel and Vide to `ember.toml`, then run `embr install`:

```toml
[dependencies]
bevel = { name = "alloy/bevel", version = "^0.1.0" }
vide = { name = "centau/vide", version = "^0.4.1", index = "wally" }

[indices]
wally = "https://github.com/UpliftGames/wally-index"
```

Mount the packages folder in `alloy.toml`, and point the `.alx` factory at Vide's `create`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]

[alx.factory]
backend = "table"
create = "create"
interpolate = "wrap"
```

## Quick start

```alloy
import { injectable, signal, type Signal } from '@packages/bevel'

-- One instance for the whole game. Any component gets it with inject().
@injectable
export struct Counter
  read count: Signal<number>
end

impl Counter
  function new(): Counter
    return new Counter { count = signal(0) }
  end

  function bump(self): ()
    self.count(self.count() + 1)
  end
end
```

```alloy
import { create } from '@packages/vide'
import { component, inject, mount } from '@packages/bevel'
import { Counter } from './Counter'

struct CounterButtonComponent
  read counter: Counter
end

@component
impl CounterButtonComponent
  function new(_props: {}): CounterButtonComponent
    return new CounterButtonComponent { counter = inject(Counter) }
  end

  function template(self): Instance
    return (
      <TextButton Activated={function() self.counter:bump() end}>
        {function() return `{self.counter.count()} clicks` end}
      </TextButton>
    )
  end
end

return mount(CounterButtonComponent)
```

Another `.alx` file imports the result and uses it as a tag: `<CounterButton />`.

## Components

`@component` sits on the `impl` of a struct. The contract asks for two public functions:

- `new(props)` takes the inputs of the tag and builds the component.
- `template(self)` returns the `Instance` that the tag shows.

An input is a field of the props. An output is a callback in the props:

```alloy
interface TodoItemProps
  todo: Todo
  toggled: () -> ()
end
```

`mount(Class)` turns the class into a function from props to an `Instance`. Return it as the
module's value, and the file is a tag.

## Services and inject

`@injectable` sits on a struct. The contract asks its impl for `new()` with no parameters, because
the injector calls `new()` itself.

`inject(Class)` returns the instance of the nearest `provide` that holds the class. With none, it
returns the root instance. The root injector builds that instance on the first call and keeps it.

`inject` stops with an error when the class has no `@injectable`.

## Scoped services and provide

`@injectable('form')`, or any scope other than `'root'`, keeps a service out of the root injector.
Such a service needs a `provide` above it, and `inject` stops with an error without one.

`provide(classes, build)` builds a new instance of each class, then runs `build`. Each `inject`
inside `build` reads those instances first. An inner `provide` reads through to the one above it,
so the lookup walks up the tree, as Angular's element injectors do.

```alloy
function template(self): Instance
  return provide([ FormState ], function()
    return <Form />
  end)
end
```

`provide` must run inside a Vide scope: a template, a `root`, or a `mount`. It uses Vide's
`context()`.

## Signals

`signal` is Vide's `source`, and `computed` is Vide's `derive`. `Signal<T>` is the type of a
signal. Call a signal with no argument to read it, and with one argument to write it.

```alloy
const count = signal(0)
const double = computed(function() return count() * 2 end)
count(4)
print(double()) -- 8
```

## API

| Name | What it is |
|---|---|
| `@component` | Marks a component impl. Requires `new` and `template(self)`. |
| `@injectable(scope = 'root')` | Marks a service struct. Requires `new()`. |
| `inject(Class)` | The instance of the nearest `provide`, else the root instance. |
| `provide(classes, build)` | Runs `build` with a new instance of each class. |
| `mount(Class)` | Turns a component class into a function from props to an `Instance`. |
| `signal(value)` | Vide's `source`. |
| `computed(fn)` | Vide's `derive`. |
| `Signal<T>` | The type of a signal. |

## Development

```sh
embr install   # Vide, into packages/roblox
alloy build    # src into build
alloy test     # writes the specs under tests/
lest           # runs them
alloy flux     # the type check
```

The tests are the `@test` namespace at the end of `src/index.aly`. Vide connects to
`RunService.Heartbeat` as it loads, so the tests need an Alloy whose test shim carries that event.

`embr publish --dry-run` builds the package and lists the files it would ship.

## License

MIT. See [LICENSE](LICENSE).
