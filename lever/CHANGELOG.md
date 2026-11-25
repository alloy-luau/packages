# Changelog

## 0.2.1

- `axis` and `axis2` read a pulse that comes and goes inside one frame, such
  as a notch of the mouse wheel. Before, they read zero, so an Axis bound to
  `MouseWheel` never moved. After the step, the axis reads the largest value
  of that frame for one frame. A Button tap reads 1 on `axis` the same way.

## 0.2.0

`new Lever` takes its two enums with the type `enum`. This is a breaking
change for a call that passes another value.

- The `actions` and `contexts` parameters of `new Lever` have the type
  `enum`. The old type was `unknown`. The type check now refuses a value
  that is no enum, such as a struct or a string.
- The source writes two guards on nil as `?? return` and `?? continue`.
- Lever needs Alloy 0.2.0-rc.7 or later. The type `enum` is behind the flag
  `AlloyExperimentalSupertypes`, and the guards are behind
  `AlloyExperimentalBailSyntax`. Lever sets the flags in its own
  `.config.aly`, so a game sets no flag.

The code that runs is the same as 0.1.1.

## 0.1.1

- The doc comments use the Moonwave format. The API tab on Ember shows
  each class, function, parameter, and return value.

The code is the same as 0.1.0.

## 0.1.0

The first release.

- Actions from a unit enum of the game. A spec gives each action its kind
  (`button`, `axis`, `axis2`), its context, and its default bindings. Lever
  builds them on the Input Action System of Roblox.
- Bindings for the keyboard and mouse, the gamepad, and touch, with the
  composites `wasd()` and `arrows()`.
- `began`, `ended`, `held`, and a frame `step` for `pressed` and `released`.
  `axis` and `axis2` read the axes.
- Contexts with a priority and a sink, which `enable` and `disable` turn on and
  off. A context that turns off ends its held actions. While a TextBox has the
  focus, no action begins, except in a context with `typing`.
- Rebinding at run time with `rebind`, `unbind`, `reset`, and `capture`. A key
  that another action of the context holds swaps with it.
- `save` and `load`: the player's changes as a table of names, which a
  DataStore keeps.
- The player's device from `PreferredInput`, `device_changed`, and `label`,
  `key_label`, and `image` for prompts.
- `touch` for GuiButtons and `fire` for a touch stick of the game's own.
- A `how_to` tour in nine steps, and three example games.
