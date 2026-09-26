# Forge

OOP providers for [Alloy](https://github.com/alloy-luau/alloy), after
[Chief](https://github.com/ryancundiff/chief).

A provider is a singleton: a `struct`, an `impl`, and one instance as its
module's default export. `@Provider` lists the lifecycles the provider
uses. Its contract makes the compiler check that the impl writes a
private method for each one. Forge loads the providers, runs `Init` in
order, wires the lifecycles, and starts them.

Forge has Chief's parts in one package: the core, lifecycles,
dependencies, traits, and Bin.

## Contents

- [Quick start](#quick-start)
- [Boot sequence](#boot-sequence)
- [Providers](#providers)
- [Lifecycles](#lifecycles)
- [Dependencies](#dependencies)
- [Traits](#traits)
- [Bin](#bin)
- [Extensions](#extensions)
- [Errors](#errors)
- [API](#api)
- [Using Forge from a game](#using-forge-from-a-game)
- [Alloy issues Forge works around](#alloy-issues-forge-works-around)
- [Development](#development)
- [Inspired by Chief](#inspired-by-chief)

## Quick start

A main script only sets up Forge:

```lua
import { Forge } from 'forge/src/index'

new Forge()
  :AddProviders(script.Parent->'Providers'!:GetChildren())
  :AddLifecycles([ Forge.every(60, 'Autosave') ])
  :Start()
```

A provider is a struct and its impl:

```lua
import { HashMap } from '@alloy/std/collections'
import { Forge, Lifecycle, Provider } from 'forge/src/index'
import type { DataProvider } from './DataProvider'

--- Keeps each player in the game.
@Provider({ lifecycles = [ Lifecycle.Init, Lifecycle.PlayerAdded, 'Autosave' ], priority = 10 })
export struct PlayerProvider
  private data: DataProvider = Forge.use('DataProvider')
  private players: HashMap<string, Player> = HashMap.new()
end

impl PlayerProvider
  private function Init(self) -> ()
    -- Runs in order, before any Start. It must not yield.
  end

  private function PlayerAdded(self, player: Player) -> ()
    self.players:set(player.Name, player)
  end

  private function Autosave(self, _dt: number) -> ()
    self.data:save_all()
  end

  --- A player in the game, by name.
  function get(self, name: string) -> Player?
    return self.players:get(name)
  end
end

export default new PlayerProvider {}
```

Other code uses the provider through its module:

```lua
import PlayerProvider from '../Providers/PlayerProvider'

print(PlayerProvider:get('Builderman'))
```

## Boot sequence

`Forge:Start()` runs these phases in order:

```
PreLoad -> Load -> PostLoad -> PreInit -> Init -> PostInit -> PreStart -> Start -> PostStart
```

1. `PreLoad`: the extensions hear it.
2. `Load`: Forge requires each provider module, and each fires `Loaded`.
3. Forge sorts the providers by priority, higher first. Ties keep the
   order they were added in.
4. `PostLoad`: the extensions hear it, and one may call `LoadProvider`.
   Then Forge loads each provider that a `Forge.use` names by its
   ModuleScript and that no one added.
5. `PreInit`: Forge orders the providers so that each one inits after
   the providers it uses, and puts each provider into the fields that
   use it. Then the extensions hear it; one may reorder the entries.
   After this phase the order is final.
6. `Init`: each listed `Init` runs, in order, on its own thread. An
   `Init` that yields errors with `NoYield:`. Each provider fires
   `Initialized`.
7. `PostInit`: Forge wires the lifecycles, then the extensions hear it.
8. `PreStart`: the extensions hear it.
9. `Start`: each listed `Start` is spawned. Each provider fires
   `Started`.
10. `PostStart`: Forge starts to watch the trait tags, then the
    extensions hear it.

With `Verbose` on, which is the default, Forge then prints a summary:

```
Forge (Server) started 6 providers in 4ms
```

## Providers

A provider is one instance of a struct that carries `@Provider`:

```lua
export attribute Provider(lifecycles: (Lifecycle | string)[] = {}, priority: number = 0) on struct, impl as
  requires private function each lifecycles
end
```

- `lifecycles` lists each method Forge calls: a `Lifecycle`, or the
  method name of a lifecycle added with `AddLifecycles`, as a string.
- `priority` orders `Init`. A higher priority inits first. A
  dependency still inits before the provider that uses it.
- `@Provider()` with no argument makes a provider that uses no
  lifecycle.

The contract. The `each` clause writes one `requires private function`
clause for each entry of the list. A provider that lists `Start` and
writes no private `Start` method does not compile:

```
error(3.11): AttributeContract: `@Provider` requires a private function `Start`; `PlayerProvider` declares none
```

The contract fixes the names it asks for, so `naming_convention` does
not report `Init` or `PlayerAdded` in a project that writes methods in
snake_case.

Forge checks the same contract when it loads a provider, and errors with
`BadArgument:` when a listed method is missing. It also warns when a
provider writes `Init` or `Start` and does not list it, since Forge does
not call a method that the provider does not list.

The singleton. The module's default export is the one instance. Forge
reads `getmetatable(instance)` to find the struct, and
`Attributes.get(struct, Provider)` for the options. A second instance of
the same struct errors with `BadArgument:`.

Private methods. `private` is a compile-time check in Alloy. At runtime
the methods sit on the struct's table like any other, so Forge calls
them through the instance it holds. The user's methods stay private, and
no other module of the game can call `Init` or `Start`.

Where the attribute goes. `@Provider` may sit on the struct or on its
impl, and the contract reads both, since a struct and its impls are one
type. Alloy 0.1.0-rc.8 writes only the struct's attributes at runtime,
so put `@Provider` on the struct until the impl form ships. See
[Alloy issues Forge works around](#alloy-issues-forge-works-around).

## Lifecycles

A lifecycle binds an event to a method. A provider opts in when it lists
the method in `@Provider`, and the contract makes it write the method.
Forge wires every lifecycle at `PostInit`, after each `Init` and before
any `Start`, so a handler may use any provider.

### Built-in lifecycles

A built-in wires itself once a provider lists it. The game does not
register it.

| Lifecycle | Method | Notes |
| --- | --- | --- |
| `Init` | `Init(self)` | Runs in order. It must not yield. |
| `Start` | `Start(self)` | Spawned once each `Init` has run. |
| `PlayerAdded` | `PlayerAdded(self, player)` | Also fires for each player in the game at wiring. |
| `PlayerRemoving` | `PlayerRemoving(self, player)` | |
| `CharacterAdded` | `CharacterAdded(self, player, character)` | Also fires for each character at wiring. |
| `CharacterRemoving` | `CharacterRemoving(self, player, character)` | |
| `AppearanceLoaded` | `AppearanceLoaded(self, player, character)` | |
| `HumanoidDied` | `HumanoidDied(self, player, character, humanoid)` | |
| `Closing` | `Closing(self)` | `game:BindToClose`. The server waits for each handler (Await). Server only. |
| `PreRender` | `PreRender(self, dt)` | Each frame, before it draws. Inline. Client only. |
| `PreAnimation` | `PreAnimation(self, dt)` | Each frame, before animations step. Inline. |
| `PreSimulation` | `PreSimulation(self, dt)` | Each frame, before physics. Inline. |
| `PostSimulation` | `PostSimulation(self, dt)` | Each frame, after physics. Inline. |
| `Heartbeat` | `Heartbeat(self, dt)` | The old name for `PostSimulation`. Write one of the two. Inline. |
| `LocalCharacterAdded` | `LocalCharacterAdded(self, player, character)` | The local player. Also fires for the character at wiring. Client only. |
| `LocalCharacterRemoving` | `LocalCharacterRemoving(self, player, character)` | Client only. |
| `LocalAppearanceLoaded` | `LocalAppearanceLoaded(self, player, character)` | Client only. |
| `LocalHumanoidDied` | `LocalHumanoidDied(self, player, character, humanoid)` | Client only. |

A provider that lists a client-only lifecycle on the server errors with
`BadCall:`, and so does a server-only one on the client.

### every

`Forge.every(seconds, method)` makes a lifecycle that calls
`method(self, dt)` about every `seconds`, with the real time since the
last tick. A lag spike gives one late tick, not a burst. Each call makes
a new lifecycle, so two intervals can drive two methods.

```lua
new Forge()
  :AddProviders(providers)
  :AddLifecycles([ Forge.every(60, 'Autosave'), Forge.every(1, 'Regenerate') ])
  :Start()
```

A provider lists the method by its name: `lifecycles = [ 'Autosave' ]`.

### Custom lifecycles

A lifecycle is a table:

```lua
export type CustomLifecycle = {
  Method: string,
  Connect: RBXScriptSignal | (fire: Fire) -> RBXScriptConnection?,
  Dispatch: Dispatch?,
}
```

- `Method`: the name a provider lists and writes.
- `Connect`: a signal, or a function that takes `fire` and returns a
  connection, or nil when nothing stays connected. Call `fire(...)` to
  hand the arguments to each provider that lists the method.
- `Dispatch`: how `fire` runs the handlers. `Spawn` when nil.

```lua
import Players from '@game/Players'
import { Dispatch, type CustomLifecycle } from 'forge/src/index'

--- Each chat message of each player.
const chatted: CustomLifecycle = {
  Method = 'PlayerChatted',
  Connect = function(fire) -> RBXScriptConnection?
    return Players.PlayerAdded:Connect(function(player: Player) -> ()
      player.Chatted:Connect(function(message: string) -> ()
        fire(player, message)
      end)
    end)
  end,
  Dispatch = Dispatch.Spawn,
}

new Forge():AddProviders(providers):AddLifecycles([ chatted ]):Start()
```

A string entry in `lifecycles` names a custom method, so the attribute
takes `(Lifecycle | string)[]`. The contract reads a string entry as the
method name, the same way it reads an enum variant.

Two lifecycles may share a method, since one method may listen to two
events. One lifecycle added twice errors, since each copy would fire
every handler again. A provider that lists a method that no lifecycle
fires errors with `BadArgument:` at `PostInit`, since the handler would
never run.

An extension may add a lifecycle up to `PreInit`. After Forge wires them,
`AddLifecycle` errors with `BadCall:`.

### Dispatch

| Mode | What `fire` does |
| --- | --- |
| `Spawn` | Each handler runs on its own thread, and `fire` returns at once. A slow or failing handler cannot hold up the others. The default. |
| `Inline` | The handlers run one after another on the event's thread. No thread per event, so the per-frame built-ins use it. A slow handler delays the rest, and an error stops the dispatch. |
| `Await` | The handlers run at once, and `fire` returns when the last one ends. An error in one does not stop the others; Forge warns with `LifecycleError:`. For an event source that must wait, such as `BindToClose`. |

## Dependencies

A provider field holds another provider:

```lua
import { Forge, Lifecycle, Provider } from 'forge/src/index'
import type { DataProvider } from './DataProvider'

@Provider({ lifecycles = [ Lifecycle.Init ] })
export struct ShopProvider
  private data: DataProvider = Forge.use('DataProvider')
end

impl ShopProvider
  private function Init(self) -> ()
    -- self.data is the DataProvider, and its Init has run.
    print(self.data:coins())
  end
end

export default new ShopProvider {}
```

`Forge.use(target)` returns a marker. The field keeps it until
`PreInit`, when Forge puts the provider it names there.

- A name, `Forge.use('DataProvider')`, finds a provider the game added
  by its ModuleScript's name.
- A ModuleScript, `Forge.use(script.Parent->'DataProvider'!)`, also
  loads that provider when no one added it.

Forge orders `Init` by the uses (Kahn's algorithm). Among the providers
whose uses have all run, the higher priority goes first. A cycle errors
with its path:

```
CyclicDependency: Forge cannot order Init: ShopProvider -> DataProvider -> ShopProvider.
```

A use that names a provider Forge never loaded errors with
`MissingDependency:`.

Why a marker, not a field attribute. A field attribute such as
`@use('DataProvider') data: DataProvider` reads well, but
`new ShopProvider {}` then needs a value for `data`, since a field with
no default must be set. The marker is that value. It types as the field's
type, so the field needs no cast.

Why no circular require. The field's type comes from
`import type { DataProvider }`, so one provider's module never requires
another's, and two providers may name each other's types. Alloy
0.1.0-rc.8 still runs the require of a type-only import unless the game
sets `[emit] erase_type_imports = true`. Set it: without it, two
providers that name each other's types require each other in a loop.

## Traits

A trait binds behavior to each instance that carries a CollectionService
tag. Each tagged instance gets an object of its own, so a trait is not a
singleton, and its module exports the struct.

```lua
import { Attribute, Bin, Child, Trait, TraitLifecycle } from 'forge/src/index'

--- A door that opens on its prompt.
@Trait({ tag = 'Door', instance = 'BasePart', lifecycles = [ TraitLifecycle.Init, TraitLifecycle.Destroy ] })
export struct Door
  instance: BasePart
  bin: Bin
  @Attribute('Locked', 'boolean') locked: boolean
  @Attribute('OpenAngle', 'number') open_angle: number = 90
  @Child('Prompt', 'ProximityPrompt', 5) prompt: ProximityPrompt
end

impl Door
  private function Init(self) -> ()
    self.bin:Add(self.prompt.Triggered:Connect(function() -> ()
      if not self.locked then
        print(`{self.instance.Name} opens by {self.open_angle} degrees`)
      end
    end))
  end

  private function Destroy(self) -> ()
    print('the door is gone')
  end
end

export default Door
```

```lua
new Forge()
  :AddProviders(providers)
  :AddTraits(script.Parent->'Traits'!:GetChildren())
  :Start()
```

`@Trait(tag, instance = 'Instance', lifecycles = {})`:

- `tag`: the CollectionService tag to watch.
- `instance`: the class the tagged instance must be.
- `lifecycles`: the steps the impl writes.

The contract asks for an `instance` field and a `bin` field, which Forge
fills, and a private method for each listed step.

The field contracts:

- `@Attribute(name, kind)` fills a field from the attribute `name` of the
  instance, which must be of `typeof` kind `kind`, such as `'number'`.
- `@Child(name, class_name = 'Instance', wait = 0)` fills a field from
  the child `name`, which must be a `class_name`. With `wait` above 0,
  Forge waits that many seconds for it.

A field with a default is optional. Forge writes the default back to the
instance, so it shows in Studio. An instance that lacks a required
attribute or child, or that has one of the wrong kind or class, does not
bind, and Forge warns with `BadInstance:` and the reason.

The steps:

| Step | Method | Notes |
| --- | --- | --- |
| `Init` | `Init(self)` | Runs once the object is built. It must not yield. |
| `Start` | `Start(self)` | Spawned after `Init`. The unbind cancels it. |
| `AttributeChanged` | `AttributeChanged(self, field, value)` | An `@Attribute` field changed on the instance; the field holds the new value already. |
| `Destroy` | `Destroy(self)` | Runs on unbind, before the bin empties. It must not yield. |

The binding:

1. Watching starts at `PostStart`, after the whole boot, so a trait may
   use any provider.
2. Bind: Forge resolves the children, waiting where `wait` says, reads
   the attributes, builds the object, runs `Init`, and spawns `Start`.
3. Unbind: when the tag goes or the instance is destroyed, `Destroy` runs
   and the `bin` empties, which also cancels a `Start` that still runs.

A failing `Init` or `Destroy` reports `TraitError:` with a traceback, and
the cleanup still runs. The same trait added to two Forges warns with
`DuplicateTrait:`. An attribute that changes to the wrong kind warns with
`BadAttribute:` and the field keeps its value.

`Forge.trait<<Door>>(part, Door)` gives the object a trait bound to an
instance, or nil. `Forge.traits<<Door>>(Door)` gives every object a trait
has bound.

Write an attribute through the instance: `self.instance:SetAttribute(...)`.
The field follows at the next `AttributeChanged`.

## Bin

A Bin holds connections, instances, threads and functions. `Empty`
disconnects, destroys, cancels and calls them, newest first, so teardown
unwinds setup. Forge gives one to every trait object.

```lua
import { Bin } from 'forge/src/index'

const bin = Bin.new()
const touched = bin:Add(part.Touched:Connect(on_touch))
bin:Add(Instance.new('Folder'))
bin:Add(task.spawn(loop))
bin:Add(function() -> ()
  print('emptied')
end)

bin:Empty()
```

- `Bin.new()`: an empty Bin.
- `bin:Add(item)`: adds an item and returns it, so an add can wrap an
  assignment. Anything else errors with `BadArgument:`.
- `bin:Empty()`: cleans every item, newest first. The Bin stays usable.

## Extensions

An extension is a table with a name and up to two hooks:

```lua
export type Extension = {
  Name: string,
  onPhase: ((forge: Forge, phase: Phase, entries: { Entry }) -> ())?,
  onProvider: ((forge: Forge, event: ProviderEvent, entry: Entry) -> ())?,
}
```

- `onPhase(forge, phase, entries)` fires at each `Phase`: `PreLoad`,
  `PostLoad`, `PreInit`, `PostInit`, `PreStart` and `PostStart`, with
  every entry loaded so far.
- `onProvider(forge, event, entry)` fires as each provider is `Loaded`,
  `Initialized` and `Started`.

The rules:

- At `PostLoad` an extension may call `forge:LoadProvider(module)`.
  Loading closes after `PostLoad`.
- At `PostLoad` or `PreInit` an extension may reorder `entries` in
  place. After `PreInit` the order is final.
- Up to `PreInit` an extension may call `forge:AddLifecycle(...)`.

An entry:

```lua
export type Entry = {
  Name: string,
  ModuleScript: ModuleScript,
  Provider: ProviderObject,
  Lifecycles: { string },
  Priority: number,
}
```

```lua
import { Forge, ProviderEvent, type Extension } from 'forge/src/index'

--- Prints each provider as it initializes.
const logger: Extension = {
  Name = 'Logger',
  onProvider = function(_forge, event, entry) -> ()
    if event == ProviderEvent.Initialized then
      print(`{entry.Name} initialized`)
    end
  end,
}
```

## Errors

Each error starts with its kind:

| Kind | When |
| --- | --- |
| `BadArgument:` | A call got a value of the wrong type or shape; a provider breaks its contract; a module exports no provider; a second instance of a provider struct; a listed method that no lifecycle fires. |
| `BadCall:` | A call at the wrong time: an add after `Start`, a second `Start`, `LoadProvider` outside `Load` and `PostLoad`, `AddLifecycle` after the wiring, a lifecycle for the other side. |
| `NoYield:` | An `Init`, or a trait's `Init` or `Destroy`, yielded. |
| `CyclicDependency:` | The uses form a cycle; the message has the path. |
| `MissingDependency:` | A use names a provider Forge never loaded. |
| `FrozenModule:` | Forge cannot put a dependency into a frozen provider. |

And the warnings: `LifecycleError:` (an Await handler failed),
`BadInstance:` (an instance fails a trait's contract), `BadAttribute:`
(an attribute changed to the wrong kind), `TraitError:` (a trait's
`Init` or `Destroy` failed), and `DuplicateTrait:`.

## API

### Forge

| Call | What it does |
| --- | --- |
| `new Forge(options?)` | A new Forge. `options.Verbose`, true when nil, prints the boot summary. |
| `forge:AddProvider(module)` | Adds a provider module. The same module twice does nothing. Chainable. |
| `forge:AddProviders(modules)` | Adds an array of provider modules, such as a folder's children. Chainable. |
| `forge:AddExtension(extension)` | Adds an extension. Chainable. |
| `forge:AddExtensions(extensions)` | Adds an array of extensions. Chainable. |
| `forge:AddLifecycle(lifecycle)` | Adds a custom lifecycle, up to `PreInit`. Chainable. |
| `forge:AddLifecycles(lifecycles)` | Adds an array of custom lifecycles. Chainable. |
| `forge:AddTrait(module)` | Adds a trait module, and requires it now. Chainable. |
| `forge:AddTraits(modules)` | Adds an array of trait modules. Chainable. |
| `forge:LoadProvider(module)` | Loads one provider now and gives its entry. For extensions, during `Load` and `PostLoad`. |
| `forge:Start()` | Runs the boot. Chainable. |
| `Forge.every(seconds, method)` | A lifecycle that ticks about every `seconds`. |
| `Forge.use(target)` | A dependency marker, by provider name or ModuleScript. |
| `Forge.trait<<T>>(instance, trait)` | The object a trait bound to an instance, or nil. |
| `Forge.traits<<T>>(trait)` | Every object a trait has bound. |

### Attributes

| Attribute | On | What it does |
| --- | --- | --- |
| `@Provider(lifecycles = {}, priority = 0)` | struct, impl | Marks a provider. |
| `@Trait(tag, instance = 'Instance', lifecycles = {})` | struct, impl | Marks a trait. |
| `@Attribute(name, kind)` | field | Fills a trait field from an attribute. |
| `@Child(name, class_name = 'Instance', wait = 0)` | field | Fills a trait field from a child. |

### Enums and types

| Name | What it is |
| --- | --- |
| `Lifecycle` | The built-in lifecycles, `Init` and `Start` among them. |
| `TraitLifecycle` | `Init`, `Start`, `AttributeChanged` and `Destroy`. |
| `Phase` | The phases an extension hears. |
| `ProviderEvent` | `Loaded`, `Initialized` and `Started`. |
| `Dispatch` | `Spawn`, `Inline` and `Await`. |
| `CustomLifecycle` | A lifecycle table. |
| `Extension` | An extension table. |
| `Entry` | One loaded provider. |
| `Options` | The options of `new Forge`. |
| `Bin`, `BinItem` | The cleanup container and what it holds. |

## Using Forge from a game

Forge is a project of its own. A game imports its source, so the compiler
reads `@Provider` and checks each contract in the game, and mounts its
build output where the require expects it.

The layout that works with Alloy 0.1.0-rc.8:

1. A link to Forge inside the game's root, so the build finds the
   dependency and maps its output to the mount:

   ```sh
   ln -s ../Forge forge
   ```

2. A mount for Forge's build output, and the type-only imports erased:

   ```toml
   [emit]
   erase_type_imports = true

   [mount]
   forge = ["forge/build", "@game/ReplicatedStorage/Forge"]
   ```

3. One module of the game that imports Forge, and that every other
   module imports from:

   ```lua
   --@alloy-ignore flux cannot follow an import into another project
   import { Provider } from '../../forge/src/index'
   import { Forge, Lifecycle } from '../../forge/build/index'

   export { Forge, Lifecycle, Provider }
   ```

   The attributes come from the source, so the compiler checks each
   contract. The values and types come from the build output, which
   `alloy flux` resolves. The next section says why.

`alloy build`, `alloy check` and `alloy flux` in the game then build Forge
first and resolve it, and the built require names
`@game/ReplicatedStorage/Forge/index`, where the mount puts it.

## Alloy issues Forge works around

Alloy 0.1.0-rc.8 gets these wrong. Each workaround is small, and each can
go when Alloy fixes the issue.

- An attribute on an `impl` is not written at runtime, so
  `Attributes.get(Struct, Provider)` gives nil. Put `@Provider` and
  `@Trait` on the struct; the contract reads the impl the same way.
- A keyed argument, `@Provider({ lifecycles = [...], priority = 10 })`,
  reaches the runtime as the first positional argument. The contract
  reads it by key. Forge reads both shapes.
- A `[]` default for a list parameter is written into the Luau as it is,
  and the module does not load. Forge's attributes default to `{}`.
- An import into another project from outside the root keeps its disk
  path in the ship artifact, and inside the root flux still cannot
  resolve it. Hence the link and the split import above.
- `Attributes.get` raises on a target that is no table. Forge checks the
  target first.

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests live beside the code, as `@test` functions and `@test`
namespaces. A `@test` namespace holds the fixture structs a test binds,
and the ship artifact leaves the whole group out. The tests cover the
priority sort, the dependency order and its cycles, the dispatch modes,
the Bin order, the contract, the phase order, the trait contracts, and
the errors.

## Inspired by Chief

Forge follows [Chief](https://github.com/ryancundiff/chief) by Ryan
Cundiff: its boot barrier, its extension hooks, its lifecycles and
dispatch modes, its dependency ordering, its traits and its Bin, and its
names, so a Chief user finds each call where they expect it. Forge moves
them to Alloy: a provider is a struct with an attribute, and the
compiler checks the lifecycles each one lists.
