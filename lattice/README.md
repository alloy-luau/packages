# Lattice

A typed ECS for [Alloy](https://github.com/alloy-luau/alloy), over
[jecs](https://github.com/Ukendio/jecs).

jecs keeps the entities and their components, and Lattice gives it the shape
of Alloy. A component is a typed id that holds a struct or a plain value. A
query names its components and yields their values with their types. A system
is a named function in a phase. Lattice has a small scheduler of its own that
runs each phase on its RunService event, so a game needs no Planck. A game adds
phases of its own from an enum that it declares.

Lattice sends the server's world to each client. A game marks a table of
components as shared, and the scheduler does the rest, over
[replecs](https://github.com/PepeElToro41/replecs). Each control of replecs
has a typed call in Lattice: the players that get an entity or one component,
the code that packs a value, and the hooks of a client.

A Lattice world is a jecs world. Each jecs method works on it, and jabby takes
it as it is.

Lattice needs Alloy 0.2.0-rc.7 or later and embr 0.9.1 or later. It depends on
jecs 0.11 and replecs 0.4, which Ember installs with it. Lattice turns on the
language flags that it uses for itself, so a game sets no flag.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/lattice/how_to) is
  a tour of Lattice in numbered steps, from `001_hello_world.aly` to
  `042_custom_replication.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/lattice/examples)
  holds small games: balls that fall and bounce, dummies that burn and heal,
  spawners whose coins have a lifetime, and orbs that the server moves and
  each client shows, with an orb for each player.

## Install

Add Lattice to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
lattice = { name = "alloy/lattice", version = "^0.3.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

A game that also uses jecs itself, for jabby, lists it with the same version,
`^0.11.0`. Ember then gives Lattice and the game one copy of jecs, and the ids
of one are the ids of the other.

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

The name is the jecs `Name` of the id. jabby shows it, and replication matches
the ids of the server and the client by it.

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
does not name runs in Simulation. `Phase` holds the built-in phases:

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

## Phases of the game

A game adds its own phases to a scheduler. It declares them in a unit enum, and
tells the scheduler where each one runs:

```aly
import RunService from '@game/RunService'
import { Phase, Scheduler, system, world, type World } from '@packages/lattice'

enum GamePhase
  Input
  Damage
  Cleanup
  Tick
end

--- Reads what the players did.
function input(_world: World, _dt: number) -> ()
  print('input')
end

const scheduler = new Scheduler(world())
  :before(Phase.PreSimulation, GamePhase.Input)
  :after(Phase.Simulation, GamePhase.Damage, GamePhase.Cleanup)
  :on(RunService.Heartbeat, GamePhase.Tick)

scheduler:add(system(input, GamePhase.Input)):start()
```

- `before(anchor, ...)` and `after(anchor, ...)` put phases beside a phase that
  the scheduler has, in the order of the call. They run on the event of the
  anchor. An anchor can be a phase of the game.
- `on(event, ...)` gives phases an event of their own. A system there gets the
  first argument of the event as `dt` when that is a number, and 0 when it is
  not.
- A phase beside `Startup` runs one time, when the scheduler starts. A phase
  beside `Render` runs on a client only.
- A phase is the name of its variant. Each phase of one scheduler has its own
  name, so a game enum does not use a name of `Phase`.
- Add each phase before `add` gets a system of that phase, and before `start`.
  A system whose phase the scheduler does not have stops `add` with an error.
- `step(dt)` runs the phases of the game in their places. The `on` phases come
  last.

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

Lattice sends the server's world to each client. The game marks a table of
components with `shared`, in a module that both sides import:

```aly
import { component, shared, tag } from '@packages/lattice'

export default shared({
  Position = component<<Vector3>>('Position'),
  Health = component<<number>>('Health'),
  Burning = tag('Burning'),
})
```

That is all the game writes. An entity that holds a shared id replicates, with
that id. The main script of each side starts its scheduler as before:

```aly
import { Scheduler, world } from '@packages/lattice'
import burn from './systems/burn'

new Scheduler(world()):add(burn):start()
```

`start` sees the shared ids and replicates the world. The server makes two
remotes in ReplicatedStorage, `LatticeReliable` and `LatticeUnreliable`. It
sends the full world to each client that joins, and sends the changes 20 times
a second. A client asks for the world and applies what arrives. The game
writes no system and no remote for this.

- Call `shared` before the first `world()`. Each shared id needs a name: the
  two sides match the ids by name.
- An entity replicates from its first shared id. A `set`, a `remove`, and a
  `delete` on the server reach each client.
- The entity of a client has its own number. A pair of a shared relationship
  arrives with the client's entity as its target, when the target replicates
  too.
- A client changes its own world only. Nothing goes back to the server.
- One world replicates on each side.

`shared(ids, share)` takes the channel as its second argument:

| Share | How the ids replicate |
| --- | --- |
| `Share.Reliable` | Each change arrives, in order. The default. |
| `Share.Unreliable` | The server sends the value at each interval, and a value can get lost. A tag and a pair replicate as Reliable. |
| `Share.Once` | The id arrives when an entity gets it, and goes when the entity loses it. A later value stays on the server until `resend`. |
| `Share.Manual` | Lattice replicates nothing by itself. The game names each entity and each id with `share`. |

```aly
import { Share, component, shared } from '@packages/lattice'

export default shared({ Aim = component<<Vector3>>('Aim') }, Share.Unreliable)
```

Roblox drops an unreliable packet of more than 1000 bytes. replecs splits the
Unreliable values into packets of 900 bytes at most, but it cannot count a
value that crosses beside the buffer. Give each Unreliable component a
`serdes`, as "Custom replication" shows.

### The replicator

`replicate(world, interval, unreliable_interval)` replicates a world and gives
its `Replicator`. `start` calls it for a game with shared ids, so a game calls
it for one of three reasons:

- For the `Replicator`. A later call for the same world gives the same one, so
  a system calls `replicate(world)` where it needs it.
- For other intervals. Call it before `start`. The first number is the seconds
  between two sends of the changes, and the default is 1/20. The second is the
  seconds between two sends of the Unreliable values, and its default is the
  first.
- For a world with no scheduler. Call it on each side.

```aly
import { Scheduler, replicate, world } from '@packages/lattice'

const w = world()

-- The changes go 10 times a second, and the Unreliable values 30 times.
const net = replicate(w, 1 / 10, 1 / 30)

new Scheduler(w):start()
print(net.world == w)
```

`net:stop()` ends the replication. The server destroys its remotes, and a
client applies no more. A later `replicate` starts again.

The methods of the server stop with an error on a client, and the methods of a
client stop with an error on the server.

## Players

Each player gets each entity that holds a shared id, with each of its shared
ids. The `Replicator` of the server changes that. `track(entity, to)` sets the
players that get an entity. `share(entity, id, to)` sets the players that get
one id of it:

```aly
import { Share, component, filter, replicate, shared, type World } from '@packages/lattice'

const ct = shared({ Position = component<<Vector3>>('Position') })
const secret = shared({ Loot = component<<number>>('Loot') }, Share.Manual)

--- The players that can read each chest.
const admins = filter()

--- Makes a chest that its owner gets, with loot that the owner and the
--- admins get.
export function chest(world: World, owner: Player) -> ()
  const net = replicate(world)
  const entity = world:entity()

  net:track(entity, owner)
  world:set(entity, ct.Position, Vector3.zero)
  world:set(entity, secret.Loot, 250)
  net:share(entity, secret.Loot, admins:extend(owner))
end

--- Lets a player read each chest, from the next send on.
export function promote(player: Player) -> ()
  admins:add(player)
end
```

`to` takes one of these:

| `to` | The players |
| --- | --- |
| nothing | Each player, for `track`. Each player of the entity, for `share`. |
| `player` | One player. |
| `{ [a] = true, [b] = true }` | The players of the set. |
| `{ [a] = false }` | Each player but the players of the set. |
| `function(player) return ... end` | The players for which the function gives true. The server calls it for each player at each send. |
| a `Filter` | The players of a group from `filter(initial?)`, which the game changes at run time. |

- A second `track` or `share` changes the players. A player that leaves the
  filter deletes the entity or removes the id. A player that comes in gets it.
- A player that joins later gets only what its filters allow, in its full
  state.
- A player gets an id only when it gets the entity.
- `track` before the first shared id of an entity means that no other player
  gets the entity, not for one send. A followed id leaves the players of the
  game as they are.
- `untrack(entity)` deletes the entity on each client, and the server keeps
  it. It holds until the next `track`, which sends the entity with each id.
  `unshare(entity, id)` does the same for one id.
- A `Share.Manual` id replicates only where `share` names it. The fourth
  argument of `share` is its channel: `Share.Reliable`, `Share.Unreliable`, or
  `Share.Once`.
- `share(entity, pair(Guards, gate))` replicates one pair, and
  `share_relation(entity, Guards)` each pair of a relationship.
  `unshare` and `unshare_relation` stop them.
- `resend(entity, id)` sends the value of a `Share.Once` id again.
  `send_now(entity)` sends one entity between two intervals.

A group is a `Filter`. `add`, `remove`, `set`, and `clear` change its players,
and each entity and each id that uses it follows at the next send.
`a:extend(b)` is the players of a or of b, `a:filter(b)` is the players of
both, and `a:exclude(b)` is the players of a that are not in b. Each of the
three makes a group that follows its parts. `members()` and
`contains(player)` read a group. `net:everyone()` is the group of each player,
and `net:predicate(test)` is a group from a function.

`net:alias(player, entity)` makes an entity stand for its player in a filter,
for a game that keeps an entity for each player. Call `net:unalias(entity)`
when the player leaves.

## Custom replication

Some components need code of the game when they replicate. Lattice has three
places for it, and each one keeps the types of the component.

The example is a game that keeps an entity for each player. Its `Owner`
component holds a `Player`. Its `Wallet` holds a struct, which loses its
methods on a remote. Its `Position` crosses in 12 bytes. The shared module
wraps each component where it makes it:

```aly
import { component, preprocess, serdes, shared } from '@packages/lattice'

--- What a player carries.
export struct Purse
  coins: number
end

impl Purse
  --- Whether the purse pays for a price.
  function affords(self, price: number) -> boolean
    return self.coins >= price
  end
end

export default shared({
  -- A Player crosses as it is, and arrives as the client's own Player.
  Owner = component<<Player>>('Owner'),
  Health = component<<number>>('Health'),

  -- The server sends the number, and each client makes a Purse again.
  Wallet = preprocess(component<<Purse>>('Wallet'), {
    serialize = function(purse: Purse) -> number
      return purse.coins
    end,
    deserialize = function(coins: number) -> Purse
      return new Purse { coins = coins }
    end,
  }),

  -- Three floats in the packet itself.
  Position = serdes(component<<Vector3>>('Position'), {
    bytespan = 12,
    serialize = function(position: Vector3) -> buffer
      const packed = buffer.create(12)
      buffer.writef32(packed, 0, position.X)
      buffer.writef32(packed, 4, position.Y)
      buffer.writef32(packed, 8, position.Z)
      return packed
    end,
    deserialize = function(packed: buffer) -> Vector3
      return new Vector3(
        buffer.readf32(packed, 0),
        buffer.readf32(packed, 4),
        buffer.readf32(packed, 8)
      )
    end,
  }),
})
```

The server makes the entity of each player. `identify` lets each client pick
its own entity for it, by the value of `Owner`:

```aly
import Players from '@game/Players'
import { Phase, replicate, system, type World } from '@packages/lattice'
import ct, { Purse } from '../components'

--- Gives each player an entity, on the server.
function avatars(world: World, _dt: number) -> ()
  const net = replicate(world)

  Players.PlayerAdded:Connect(function(player: Player) -> ()
    const avatar = world:entity()
    world:set(avatar, ct.Owner, player)
    world:set(avatar, ct.Health, 100)
    world:set(avatar, ct.Wallet, new Purse { coins = 5 })
    world:set(avatar, ct.Position, Vector3.zero)
    net:identify(avatar, ct.Owner)
  end)
end

export default system(avatars, Phase.Startup)
```

The client sets its hooks in a Startup system, which runs before the first
packet. `received` runs when a value arrives, with the value typed from the
component. Here it keeps a table of the entity of each player:

```aly
import Players from '@game/Players'
import { Phase, replicate, system, type Entity, type World } from '@packages/lattice'
import ct, { type Purse } from '../components'

-- The entity of each player, on this client.
local refs: { [Player]: Entity } = {}

--- Sets the hooks of the client.
function hooks(world: World, _dt: number) -> ()
  const net = replicate(world)

  -- This client made the entity of its own player already. `resolve`
  -- gives that entity for the server's entity with the same Owner, and
  -- the ids of the server go on it.
  const me = world:entity()
  refs[Players.LocalPlayer] = me

  net:resolve(ct.Owner, function(player) -> Entity?
    return player == Players.LocalPlayer ? me : world:entity()
  end)

  -- `player` is a Player: the value has the type of the component.
  net:received(ct.Owner, function(entity, _id, player) -> ()
    refs[player] = entity

    -- The other ids of the entity are there when the packet is done.
    net:after(function() -> ()
      print(player.Name, 'has', world:get(entity, ct.Health), 'health')
    end)
  end)

  -- For a struct, write the type of the value.
  net:received(ct.Wallet, function(_entity, _id, purse: Purse) -> ()
    print('can buy a sword:', purse:affords(10))
  end)

  -- `removed` runs before the id goes, so `get` still reads it.
  net:removed(ct.Owner, function(entity) -> ()
    if const player = world:get(entity, ct.Owner) then
      refs[player] = nil
    end
  end)
end

export default system(hooks, Phase.Startup)
```

### The code of a value

- `preprocess(id, { serialize, deserialize })` changes the value before it
  crosses, and back after. Use it for a value that cannot cross as it is.
- `serdes(id, { serialize, deserialize, bytespan?, includes_variants? })`
  writes the value into the buffer of the packet. `bytespan` is the size of the
  buffer when each value has the same size; replecs then sends no size. With
  `includes_variants`, `serialize` gives a list after the buffer, for the
  values that a buffer cannot hold, and `deserialize` gets that list.
- With both on one component, the preprocessor runs first on the server and
  last on a client.
- Wrap the component before the first `world()`. Both sides run the module, so
  both have the same code. The join compares the two sides, and a difference
  shows as a warning on the client.
- A relationship that holds values takes the code of its first id: wrap
  `Owes`, and each `pair(Owes, target)` uses it. A pair that `share` names by
  itself takes the code of `preprocess(pair(Owes, Wildcard), ...)`.
- A Player, or another Instance, crosses as a reference. It arrives as nil on a
  client that does not have the Instance.
- An entity in a value crosses as its number on the server. Use a pair for it:
  `pair(Targets, enemy)` arrives with the client's entity as its target. For a
  number that must cross, `net:client_entity(number)` gives the client's
  entity after that entity arrived.

### The hooks of a client

Each hook gives back a function that stops it.

| Hook | When it runs |
| --- | --- |
| `net:received(id, run)` | Each time a value of `id` arrives, after the world has it. For a relationship, `run` gets each pair. |
| `net:removed(id, run)` | When the server removes `id`, before the world loses it. |
| `net:deleted(entity, run)` | When the server deletes `entity`, before the client deletes it. |
| `net:created(run)` | When the client makes an entity for an entity of the server. The entity is empty then. |
| `net:after(run)` | One time, when the packet that applies now is done. |
| `net:override_received(id, run)` | In place of the write. The world gets the value only when `run` sets it. |
| `net:override_removed(id, run)` | In place of the removal. |
| `net:override_deleted(entity, run)` | In place of the delete. The client keeps the entity. |

A plain hook beside an override of the same id still runs. When the override
stops, the client writes again.

### The entities of the two sides

A client's entity has its own number. replecs keeps the two numbers of each
entity, and maps the target of each pair.

- `net:identify(entity, component)` on the server, with
  `net:resolve(component, find)` on a client, lets the client pick the entity.
  `find` gets the value of the component and gives an entity of the client, or
  nil to drop the entity of the server. The component must be a shared id that
  the entity holds and that replicates reliably.
- `custom_id(name, handle)` makes an id for the same job with more to read.
  `handle` gets a `Context`: `component(id)`, `has(tag)`, `target(relation)`,
  `pair_value(relation, target)`, `has_pair(relation, target)`, and
  `entity(number)`. Make it in a module of both sides, and give it to
  `identify`.
- `net:global(entity, number)` gives an entity a number from 0 to 245 that
  crosses as one byte in place of six. `net:globals(find)` maps the number on a
  client.
- `net:server_entity(entity)` and `net:client_entity(number)` read the map on a
  client. `net:bind(entity, number)` ties an entity of the client to an entity
  of the server before it arrives, and `net:unbind(entity)` ends the tie.

### When the code of the game fails

- A `deserialize` that raises shows a warning with its traceback. The entity
  keeps the value that it had, or stays without the id. Each other id of the
  packet arrives. The hooks of that id do not run for that value.
- A hook, a `resolve`, or the handler of a custom id that raises shows a
  warning. The value is in the world, the next hook runs, and the rest of the
  packet applies. A `resolve` that raises drops that entity.
- A `serialize` that raises stops that send of the server, and Roblox shows
  the error. The changes of that send can get lost. The server has all the
  data that the function needs, so fix the function.

## Lattice and replecs

Lattice adds no layer between the world and replecs. A game that marks its ids
with `shared` gets the same calls that a game on replecs writes by hand, and
the same bytes.

- An entity that gets a followed id runs one observer of jecs. For the first
  id of an entity, it calls `set_networked`. For each new id, it calls one
  `set_reliable`, `set_unreliable`, `set_once`, or `set_relation`. A later
  `set` of the value runs no code of Lattice.
- The server sends on Heartbeat, at the two intervals of `replicate`. A frame
  with no send costs two additions and two comparisons.
- A list of values with no value in it does not cross, so a game whose shared
  ids all have a `serdes` sends the buffer alone.
- `Share.Manual` turns the first rule off for a table of ids.
- An id with a `serdes` or a `preprocess` of Lattice pays for its guard on a
  client: one protected call for each value that arrives, and Lattice does the
  write. For the plain speed of replecs, set the code on the id in the world
  of each side before `start`: `world:set(ct.Position, replecs.serdes, codec)`.
  A `deserialize` that raises then stops the packet.

Each feature of replecs 0.4.0-rc.5, and the Lattice call for it:

| replecs | Lattice |
| --- | --- |
| `replecs.shared` on an id | `shared({ ... })` |
| `set_networked(entity, filter)`, `replecs.networked` | Follows a shared id by itself. `net:track(entity, to)` for a filter. |
| `stop_networked` | `net:untrack(entity)` |
| `set_reliable`, `pair(replecs.reliable, id)` | `Share.Reliable`, or `net:share(entity, id, to)` |
| `set_unreliable`, `pair(replecs.unreliable, id)` | `Share.Unreliable`, or `net:share(entity, id, to, Share.Unreliable)` |
| `set_once`, `pair(replecs.once, id)` | `Share.Once`, or `net:share(entity, id, to, Share.Once)` |
| `force_change` | `net:resend(entity, id)` |
| `set_relation`, `pair(replecs.relation, id)` | A shared relationship follows by itself. `net:share_relation(entity, relation, to)` by hand. |
| `set_pair`, `set_unreliable_pair` | `net:share(entity, pair(a, b), to, share)` |
| `stop_reliable`, `stop_unreliable`, `stop_once`, `stop_pair`, `stop_unreliable_pair` | `net:unshare(entity, id)` |
| `stop_relation` | `net:unshare_relation(entity, relation)` |
| A filter: a player, a set, a function | The `to` of `track`, `share`, and `share_relation` |
| `replecs.filter`, `server:filter` | `filter(initial?)` |
| `server:everyone`, `server:filter_predicator` | `net:everyone()`, `net:predicate(test)` |
| `add`, `remove`, `set`, `clear`, `filter`, `extend`, `exclude`, `members`, `contains` of a filter | The same methods, on a `Filter` |
| `add_player_alias`, `remove_player_alias` | `net:alias(player, entity)`, `net:unalias(entity)` |
| `mark_player_ready`, `get_full`, `apply_full` | Lattice runs the join. `net:is_ready(player)` and `net:is_synced()` read it. |
| `collect_updates`, `apply_updates` | Lattice sends at `interval` |
| `collect_unreliable`, `apply_unreliable`, the 900-byte split | Lattice sends at `unreliable_interval` |
| `collect_entity`, `apply_entity` | `net:send_now(entity)` |
| `generate_handshake`, `verify_handshake` | Lattice checks the client against the server at the join, and warns |
| `replecs.serdes`, `set_serdes` | `serdes(id, codec)` |
| `set_preprocessor` | `preprocess(id, preprocessor)` |
| `set_custom(entity, component)`, `pair(replecs.custom, id)` | `net:identify(entity, component)` |
| `replecs.custom_handler` | `net:resolve(component, find)` |
| `create_custom_id`, `register_custom_id`, `set_custom(entity, custom)` | `custom_id(name, handle)`, `net:identify(entity, custom)` |
| `remove_custom` | `net:unidentify(entity)` |
| `set_global`, `remove_global`, `handle_global` | `net:global(entity, number)`, `net:unglobal(entity)`, `net:globals(find)` |
| `hook('changed')`, `hook('removed')`, `hook('deleted')` | `net:received`, `net:removed`, `net:deleted` |
| `override(...)` | `net:override_received`, `net:override_removed`, `net:override_deleted` |
| `client:added` | `net:created(run)` |
| `after_replication` | `net:after(run)` |
| `get_server_entity`, `get_client_entity` | `net:server_entity(entity)`, `net:client_entity(number)` |
| `register_entity`, `unregister_entity` | `net:bind(entity, number)`, `net:unbind(entity)` |
| `encode_component`, `decode_component`, `get_shared_count` | `net:encode(id)`, `net:decode(number)`, `net:shared_count()` |
| `init`, `destroy` | `replicate(world)`, `net:stop()` |
| `MAX_CLIENTS` | Lattice sets it to `Players.MaxPlayers` when that is more than 128 |
| Entity ids, pair targets, bit masks for each player, changes only | replecs does this below Lattice, with no change |

What Lattice does not wrap:

- `set_delta_serializer`. replecs 0.4.0-rc.5 keeps the table and never calls
  it, so it changes nothing.
- `remove_serdes` and `remove_preprocessor`. The code of an id is set before
  the first world, for both sides at one time.
- `mark_player_ready` and `remove_player` by hand. Lattice marks a player when
  its client asks for the world, and replecs removes a player that leaves.
- `replecs.create`, the holder of both sides. `replicate` takes its place.

`net.server` and `net.client` are the server and the client of replecs, for a
call that this table does not have. `replecs` is the module, and its pair API
works on a Lattice world: `world:set(entity, replecs.networked, player)` is
`net:track(entity, player)`. In replecs 0.4.0-rc.5, a plain set of `false`
values through the pair API names the players that get the entity. `track` and
`share` read it as each player but those.

Lattice uses replecs 0.4.0-rc.5, the first line of replecs that takes jecs
0.11.

## jabby

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
| `scheduler:before(anchor, ...)`, `scheduler:after(anchor, ...)` | Adds phases of the game beside a phase, on its event. |
| `scheduler:on(event, ...)` | Adds phases of the game that run on an event of their own. |
| `scheduler:start()`, `scheduler:stop()` | Runs the phases on their events, or stops them. `start` replicates the world when the game has shared ids. |
| `scheduler:step(dt?)`, `scheduler:run(phase, dt?)` | Runs a frame, or one phase, by hand. |
| `shared(ids, share?)` | Marks a table of components and tags as shared, and gives it back. |
| `Share` | `Reliable`, `Unreliable`, `Once`, `Manual`. |
| `replicate(world, interval?, unreliable_interval?)` | Replicates a world, and gives its `Replicator`. The default interval is 1/20 of a second. |
| `serdes(id, codec)` | Gives a component its code for the buffer, and gives it back. |
| `preprocess(id, preprocessor)` | Gives a component code that changes its value for the wire, and gives it back. |
| `filter(initial?)` | A group of players that changes at run time. |
| `custom_id(name, handle)` | An id that lets a client pick its own entity. |
| `net:track(entity, to?)`, `net:untrack(entity)` | Sets the players of an entity, or stops its replication. Server. |
| `net:share(entity, id, to?, share?)`, `net:unshare(entity, id)` | Sets the players and the channel of one id, or stops it. Server. |
| `net:share_relation(entity, relation, to?)`, `net:unshare_relation(entity, relation)` | The same for each pair of a relationship. Server. |
| `net:resend(entity, id)`, `net:send_now(entity)` | Sends a Once value again, or one entity now. Server. |
| `net:everyone()`, `net:predicate(test)` | The group of each player, and a group from a function. Server. |
| `net:alias(player, entity)`, `net:unalias(entity)` | Makes an entity stand for a player in a filter. Server. |
| `net:identify(entity, by)`, `net:unidentify(entity)` | Lets a client pick its entity, by a component or a custom id. Server. |
| `net:global(entity, number)`, `net:unglobal(entity)` | Gives an entity a number that crosses as one byte. Server. |
| `net:is_ready(player)` | Whether a player got the full state. Server. |
| `net:received(id, run)`, `net:removed(id, run)`, `net:deleted(entity, run)` | The hooks of a client. Each gives a function that stops it. |
| `net:override_received`, `net:override_removed`, `net:override_deleted` | The same hooks, in place of the client's own change. |
| `net:created(run)`, `net:after(run)` | Runs for each new entity, and when the packet is done. |
| `net:resolve(id, find)`, `net:globals(find)` | Gives the client's entity for a component value, or for a global id. Client. |
| `net:server_entity(entity)`, `net:client_entity(number)` | Reads the map of the entities. Client. |
| `net:bind(entity, number)`, `net:unbind(entity)` | Ties an entity of the client to one of the server. Client. |
| `net:is_synced()` | Whether the full state arrived. Client. |
| `net:encode(id)`, `net:decode(number)`, `net:shared_count()` | The numbers of the shared ids on the wire. |
| `net:stop()` | Ends the replication of the world. |
| `net.world`, `net.server`, `net.client` | The world, and the server or the client of replecs. |
| `replecs` | The replecs module. |
| `ChildOf`, `Wildcard`, `Name` | jecs's built-in relationship, wildcard, and name component. |
| `Exclusive`, `OnDelete`, `OnDeleteTarget`, `Delete`, `Remove` | jecs's traits. |
| `World`, `Entity<T>`, `Query<T...>`, `System`, `Replicator`, `Filter`, `To`, `Serdes<T>`, `Preprocessor<T, A>`, `CustomId`, `Context` | The types. |

## Development

```sh
embr install         # jecs and replecs, into packages/
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

The tests of the replication run `replicate` itself, for both sides in one
process. A test gives the two remotes that `replicate` makes a fake transport,
and gives each player a world of its own.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/lattice/LICENSE).
