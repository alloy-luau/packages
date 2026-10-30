# Changelog

## 0.1.0

The first release.

- Pools: `Pool.start` clones the worker Actor of the game into a count of
  workers. `device_workers` picks the count from the device.
- Channels: `pool:channel(message, work)` types both ends of a job message.
  `run` sends a job to the ready worker with the fewest jobs out, under a cap
  for each worker. A job past the cap waits in a queue.
- The serial fallback: a job past its timeout, or a job that fails, stops the
  pool. Each job then runs serial through the same `work`, so `done` gets each
  result once.
- Stats for a debug overlay: `pool:stats()`.
- The worker end: `serve(work)` and `ready(script)`.
- `@job`, a contract on a job message.
- A `how_to` tour in numbered steps, and three example games.
