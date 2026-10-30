# Alloy packages

The official packages for [Alloy](https://github.com/alloy-luau/alloy).
Each one is a member of one [Ember](https://emberpm.dev) workspace, with
a version of its own.

| Package | Ember | What it is |
| --- | --- | --- |
| [Forge](forge) | `alloy/forge` | OOP providers, after Chief: lifecycles, dependencies, traits, and Bin. |
| [Bevel](bevel) | `alloy/bevel` | Components, services, injectors, and signals, after Angular. |
| [Gear](gear) | `alloy/gear` | State machines over Alloy enums, with transitions that a `match` checks. |
| [Ledger](ledger) | `alloy/ledger` | Typed player data on DataStores: session locks, migrations, autosave, and purchase receipts. |
| [Foundry](foundry) | `alloy/foundry` | Worker pools on Roblox actors, with typed jobs and a serial fallback. |
| [Anvil](anvil) | `alloy/anvil` | Console commands typed from their remote functions: arguments, permissions, and help. |

Each package folder holds:

- `src`: the source. `alloy build` writes `build` from it.
- `how_to`: a tour of the package in numbered steps.
- `examples`: small games that use the package.
- `README.md` and `CHANGELOG.md`.

## Install a package

Add it to the `ember.toml` of an Alloy game, then run `embr install`:

```toml
[dependencies]
forge = { name = "alloy/forge", version = "^0.1.1" }
```

The README of each package says how to mount it and import it.

## Work on a package

```sh
embr install         # each member installs its own dependencies
cd forge
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
embr publish         # publishes this member alone
```

`embr publish` at the root publishes every member, so run it from the
folder of the package that changed.

## License

MIT. See [LICENSE](LICENSE).
