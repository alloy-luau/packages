# Changelog

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
