# Lattice

A typed ECS for [Alloy](https://github.com/alloy-luau/alloy), over
[jecs](https://github.com/Ukendio/jecs).

jecs keeps the entities and their components, and Lattice gives it the shape
of Alloy. A component is a typed id that holds a struct or a plain value. A
query names its components and yields their values with their types. A system
is a named function in a phase. Lattice has a small scheduler of its own that
runs each phase on its RunService event, so a game needs no Planck.

A Lattice world is a jecs world. Each jecs method works on it, and replecs and
jabby take it as it is.

Lattice needs Alloy 0.1.5 or later and embr 0.9.1 or later. It depends on jecs
0.11, which Ember installs with it.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/lattice/how_to) is
  a tour of Lattice in numbered steps, from `001_hello_world.aly` to
  `030_testing.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/lattice/examples)
  holds small games: balls that fall and bounce, dummies that burn and heal,
  and spawners whose coins have a lifetime.

## Install

Add Lattice to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
lattice = { name = "alloy/lattice", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

A game that also uses jecs itself, for jabby or replecs, lists it with the
same version, `^0.11.0`. Ember then gives Lattice and the game one copy of
jecs, and the ids of one are the ids of the other.

## Quick start

The components go in one table, in a module of their own:

```aly
import { component, tag } from '@packages/lattice'

export default {
  Position = component<<Vector3>>('Position'),
  Velocity = component<<Vector3>>('Velocity'),
  Frozen = tag('Frozen'),
}
```

A system is a named function in a module of its own, with its query at the
top and its export at the bottom:

```aly
import { Phase, query, system, type World } from '@packages/lattice'
import ct from '../components'

const moving = query(ct.Position, ct.Velocity):without(ct.Frozen)

--- Moves each entity by its velocity.
function movement(world: World, dt: number) -> ()
  for entity, position, velocity in moving:of(world) do
    world:set(entity, ct.Position, position + velocity * dt)
  end
end

export default system(movement, Phase.Simulation)
```

The main script makes the world and starts the scheduler:

```aly
import { Scheduler, world } from '@packages/lattice'
import ct from './components'
import movement from './systems/movement'

const w = world()
const rock = w:entity()
w:set(rock, ct.Position, Vector3.zero)
w:set(rock, ct.Velocity, new Vector3(0, 1, 0))

new Scheduler(w):add(movement):start()
```

## Components and tags

`component<<T>>(name)` makes an id that holds a value of type `T`: a number, a
Roblox datatype, an Instance, or an Alloy struct. `tag(name)` makes an id that
holds no value. A wrong value is a type error:

```aly
w:set(rock, ct.Position, 'up') -- error: Position holds a Vector3
```

The name is the jecs `Name` of the id. jabby shows it, and replecs needs it.

Make each component and each tag before the first `world()`. jecs gives each
world the ids that exist when it starts, so an id made later is not in that
world. `component` and `tag` stop with an error after the first world. A
components module that the game loads first keeps to this rule. jecs has room
for 256 components; a tag takes no room there.

A struct component comes back from `get` and from a query as the struct
itself, so a method call changes it in place. A change in place runs no
observer. Write it back with `set` to run them.

## Queries

`query(a, b, ...)` takes one to eight components and needs no world. A system
module makes its query once, at the top. `of(world)` gives the cached jecs
query of that world. Lattice builds it on the first call for each world and
keeps it. A test world and the game's world thus each have their own.

```aly
const moving = query(ct.Position, ct.Velocity):without(ct.Frozen)
const shown = query(ct.Position):with(ct.Visible)

for entity, position, velocity in moving:of(world) do
  -- position and velocity are Vector3s
end
```

- `with(...)` keeps the entities that also hold each id, and yields no value
  for them. `without(...)` drops the entities that hold any of them.
- Each call adds to the filters, in any order. jecs's own `with` replaces the
  last one. Call them where the query is made, before the first `of`.
- A query walks its entities from the last to the first. A `set` of a component
  that the entity holds, or a `delete` of the entity of that turn, is safe in
  the loop. Keep an `add` or a `remove` for after the loop: it moves the entity
  to another table of jecs.
- `moving:of(world):has(entity)` tells whether an entity matches.

## Relationships

A pair joins two ids into one. `pair(Likes, bob)` is the id "likes bob", and an
entity holds it as it holds a tag. A pair whose first id is a component holds a
value:

```aly
w:add(alice, pair(ct.Likes, bob))
w:set(alice, pair(ct.Owes, bank), 250)

for entity in query(pair(ct.Likes, Wildcard)):of(w) do
  print(entity, 'likes someone')
end
```

- `targets(world, entity, relation)` lists each target, and
  `world:target(entity, relation)` gives the first.
- `ChildOf` is built in: `w:add(door, pair(ChildOf, house))`. When the parent
  goes, its children go too. `world:parent(e)` and `world:children(e)` read it.
- A trait after the name of a component or a tag goes on the id in each world.
  `tag('MarriedTo', Exclusive)` keeps one target.
  `tag('OwnedBy', pair(OnDeleteTarget, Delete))` deletes each holder when its
  target goes.
- `Wildcard` works in a query and in `has`, not in `add` or `set`.

## Systems and phases

A system is a function `(world: World, dt: number) -> ()`. `system(fn, phase)`
names its phase and gives the function back, so a system module ends with
`export default system(movement, Phase.Simulation)`. A system that `system`
does not name runs in Simulation.

| Phase | When it runs |
| --- | --- |
| `Startup` | Once, before the first frame. `dt` is 0. |
| `PreSimulation` | On `RunService.PreSimulation`, before Roblox's physics. |
| `Simulation` | On `RunService.PostSimulation`, after the physics. |
| `PostSimulation` | On `RunService.PostSimulation`, after Simulation. |
| `Render` | On `RunService.PreRender`, on a client only. |

Put the work that writes to parts before the physics step in PreSimulation.
Put the work that reads what the physics did in Simulation or PostSimulation,
and the camera and the view in Render.

A system takes its world as an argument, so the same system runs on the game's
world and on the world of a test.

`system` keeps the phase in a table of Lattice's own. An attribute,
`@system(Phase.Simulation)`, would read better, but Alloy 0.1 keeps the
attributes of a function in the runtime of the module that declares it. A
package has its own runtime, so it cannot read them.

## The scheduler

`new Scheduler(world)` makes a scheduler for one world. `add(...)` puts each
system in its phase; inside a phase, the systems run in the order the scheduler
got them. A system that the scheduler holds keeps its first place, so a game
can add the systems that must run first, then a whole folder:

```aly
import { Scheduler, world, type System } from '@packages/lattice'

-- Load the systems before the world: their modules make components.
const systems: { System } = {}

for _, module in script.Parent->'systems'!:GetChildren() where module is ModuleScript do
  table.insert(systems, import<<System>>(module))
end

new Scheduler(world()):add(table.unpack(systems)):start()
```

- `start()` runs Startup once, then connects each phase to its event. A second
  `start` while it runs stops with an error.
- `stop()` ends the events. A later `start` goes on with no second Startup.
- In a scheduler that `start` runs, an error in a system shows as a warning
  with the system's name and its traceback. The systems after it still run.
- `step(dt)` runs one frame by hand: Startup on the first call, then
  PreSimulation, Simulation, PostSimulation, and Render. `run(phase, dt)` runs
  one phase. Both need no RunService, and an error goes to the caller.

## Observers

jecs gives three observers, and each one is typed by its component:

```aly
world:added(ct.Health, function(entity, _id, value) end)
world:changed(ct.Health, function(entity, _id, value) end)
world:removed(ct.Health, function(entity, _id, deleted) end)
```

Each call returns a function that stops the observer. A Startup system is the
place to start them, since it gets the world.

- `added` runs on the first `set` or `add`, and `changed` on each later `set`.
- `removed` runs before jecs takes the component away, so `get` still reads
  it. `deleted` is true when the whole entity goes.
- An observer runs in the middle of the change that started it. Keep it small:
  note the entity in a list, and let a system change the world.
- For a component that holds a table or a struct, write the type of the value:
  `function(entity, _id, health: Health)`. Luau does not find it from the
  component there.
- jecs's hooks, `world:set(ct.Health, OnAdd, fn)`, take one function of each
  kind for each component. Their type is generic, so an Alloy function with a
  typed value does not fit them. Use the observers.

## Testing

A test gives the system a new world, steps it by hand, and reads the result:

```aly
import { Phase, Scheduler, component, query, system, world, type World } from '@packages/lattice'

const ct = {
  Position = component<<Vector3>>('Position'),
  Velocity = component<<Vector3>>('Velocity'),
}

const moving = query(ct.Position, ct.Velocity)

--- Moves each entity by its velocity.
function movement(w: World, dt: number) -> ()
  for entity, position, velocity in moving:of(w) do
    w:set(entity, ct.Position, position + velocity * dt)
  end
end

system(movement, Phase.Simulation)

@test
namespace MovementTests
  --- An entity moves by its velocity times dt.
  public function movement_follows_the_velocity() -> ()
    const w = world()
    const e = w:entity()
    w:set(e, ct.Position, Vector3.zero)
    w:set(e, ct.Velocity, new Vector3(2, 0, 0))

    new Scheduler(w):add(movement):step(0.5)

    $assert_eq(w:get(e, ct.Position), new Vector3(1, 0, 0))
  end
end
```

`alloy test --run` runs it on lest. Make the components at the top of the
module, not in a test: the first `world()` closes the list of ids.

## Replication

Lattice sends nothing over the network.
[replecs](https://github.com/pepeeltoro41/replecs) (`pepeeltoro41/replecs` on
Wally) sends jecs entities from the server to each client, and it takes a
Lattice world as it is:

- Require replecs before the components module. It makes ids of its own, and
  those must come before the first world too.
- Mark each component to send with replecs's trait:
  `component<<Vector3>>('Position', replecs.shared)`. The name that Lattice
  gives the component is the jecs `Name` that replecs needs.
- In a Startup system, make the server with `replecs.create_server(world)` and
  call `init()`. In a PostSimulation system, send `collect_updates()` over the
  `remote` keyword or yetanothernet. The client does the same with
  `create_client`.

To see the world in jabby, register it with the world applet:
`jabby.register({ name = 'World', applet = jabby.applets.world, configuration = { world = w } })`.

## API

| Name | What it is |
| --- | --- |
| `world(debug?)` | A new jecs world that knows each id made before it. `true` gives jecs's debug world. |
| `component<<T>>(name?, ...traits)` | An id that holds a `T`. |
| `tag(name?, ...traits)` | An id that holds no value. |
| `pair(first, second)` | The id of a relationship and its target. |
| `targets(world, entity, relation)` | Each target of a relationship on an entity. |
| `query(a, ...)` | A `Query` over one to eight components. |
| `q:with(...)`, `q:without(...)` | Add filters to a query. |
| `q:of(world)` | The cached jecs query of one world. |
| `system(fn, phase)` | Names the phase of a system, and gives it back. |
| `Phase` | `Startup`, `PreSimulation`, `Simulation`, `PostSimulation`, `Render`. |
| `new Scheduler(world)` | A scheduler for one world. |
| `scheduler:add(...)` | Adds systems, each to its phase. |
| `scheduler:start()`, `scheduler:stop()` | Runs the phases on RunService, or stops them. |
| `scheduler:step(dt?)`, `scheduler:run(phase, dt?)` | Runs a frame, or one phase, by hand. |
| `ChildOf`, `Wildcard`, `Name` | jecs's built-in relationship, wildcard, and name component. |
| `Exclusive`, `OnDelete`, `OnDeleteTarget`, `Delete`, `Remove` | jecs's traits. |
| `World`, `Entity<T>`, `Query<T...>`, `System` | The types. |

## Development

```sh
embr install         # jecs, into packages/
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API.

Lattice lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `lattice` folder. Each
package there is a member of one Ember workspace and keeps its own version. Run
the commands above from `lattice`. `embr publish` from `lattice` publishes
Lattice alone.

The `how_to` and `examples` files import `@packages/lattice`. To check them,
copy them into a game that installs Lattice, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/lattice/LICENSE).
