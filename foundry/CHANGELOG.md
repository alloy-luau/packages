# Changelog

## 0.1.1

- The tests write each callback of one expression as a lambda:
  `|n: number| n * 2`.
- Foundry needs Alloy 0.2.0-rc.7 or later. The lambda is behind the flag
  `AlloyExperimentalLambdaSyntax`. Foundry sets the flag in its own
  `.config.aly`, so a game sets no flag.

The API and the code that runs are the same as 0.1.0.

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
