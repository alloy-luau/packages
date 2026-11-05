# Changelog

## 0.2.0

Only the core installs by default now. Lifecycles, dependencies, traits,
and Bin are opt-in Ember file features. This is a breaking change.

### What moved behind which feature

| Feature | What moved | Turns on |
| --- | --- | --- |
| `lifecycles` | The built-in lifecycles past `Init` and `Start`, the wiring of custom lifecycles, `Forge.every`, and `forge:AddLifecycle(s)`. New: the free function `every`. | |
| `dependencies` | `Forge.use`, the Init order by the uses, and the type `Target`. New: the free function `use`. | |
| `traits` | `Trait`, `TraitLifecycle`, `Attribute`, `Child`, `forge:AddTrait(s)`, `Forge.trait`, and `Forge.traits`. | `bin` |
| `bin` | `Bin` and `BinItem`. | |

The core keeps `Forge`, `Provider`, `Lifecycle`, `Phase`,
`ProviderEvent`, `Dispatch`, and the types `Extension`, `Options`,
`Entry`, `ProviderObject`, `CustomLifecycle` and `Fire`. `Dispatch`,
`CustomLifecycle` and `Fire` moved from the lifecycles module to the
provider module, since the core names them in `forge:AddLifecycle`. The
import path `@packages/forge` stays the same.

### Other changes

- Each Forge method stays on `Forge`. The method of a feature that is
  off errors with the new kind `MissingFeature:`, and the message names
  the feature to turn on.
- With the `lifecycles` feature off, a provider that lists a lifecycle
  past `Init` and `Start` errors with `MissingFeature:` when Forge loads
  it.
- The features hear each phase before the game's extensions do. The
  order of the boot steps does not change.

### Migrate a game from 0.1.x

1. Turn on the features the game uses, in `ember.toml`:

   ```toml
   forge = { name = "alloy/forge", version = "^0.2.0", features = ["lifecycles", "dependencies", "traits"] }
   ```

   Or run `embr add alloy/forge --features lifecycles,dependencies,traits`.
   A game that used all of 0.1.x turns on these three. `traits` turns on
   `bin`. A game that uses `Bin` without traits turns on `bin`.
2. Run `embr install`.
3. Change no code. The names, the Forge methods and the import path are
   the same. A name of a feature that is off fails the compile, and a
   method of a feature that is off fails with `MissingFeature:`.

A game that mounts Forge from a path, with no Ember, writes one
`export ... from` line for each feature in the module that brings Forge
in. The README shows it.

### Ember versions

The split needs embr 1.0.0-rc.3 or later. An older embr ignores
`[feature-files]`: it installs every file, but its link file requires
the core only. Each feature then stays off.

## 0.1.2

- The doc comments use the Moonwave format. The API tab on Ember shows
  each class, function, parameter, and return value.

The code is the same as 0.1.1.

## 0.1.1

- The README reads well on the Ember site. Code blocks are `aly`, and
  each link is a full URL into
  [alloy-luau/packages](https://github.com/alloy-luau/packages).
- New `how_to` folder: a tour of Forge in numbered steps.
- New `examples` folder: a coin economy, and doors driven by a trait.
- The package names its repository, `alloy-luau/packages`.

The code is the same as 0.1.0.

## 0.1.0

- The first release: providers, lifecycles, dependencies, traits, Bin,
  and extensions.
