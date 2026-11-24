# Changelog

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
