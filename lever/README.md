# Lever

Input actions for Roblox games, written in
[Alloy](https://github.com/alloy-luau/alloy).

A game declares its actions as a unit enum, such as an `Action` with `Jump`,
`Attack`, and `OpenInventory`. A spec gives each action its kind, its context,
and its keys for the keyboard and mouse, the gamepad, and touch. Write the spec
as a `match` with no default arm, and a new action with no keys fails the
build. Lever fires `began` and `ended`, keeps the state of each action, and
holds the bindings as data for a rebind screen and a save.

Lever builds on the Input Action System of Roblox: one `InputContext` for each
context, one `InputAction` for each action, and one `InputBinding` for each
binding. The engine reads the hardware and applies the priority and the sink of
each context. Lever depends on no other package.

Lever needs Alloy 0.1.5 or later and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/lever/how_to) is a
  tour of Lever in numbered steps, from `001_hello_world.aly` to
  `040_prompts.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/lever/examples)
  holds small games: the controls of a platformer, a pause menu that a gamepad
  drives, and a rebind screen that saves on the server.

## Install

Add Lever to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
lever = { name = "alloy/lever", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

## Quick start

A module declares the actions, the contexts, and their specs:

```aly
import {
  Lever,
  axis2,
  button,
  wasd,
  type ActionSpec,
  type ContextSpec,
} from '@packages/lever'

--- What the player can do.
export enum Action
  Move
  Jump
  Interact
end

--- When the actions apply.
export enum Context
  Gameplay
end

--- The keys of each action. A new action fails the build until it has
--- an arm here.
local function action(each: Action) -> ActionSpec<Context>
  return match each with
    case Move then axis2(Context.Gameplay, { wasd(), Enum.KeyCode.Thumbstick1 })
    case Jump then button(Context.Gameplay, { Enum.KeyCode.Space, Enum.KeyCode.ButtonA })
    case Interact then button(Context.Gameplay, { Enum.KeyCode.E, Enum.KeyCode.ButtonX })
  end
end

--- The one context of the game.
local function context(_each: Context) -> ContextSpec
  return { priority = 0 }
end

--- The controls of the game.
export const controls = new Lever(Action, Context, action, context)
```

A client script reads them:

```aly
import RunService from '@game/RunService'
import { Action, controls } from './Controls'

controls.began:connect(function(each: Action)
  match each with
    case Jump then print('jump')
    case Interact then print(`interact on {controls:label(Action.Interact)}`)
    case Move then return
  end
end)

RunService.Heartbeat:Connect(function()
  controls:step()

  const move = controls:axis2(Action.Move)

  if move.Magnitude > 0 then
    print(`move {move}`)
  end
end)
```

Lever runs on the client. `new Lever(actions, contexts, action, context)` takes
the two enums themselves, then their specs, and reads the variants of each
enum. A fifth argument names the parent of Lever's folder, which is
`ReplicatedStorage` when it is nil.

## Actions and bindings

`button`, `axis`, and `axis2` make the spec of an action:

| Spec | Kind | Value | Example inputs |
| --- | --- | --- | --- |
| `button(context, bindings, name?)` | `Button` | `boolean` | a key, a click, a gamepad button |
| `axis(context, bindings, name?)` | `Axis` | `number` | a trigger, the mouse wheel, `{ up = E, down = Q }` |
| `axis2(context, bindings, name?)` | `Axis2` | `Vector2` | a stick, `wasd()`, `arrows()` |

A binding is a table of keys by part: `{ key = ... }` for one input, or
`{ up = ..., down = ..., left = ..., right = ... }` for a composite. A bare
`Enum.KeyCode` is a binding of one key. Lever sorts each binding by the device
of its keys:

- `Device.Keyboard`: the keys of the keyboard and the buttons of the mouse.
- `Device.Gamepad`: the buttons, the D-pad, the triggers, and the sticks.
- `Device.Touch`: touch gestures such as `TouchDelta`, and the touch buttons
  that `touch` links.

Two bindings of one device fill two slots of that device. `name` is the text
that a controls screen shows; it is the variant with spaces when nil, so
`OpenInventory` shows as "Open Inventory".

## Began, ended, and held

- `controls.began` fires with the action when it begins: a key goes down, or an
  axis leaves zero.
- `controls.ended` fires when it ends: a key comes up, an axis comes back to
  zero, or its context turns off.
- `controls:held(action)` is true while the action is down, or off zero.
- `controls:step()` latches one frame. Call it once a frame, at the top of the
  frame's code. Then `controls:pressed(action)` and `controls:released(action)`
  are true when the action began or ended in that frame. A tap that begins and
  ends inside one frame counts as both.
- `controls:axis(action)` reads an Axis, and a Button as 1 or 0.
  `controls:axis2(action)` reads an Axis2.

The signals are std `Signal`s, so `connect` gives a connection that
`Disconnect` ends.

## Contexts

A context is a group of actions that turns on and off as one. Its spec has a
`priority`, and a context with a higher priority hears input first. With
`sink = true`, the keys that a context binds do not reach the contexts below
it. `enabled = false` starts it off.

```aly
local function context(each: Context) -> ContextSpec
  return match each with
    case Gameplay then { priority = 0 }
    case Menu then { priority = 100, sink = true, enabled = false }
    case Chat then { priority = 200, sink = true, enabled = false, typing = true }
  end
end
```

- `controls:enable(context)` and `controls:disable(context)` turn it on and
  off, and `controls:enabled(context)` reads it.
- `disable` ends each held action of the context and fires `ended` for it. A
  run that a menu cuts off does not go on behind the menu.
- While a TextBox has the focus, no action begins, so a player who types in the
  chat does not jump. An action can still end. `typing = true` lets one context
  take input then, such as the keys of a chat box of the game's own.
- Two contexts that are never on together can share a key.

## Rebinding

```aly
const other = controls:rebind(Action.Jump, Device.Keyboard, Enum.KeyCode.E)
controls:rebind(Action.Move, Device.Keyboard, { up = Enum.KeyCode.Z })
controls:rebind(Action.Jump, Device.Keyboard, Enum.KeyCode.Up, 2)
controls:unbind(Action.Jump, Device.Keyboard, 2)
controls:reset(Action.Move)
```

- `rebind(action, device, change, slot?)` changes the parts that `change`
  holds and keeps the others. `slot` is 1 when nil, and one past the last slot
  adds a binding. The engine reads the new key at once.
- A key that another action of the same context holds swaps: that action takes
  the old key. `rebind` gives that action, or nil.
- A rebind refuses a key of another device, and a key that Roblox keeps:
  Escape, Slash, F9 to F12, the Windows keys, and the gamepad's Menu and View
  buttons.
- `capture(device, done)` waits for the next key of a device, for a rebind
  screen. `done` gets the key, or nil after a press of a reserved key such as
  Escape. It gives a function that stops the wait. Turn the gameplay context
  off while a rebind screen shows, because the key also reaches the game.
- `controls.rebound` fires for each action whose bindings change.

## Saving

`controls:save()` gives the bindings that the player changed:

```text
{ version = 1, actions = { Jump = { Keyboard = { { key = "E" } } } } }
```

The table holds names, so a DataStore or JSON keeps it as it is.
`controls:load(save)` gives every action its defaults, then the bindings of the
save. An action, a device, or a key name that Lever does not know drops out, so
a save from an older version of the game loads. A device that the player did
not change is not in the save, so a new default reaches each player who kept
the old one.

DataStores work on the server only. Send the save to the server with a remote,
and keep it with the player's data. The `rebind` example does that.

## Devices and prompts

- `controls:device()` is the device that the player uses now, from
  `UserInputService.PreferredInput`. `controls.device_changed` fires when it
  changes.
- `controls:label(action, device?, slot?)` names a binding for a prompt: "E",
  "Space", "Left Click", "A", or "W/A/S/D". With no device it reads the
  player's device, and it gives `''` when there is no binding. Keyboard keys
  follow the player's layout, so KeyCode.W shows as Z on an AZERTY keyboard.
  Gamepad buttons follow the brand of the gamepad, so ButtonA shows as Cross on
  a PlayStation gamepad.
- `key_label(key)` names one key the same way.
- `controls:image(action, device?, slot?)` gives the image that Roblox has for
  the key, such as the glyph of a gamepad button.
- `controls:instance(action)` gives the `InputAction`, for an
  `InputActionLabel`.

```aly
prompt.Text = `Press {controls:label(Action.Interact)} to open`
```

## Touch

- `controls:touch(action, button)` links a GuiButton of the game's own to a
  Button action. The engine holds the action while a finger holds the button.
- `controls:fire(action, value)` drives an action through a Scriptable
  `InputBinding`: a touch stick of the game's own, or a tutorial that presses a
  key for the player. The value fits the kind: a boolean, a number, or a
  `Vector2`.

## Tests

The logic of Lever runs without real input. Under `alloy test`, the test shim
gives the instances, and a test fires the `StateChanged` signal of an
`InputAction` as the engine does:

```aly
type Fired = { Fire: (self: unknown, value: unknown) -> () }

const changed = (controls:instance(Action.Jump).StateChanged :: unknown) :: Fired
changed:Fire(true)
$assert(controls:held(Action.Jump))
```

## API

| Name | What it is |
| --- | --- |
| `new Lever(actions, contexts, action, context, parent?)` | The controls of a game. |
| `button`, `axis`, `axis2` | The spec of an action of each kind. |
| `wasd()`, `arrows()` | The two usual composites of four keys. |
| `capture(device, done)` | Waits for the next key of a device. |
| `key_label(key)` | The text of one key for this player. |
| `lever.began`, `lever.ended` | Fire with the action as it begins and ends. |
| `lever.rebound` | Fires with an action whose bindings change. |
| `lever.device_changed` | Fires with the player's new device. |
| `lever:step()` | Latches `pressed` and `released` for one frame. |
| `lever:held(a)`, `lever:pressed(a)`, `lever:released(a)` | The state of an action. |
| `lever:axis(a)`, `lever:axis2(a)` | The value of an axis. |
| `lever:enable(c)`, `lever:disable(c)`, `lever:enabled(c)` | Turn a context on and off. |
| `lever:bindings(a, device)` | A copy of an action's bindings for a device. |
| `lever:rebind(a, device, change, slot?)` | Binds new keys, and swaps a clash. |
| `lever:unbind(a, device, slot?)`, `lever:reset(a?)` | Remove a binding, or go back to the defaults. |
| `lever:save()`, `lever:load(save)` | The player's changes as data, and back. |
| `lever:device()` | The device of the player now. |
| `lever:label(a, device?, slot?)`, `lever:image(a, device?, slot?)` | The text and the image of a binding. |
| `lever:name(a)`, `lever:actions()`, `lever:instance(a)` | An action's name, every action, and its `InputAction`. |
| `lever:touch(a, button?)`, `lever:fire(a, value)` | Touch buttons, and input from code. |
| `lever:destroy()` | Takes the instances and the connections down. |
| `Kind`, `Device` | The kinds of actions, and the devices. |
| `ActionSpec<C>`, `ContextSpec`, `Binding`, `Input`, `Part`, `Save`, `SavedBinding`, `Value` | The types. |

## Why the Input Action System

Roblox has three input APIs: UserInputService, ContextActionService, and the
Input Action System. Lever builds on the Input Action System for these reasons:

- The engine applies the priority and the sink of each context, and ends the
  held keys of a context that turns off.
- A binding holds a composite direction, a stick, the threshold of a trigger,
  or a GuiButton for touch. Lever writes the binding, and the engine reads the
  hardware.
- `InputBinding.KeyCode` and the other parts are writable at run time, so a
  rebind is a property write.
- Roblox builds its newer input features on it. Under the server authority
  model (`Workspace.AuthorityMode = Server`), the input that drives the
  simulation must come from the Input Action System, not from
  UserInputService.
- `InputActionLabel` and `PreferredBinding` read the same instances.

Lever reads UserInputService for three things that the Input Action System
does not give: the player's device, the focus of TextBoxes, and the next key
for `capture`.

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API.

Lever lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `lever` folder. Each
package there is a member of one Ember workspace and keeps its own version. Run
the commands above from `lever`. `embr publish` from `lever` publishes Lever
alone.

The `how_to` and `examples` files import `@packages/lever`. To check them, copy
them into a game that installs Lever, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/lever/LICENSE).
