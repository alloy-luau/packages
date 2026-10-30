# Changelog

## 0.1.0

The first release.

- Stores: `new Store<<T>>(T, options)` keeps the data of a struct with
  `@derive(Serialize, Deserialize)`. The struct's defaults are the template.
- Migrations: a schema version, and one typed migration for each version
  before it. A load runs the chain in order. A save of a newer version does
  not load.
- Sessions: `join` and `open` load a record with a session lock in one
  `UpdateAsync`. Each save checks the lock, and a stale lock times out.
  `update`, `save`, and `close` give a `Result`.
- Saves: an autosave on an interval, a save on leave, and a save of every open
  session in `game:BindToClose`. Each DataStore call tries again with a wait
  that doubles, and waits for the request budget.
- Receipts: `store:receipts(grants)` grants each purchase once, by its
  `PurchaseId`, and answers `PurchaseGranted` only after the save.
- Backends: `DataStoreBackend`, and `MemoryBackend` for tests and Studio with no
  API access. The `Backend` trait takes a backend of your own.
- A `how_to` tour in numbered steps, and three example games.
