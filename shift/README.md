# Shift

State machines over [Alloy](https://github.com/alloy-luau/alloy) enums.

The game writes its states as an enum and its events as another enum. The
transition is a function from a state and an event to the next state, written
as a `match`, so the compiler checks that each state has an arm. Shift runs the
machine around that function: enter and exit hooks, a `changed` signal, timers
that drop when their state leaves, and a history of the last states.

A machine runs on a clock of its own, which `update(dt)` moves. A game connects
it to `Heartbeat`, and a test moves it by hand, with no real time.

Shift depends on no other package. It needs Alloy 0.1.5 or later and embr 0.8.1
or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/shift/how_to) is a
  tour of Shift in numbered steps, from `001_states.aly` to `050_testing.aly`.
  Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/shift/examples)
  holds small games: a sliding door, a mob AI, and a round loop.

## Install

Add Shift to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
shift = { name = "alloy/shift", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

## Quick start

```aly
import { Machine } from '@packages/shift'

--- The states of a door. A payload holds data that only one state has.
export enum DoorState
  Closed
  Open
  Locked(string)
end

--- What can happen to a door.
export enum DoorEvent
  Push
  Lock(string)
  Unlock(string)
end

const door = new Machine<<DoorState, DoorEvent>>({
  initial = DoorState.Closed,
  transition = function(state, event)
    return match state, event with
      case Closed, Push then DoorState.Open
      case Open, Push then DoorState.Closed
      case Closed, Lock(key) then DoorState.Locked(key)
      case Locked(key), Unlock(given) where given == key then DoorState.Closed
      default nil
    end
  end,
  enter = function(state)
    match state with
      case Closed then print('shut')
      case Open then print('open')
      case Locked(key) then print(`locked with the {key} key`)
    end
  end,
})

-- `new` ran `enter` of Closed, so "shut" is out already.
door:send(DoorEvent.Push) -- open
door:send(DoorEvent.Push) -- shut
door:send(DoorEvent.Lock('red')) -- locked with the red key
door:send(DoorEvent.Push) -- no move: nothing happens
```

Write both types with `<<...>>`. Luau cannot infer the type of the enum from a
unit variant such as `DoorState.Closed`, and without them the call does not
type check.

## Transitions

The transition takes the state and the event, and returns the next state, or
nil for no change. With nil, Shift drops the event: no hook runs and `changed`
does not fire. `default nil` covers every pair that does nothing, so the match
lists only the moves.

Shift takes nil, not the same state, as "no change". A payload variant compares
by identity, so the machine could not tell a new `Locked('red')` from the old
one. Nil says it in one word.

Any state that the transition returns is a move, the same state too. The
machine leaves the state and enters it again, so its hooks run and its timers
start again. A mob that swings every second returns `Attack(target)` on each
`Swing`.

Keep the transition free of side effects. `can(event)` calls it to ask whether
an event moves the machine, and a button can grey out with the answer.

## Enter and exit

`enter` runs when the machine enters a state, the initial state too. `exit`
runs when it leaves one. Each hook gets the state and the machine. Write a hook
as a `match` with no `default`, and a state that the hook forgets does not
compile.

One move goes in this order: `exit` of the old state, its timers drop, `enter`
of the new state, then `changed`. A hook may call `machine:send(event)`. The
event waits until the move in progress ends, so `changed` fires in the order of
the moves.

An error in a hook stops that `send` with the error. The machine is in the new
state, and the next `send` runs as usual.

## Changed

`machine.changed` is a std `Signal`. It fires after each move with the old
state, the new state, and the event:

```aly
import { Machine } from '@packages/shift'

--- A light that a switch flips.
export enum Light
  Off
  On
end

--- The flip of the switch.
export enum Switch
  Flip
end

const light = new Machine<<Light, Switch>>({
  initial = Light.Off,
  transition = function(state)
    return match state with
      case Off then Light.On
      case On then Light.Off
    end
  end,
})

light.changed:Connect(function(from, to, event)
  print(`{from} -> {to} on {event}`)
end)

light:send(Switch.Flip) -- Off -> On on Flip
```

## Timers

`machine:after(seconds, event)` sends the event later, unless the machine
leaves the state first. Call it in `enter`. `machine:update(dt)` moves the
clock and fires each timer that falls due:

```aly
import RunService from '@game/RunService'
import { Machine } from '@packages/shift'

--- A trap that snaps shut, then opens again.
export enum Trap
  Armed
  Shut
end

--- What a trap hears.
export enum TrapEvent
  Step
  Reset
end

const trap = new Machine<<Trap, TrapEvent>>({
  initial = Trap.Armed,
  transition = function(state, event)
    return match state, event with
      case Armed, Step then Trap.Shut
      case Shut, Reset then Trap.Armed
      default nil
    end
  end,
  enter = function(state, machine)
    match state with
      case Armed then
      case Shut then machine:after(5, TrapEvent.Reset)
    end
  end,
})

RunService.Heartbeat:Connect(function(dt) trap:update(dt) end)
trap:send(TrapEvent.Step)
```

- A timer fires at its own time, not at the end of the frame. One long
  `update(10)` runs a chain of timed states in full, and each `after` counts
  from the time when its state began.
- A move out of a state drops every timer of that state.
- `machine:elapsed()` gives the seconds in the state now, for a countdown.
- A machine with no timers needs no `update`.

## Guards

A guard is a `where` on an arm of the transition. The arm moves only when the
condition holds. A guard can read the payloads and any value of the game that
the transition can see:

```aly
case Locked(key), Unlock(given) where given == key then DoorState.Closed
case Idle(at), Call(floor) where floor ~= at and powered then LiftState.Moving(floor)
```

A guarded arm proves nothing for the exhaustive check, so a transition with
guards keeps its `default nil`.

## History and debugging

A machine keeps its last states, 10 by default. `history = 0` in the options
keeps none. `machine:history()` gives them as a new array, oldest first, and
the last one is the state now:

```aly
print(door:history():join(' > ')) -- Closed > Open > Closed > DoorState.Locked("red")
```

A unit variant prints as its name. A payload variant prints as
`Enum.Variant(payload)`. A log of each move is one `changed` handler.

## Testing a machine

The clock moves only when code calls `update`, so a test sends events, moves
the clock by hand, and reads the state:

```aly
import { Machine } from '@packages/shift'

--- A kettle that boils in three seconds.
export enum Kettle
  Cold
  Heating
  Boiled
end

--- The switch, and the end of the heat.
export enum KettleEvent
  On
  Done
end

--- A new kettle.
export function kettle() -> Machine<Kettle, KettleEvent>
  return new Machine<<Kettle, KettleEvent>>({
    initial = Kettle.Cold,
    transition = function(state, event)
      return match state, event with
        case Cold, On then Kettle.Heating
        case Heating, Done then Kettle.Boiled
        default nil
      end
    end,
    enter = function(state, machine)
      if state == Kettle.Heating then
        machine:after(3, KettleEvent.Done)
      end
    end,
  })
end

@test
namespace KettleTests
  --- The kettle boils three seconds after the switch.
  public function it_boils_after_three_seconds() -> ()
    const k = kettle()
    k:send(KettleEvent.On)
    k:update(2.9)

    $assert_eq(k:state(), Kettle.Heating)

    k:update(0.1)

    $assert_eq(k:state(), Kettle.Boiled)
    $assert_eq(k:history():join(','), 'Cold,Heating,Boiled')
  end
end
```

## One machine per entity

A machine holds no thread and no connection of its own, so a game can make one
for each mob, door, or player. One `Heartbeat` loop, or one system of an ECS,
calls `update(dt)` on each. Code that runs on each frame matches
`machine:state()`; code that runs once on a change goes in `enter` or `exit`. A
machine that the game drops goes with the garbage.

## API

| Name | What it is |
| --- | --- |
| `new Machine<<S, E>>(options)` | A machine in `options.initial`. Runs `enter` of that state. |
| `options.initial` | The first state. |
| `options.transition(state, event)` | The next state, or nil for no change. |
| `options.enter(state, machine)` | Optional. Runs when the machine enters a state. |
| `options.exit(state, machine)` | Optional. Runs when the machine leaves a state. |
| `options.history` | Optional. How many states `history()` keeps. The default is 10. |
| `machine:state()` | The state now. |
| `machine:send(event)` | Moves the machine. From a hook, the event waits for the move in progress. |
| `machine:can(event)` | Whether the event moves the machine now. |
| `machine:after(seconds, event)` | Sends the event later, unless the state leaves first. |
| `machine:update(dt)` | Moves the clock and fires the timers that fall due. |
| `machine:elapsed()` | The seconds in the state now, on the clock of the machine. |
| `machine:history()` | The last states as an array, oldest first. |
| `machine.changed` | A `Signal` that fires `(from, to, event)` after each move. |
| `Options<S, E>`, `Hook<S, E>` | The types of the options and of a hook. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API.

Shift lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `shift` folder. Each
package there is a member of one Ember workspace and keeps its own version. Run
the commands above from `shift`. `embr publish` from `shift` publishes Shift alone.

The `how_to` and `examples` files import `@packages/shift`. To check them, copy
them into a game that installs Shift, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/shift/LICENSE).
