# Changelog

## 0.3.1

- The README tells when the queries of jecs fit, and when a game needs `query`
  and `of`. A second `with` or `without` of jecs replaces the first.

The code is the same as 0.3.0.

## 0.3.0

Lattice 0.2.0 did not go to Ember, so this is the first release with
replication. Each control of replecs now has a typed call in Lattice.

- `replicate(world, interval, unreliable_interval)` gives the `Replicator` of
  the world. A later call for the same world gives the same one. The
  Unreliable values have an interval of their own.
- Players: `net:track(entity, to)` sets the players that get an entity, and
  `net:share(entity, id, to, share)` sets the players and the channel of one
  id. `to` is a player, a set, a function, or a `Filter`. `net:untrack` and
  `net:unshare` stop the replication.
- `filter(initial)` makes a group of players that changes at run time, with
  `add`, `remove`, `set`, `clear`, `extend`, `filter`, and `exclude`.
  `net:everyone()`, `net:predicate(test)`, `net:alias`, and `net:unalias` come
  with it.
- `serdes(id, codec)` packs a component into the buffer of the packet, and
  `preprocess(id, preprocessor)` changes a value before it crosses and back
  after. Both wrap a component in the table of `shared`, and their functions
  get the type of the component.
- The hooks of a client: `net:received`, `net:removed`, `net:deleted`,
  `net:created`, and `net:after`, and the `override_` form of the first three.
- The entities of the two sides: `net:identify` with `net:resolve`,
  `custom_id`, `net:global` with `net:globals`, `net:server_entity`,
  `net:client_entity`, `net:bind`, and `net:unbind`.
- `Share.Once` sends an id one time, and `net:resend` sends it again.
  `net:share_relation` and `net:send_now` are new too.
- `net:stop()` ends the replication, and a later `replicate` starts again.
- A `deserialize` or a hook that raises shows a warning. The entity keeps its
  value, and the rest of the packet applies.
- An entity that the game gave a filter keeps it. In 0.2.0, each new shared id
  made replecs work the players out again.
- A pair of a shared relationship replicates through one `set_relation` for
  its entity, in place of one call for each target.
- A list of values with no value in it does not cross.
- A set of `false` values names the players that do not get an entity. replecs
  0.4.0-rc.5 reads a plain set as the players that get it.
- The server has room for `Players.MaxPlayers` players. replecs stops at 128.
- A player that left gets no answer to a late request.
- Two new `how_to` steps, `041_players.aly` and `042_custom_replication.aly`.
  The `replication` example gives each player an orb of its own.
- Lattice needs Alloy 0.2.0-rc.7. It turns on the lambda and the bail flags
  for itself, and a game sets no flag.

A game on 0.1 needs no change. Against 0.2.0, `replicate` gives the
`Replicator` back and takes a third argument, and `Share` has `Once`.

## 0.2.0

- Phases of the game: `scheduler:before` and `scheduler:after` put phases
  beside a phase that the scheduler has, on the same event, and
  `scheduler:on` gives phases an event of their own. A phase is the name of
  an enum variant, so a unit enum of the game names its phases.
- `system` and `scheduler:run` take a phase of the game. `scheduler:add`
  stops with an error for a system whose phase the scheduler does not have.
- Replication: `shared` marks a table of components and tags. An entity that
  holds a shared id replicates to each client, with that id.
- `scheduler:start` replicates the world when the game has shared ids. It
  makes the remotes, sends the full world to a client that joins, and sends
  the changes after that. `replicate` does the same by hand.
- `Share` picks the channel of the shared ids: `Reliable`, `Unreliable`, or
  `Manual`. `replecs` is the replecs module, for a filter of one entity or
  one player.
- Two new `how_to` steps, `012_phases.aly` and `040_replication.aly`, and a
  `replication` example game.
- Lattice depends on replecs 0.4.0-rc.5.

A game on 0.1 needs no change.

## 0.1.1

- The doc comments use the Moonwave format. The API tab on Ember shows
  each class, function, parameter, and return value.

The code is the same as 0.1.0.

## 0.1.0

The first release.

- The world: `world`, a jecs world that knows each id made before it.
- Ids: `component<<T>>` for a typed value, such as an Alloy struct, and `tag`
  for no value. Each one takes a name and jecs traits. An id made after the
  first world stops with an error.
- Relationships: `pair`, `targets`, and jecs's `ChildOf`, `Wildcard`, and
  traits.
- Queries: `query` over one to eight components, with `with` and `without`.
  `of(world)` gives a cached jecs query for each world.
- Systems: named functions in five phases, from Startup to Render.
  `system(fn, phase)` names the phase.
- The scheduler: `Scheduler` runs each phase on its RunService event, and
  `step` and `run` run them by hand for tests. An error in a running system
  shows as a warning and stops no other system.
- A `how_to` tour in eight numbered steps, and three example games.
- Lattice depends on jecs 0.11.
