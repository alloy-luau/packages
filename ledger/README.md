# Ledger

Typed player data on Roblox DataStores, for
[Alloy](https://github.com/alloy-luau/alloy).

The data is an Alloy struct, so the compiler checks its shape. A store loads
each player's record with a session lock, saves it on an interval, on leave,
and at shutdown, and runs the migrations of old saves on load. A receipt
handler grants each developer product purchase once. Every failure comes back
as a `Result`.

Ledger depends on no other package.

Ledger needs Alloy 0.1.5 or later and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/ledger/how_to) is a
  tour of Ledger in numbered steps, from `001_first_save.aly` to
  `040_testing_with_memory.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/ledger/examples)
  holds small games: coins that persist, saved settings, and a shop with
  receipts.

## Install

Add Ledger to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
ledger = { name = "alloy/ledger", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

## Quick start

A server script:

```aly
import Players from '@game/Players'
import { Deserialize, Serialize } from '@alloy/std/serde'
import { Store } from '@packages/ledger'

@derive(Serialize, Deserialize)
struct Save
  coins: number = 0
end

const store = new Store<<Save>>(Save, { name = 'Players', version = 1 })

Players.PlayerAdded:Connect(function(player)
  match store:join(player) with
    case Ok(session) then
      session:update(function(data) data.coins += 10 end)
    case Err(reason) then
      warn(reason)
      player:Kick('Your data did not load. Rejoin in a minute.')
  end
end)
```

- `new Store<<Save>>(Save, options)` makes a store of the struct `Save`.
  `<<Save>>` names the type, because Luau cannot read it from the class.
  `name` is the DataStore name.
- `store:join(player)` loads the record and takes its lock. It yields. The
  store closes the session when the player leaves: a last save, and the lock
  comes off.
- `session:update(fn)` changes the data in memory. The next save writes it.

## The template

The struct is the template. A new player gets the default of each field. An
old save that lacks a field gets its default too, because `from_table` fills in
each key the table lacks. So a field added in an update needs no migration.

`@derive(Serialize, Deserialize)` writes `to_table` and `from_table`. Ledger
uses them as the save format:

- Numbers, strings, and booleans save as they are.
- An array, a `{ [string]: T }` table, a `HashMap`, and a `Set` save as tables.
- A struct that also derives both nests as a table.
- A unit enum saves as the name of its variant. A name that no variant has does
  not load, so an old save never reads as another variant.
- `@skip` keeps a field out of the save, and `@rename('old')` keeps its key.

A DataStore keeps numbers, strings, booleans, and tables of them. Keep a
`Vector3`, a `Color3`, or an `Instance` as numbers in the struct. Keep the keys
of a map as strings: a DataStore cannot hold a table of sparse number keys.

`to_table` gives back the struct's own arrays and maps, and `from_table` takes
the arrays of its input. So each save copies the table before it yields, and a
change during the write goes into the next save.

## Migrations

A store has a schema version, and one migration for each version before it.
Migration 1 turns the data of version 1 into the data of version 2, and so on.
A load runs the chain from the version of the save up to the store's, in order.

```aly
import { Deserialize, Serialize } from '@alloy/std/serde'
import { Store } from '@packages/ledger'

@derive(Serialize, Deserialize)
struct Save
  coins: number = 0
  gems: number = 0
end

type SaveV1 = { gold: number? }

type SaveV2 = { coins: number?, premium: boolean? }

type SaveV3 = { coins: number?, gems: number? }

--- The players' saved data, at version 3.
export const store = new Store<<Save>>(Save, {
  name = 'Players',
  version = 3,
  migrations = {
    function(old: SaveV1) -> SaveV2
      return { coins = old.gold ?? 0, premium = false }
    end,
    function(old: SaveV2) -> SaveV3
      return { coins = old.coins, gems = old.premium ? 100 : 0 }
    end,
  },
})
```

- A migration works on the saved table. The struct of an old version is gone,
  so a `type` names each old shape. The checker reads each migration against
  the shapes it names.
- The checker does not link one migration's output to the next one's input.
  Luau rejects a generic alias that names itself with other type arguments, so
  a typed chain of changing shapes cannot exist.
- A migration must not yield. Never change one after it ships.
- A store whose count of migrations does not fit its version throws when it is
  made.
- A save of a newer version than the server knows does not load, and its record
  stays as it is. A migration that throws stops the load, and the lock comes
  off.

## Sessions and locks

One server at a time owns a record. The load takes a lock in the same
`UpdateAsync` that reads the record. The lock names the server (`game.JobId`),
the session, and the time of the last write. Each save checks the lock in its
own `UpdateAsync`, and writes nothing when another session holds it.

- A load that finds a live lock tries again, five times in all, with a wait
  that doubles from 1 second. Past the tries it is an Err.
- A lock that no write renewed for `lock_timeout` seconds (180 by default) is
  stale, and the next load takes it over. A crashed server leaves such a lock.
- A session that another server took is lost. Its saves and its updates are
  Errs, and it writes nothing again.
- An Err from a load means: do not play on this data. Kick the player, or let
  them play with nothing saved. Never save a new record over it.

`session:active()` says whether the session still takes changes.

## Autosave and shutdown

A save writes the data and renews the lock.

- The store saves each open session every `autosave` seconds (60 by default).
  Each session waits a random part of the interval first, so the saves of one
  server spread out. The autosave must be a third of `lock_timeout` or less.
- When a player leaves, the store closes their session.
- In `game:BindToClose`, the store closes every open session at once, and waits
  up to 25 seconds for the saves.
- `session:save()` saves now, after a change that is worth it, such as a trade.
- `session:close()` saves and takes the lock off, before a teleport.

Each DataStore call tries five times, with a wait that doubles from 1 second
and a random part. An error that no retry fixes, such as a value too large,
stops at once. A call waits up to 10 seconds for the server's request budget
(`DataStoreService:GetRequestBudgetForRequestType`) first.

## Receipts

`store:receipts(grants)` gives a handler for
`MarketplaceService.ProcessReceipt`:

```aly
import MarketplaceService from '@game/MarketplaceService'
import { Deserialize, Serialize } from '@alloy/std/serde'
import { Store } from '@packages/ledger'

@derive(Serialize, Deserialize)
struct Save
  coins: number = 0
end

const store = new Store<<Save>>(Save, { name = 'Players', version = 1 })

MarketplaceService.ProcessReceipt = store:receipts({
  [1001] = function(data: Save)
    data.coins += 500
  end,
})
```

The record keeps the `PurchaseId` of each purchase. The grant and the id go
into the data together, and one save writes both. The handler answers
`PurchaseGranted` only after that save. A receipt that comes twice grants once.

The handler answers `NotProcessedYet`, and Roblox sends the receipt again
later, when the product has no grant, when the player's session does not open
within 60 seconds, or when the save fails. The record keeps the newest 100
purchase ids. A grant must not yield.

`session:purchase(id, grant)` is the step under the handler. It gives `Ok(true)`
when the grant ran now, and `Ok(false)` when it ran before.

## Backends and tests

A store writes through a backend, and a backend has one call: `update`, which
runs a transform on one key in one step, as `UpdateAsync` does.

- `new DataStoreBackend(name, scope?)` is a Roblox DataStore. A store without a
  backend uses the DataStore of its name.
- `new MemoryBackend {}` keeps each record in a table in memory, so tests,
  Studio with no API access, and the examples run with no network.
  `backend:fail(n)` makes the next `n` calls fail, `backend:write(key, value)`
  puts a record in place, and `backend:read(key)` gives a copy of one.

```aly
import RunService from '@game/RunService'
import { Deserialize, Serialize } from '@alloy/std/serde'
import { MemoryBackend, Store } from '@packages/ledger'

@derive(Serialize, Deserialize)
struct Save
  coins: number = 0
end

--- The players' saved data. Studio keeps it in memory.
export const store = new Store<<Save>>(Save, {
  name = 'Players',
  version = 1,
  backend = RunService:IsStudio() ? new MemoryBackend {} : nil,
})
```

A backend of your own implements the `Backend` trait. Its `update` answers nil
when the call ran, and the error text when it failed; the store turns that into
a `Result`, and tries again:

```aly
import { Backend, type Transform } from '@packages/ledger'

--- Logs each call, then passes it on.
export struct LoggedBackend
  read inner: Backend
end

impl Backend for LoggedBackend
  function update(self, key: string, transform: Transform, user_ids: { number }?) -> string?
    print('update', key)

    return self.inner:update(key, transform, user_ids)
  end
end
```

## API

| Name | What it is |
| --- | --- |
| `new Store<<T>>(T, options)` | A store of the struct `T`. |
| `store:join(player)` | Opens `player_<UserId>`, and closes it when the player leaves. Yields. |
| `store:open(key, user_id?)` | Opens any key. Yields. |
| `store:get(player)` | The open session of a player, or nil. |
| `store:session(key)` | The open session under a key, or nil. |
| `store:receipts(grants)` | A handler for `ProcessReceipt`. |
| `store:close_all()` | Closes every open session. `BindToClose` calls it. |
| `session.data` | The data, a `T`. |
| `session:update(fn)` | Runs `fn` on the data. An Err after the session ends. |
| `session:save()` | Saves now. Yields. |
| `session:close()` | Saves and takes the lock off. Yields. |
| `session:purchase(id, grant)` | Grants once by `id`, then saves. Yields. |
| `session:has_receipt(id)` | Whether the record holds the purchase `id`. |
| `session:active()` | Whether the session still takes changes. |
| `MemoryBackend`, `DataStoreBackend` | The two backends. |
| `Backend` | The trait of a backend. |
| `Options`, `Migration`, `Grant<T>`, `Template<T>`, `Transform` | The types the API takes. |

The options:

| Option | Default | What it is |
| --- | --- | --- |
| `name` | | The DataStore name, and the name in each error. |
| `version` | | The schema version, from 1. |
| `migrations` | none | One migration for each version before `version`. |
| `backend` | the DataStore `name` | Where the records go. |
| `autosave` | 60 | Seconds between two autosaves. 0 turns it off. |
| `lock_timeout` | 180 | Seconds after its last write that a lock goes stale. |
| `attempts` | 5 | Tries of each backend call. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API, against `MemoryBackend`.

Ledger lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `ledger` folder. Each
package there is a member of one Ember workspace and keeps its own version. Run
the commands above from `ledger`. `embr publish` from `ledger` publishes Ledger
alone.

The `how_to` and `examples` files import `@packages/ledger`. To check them,
copy them into a game that installs Ledger, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/ledger/LICENSE).
