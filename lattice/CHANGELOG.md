# Changelog

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
