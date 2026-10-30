# Changelog

## 0.1.0

The first release.

- `Machine`: a state machine over two enums of the game, one for the states
  and one for the events. The transition is a function that returns the next
  state, or nil for no change. A `match` in it lets the compiler check each
  state.
- Hooks: `enter` and `exit` for each state. A send from a hook waits until the
  move in progress ends.
- `changed`: a std `Signal` that fires with the old state, the new state, and
  the event.
- Timers: `after(seconds, event)` on a clock of the machine's own, which
  `update(dt)` moves. A move out of a state drops its timers. `elapsed()` gives
  the seconds in the state now.
- `can(event)` asks whether an event moves the machine. Guards are `where`
  clauses in the transition.
- `history()` keeps the last states, 10 by default.
- A `how_to` tour in numbered steps, and three example games: a sliding door, a
  mob AI, and a round loop.
