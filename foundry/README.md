# Foundry

Worker pools on Roblox actors for
[Alloy](https://github.com/alloy-luau/alloy), with typed jobs.

A job is a function of its input. Foundry sends it to a pool of worker
actors, where it runs in the parallel phase. Each result goes back to the
script that asked. One `message` declaration types both ends: the input that
goes out and the result that comes back. When no worker can take a job, the
pool runs the same function here, serial. So each caller gets each result
once, from a worker or from the fallback.

Foundry depends on no other package.

Foundry needs Alloy 0.1.5 or later and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/foundry/how_to)
  is a tour of Foundry in numbered steps, from `001_a_job.aly` to
  `040_parallel_rules.client.actor.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/foundry/examples)
  holds small games: noise terrain in chunks, a batch of raycasts, and
  pathfinding for many agents.

## Install

Add Foundry to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
foundry = { name = "alloy/foundry", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

## Quick start

A job takes three files. A shared module holds the job and its message:

```aly
-- src/shared/noise.aly
import { job } from '@packages/foundry'

--- One square of the world: its corner, and the columns on each side.
export type Chunk = { x: number, z: number, size: number }

--- The height of each column of a chunk, row by row.
export function heights(chunk: Chunk) -> { number }
  local out: { number } = {}

  for row = 0, chunk.size - 1 do
    for column = 0, chunk.size - 1 do
      out[row * chunk.size + column + 1] = math.noise((chunk.x + column) / 64, (chunk.z + row) / 64)
    end
  end

  return out
end

--- Fills one chunk. The reply names the job and the seconds the worker
--- spent, then the heights, or the error.
@job
export message Fill(job: number, input: Chunk) reply(
  job: number,
  seconds: number,
  result: { number }?,
  problem: string?
) as parallel
```

The game writes the worker script. It binds the message to `serve`, then
marks itself ready:

```aly
-- src/client/worker.client.actor.aly
import { ready, serve } from '@packages/foundry'
import { Fill, heights } from '@shared/noise'

Fill.on(serve(heights))
ready(script)
```

A script beside it starts the pool and sends the jobs:

```aly
-- src/client/main.client.aly
import { Pool, device_workers } from '@packages/foundry'
import { Fill, heights } from '@shared/noise'

const pool = Pool.start(script.Parent->worker, device_workers(4, 2))
const fill = pool:channel(Fill, heights)

for x = 0, 7 do
  fill:run({ x = x * 16, z = 0, size = 16 }, function(tops)
    print(`chunk {x} has {#tops} columns`)
  end)
end
```

`fill:run` takes a `Chunk`, and `done` gets a `{ number }`. Both types come
from `Fill`, so a wrong input or a wrong result does not compile.

## Jobs and messages

A job message is an Alloy `message` with a `reply`, in one shape:

- The parameters are the job number and one input: `(job: number, input: I)`.
- The reply is the job number, the seconds that the worker spent, the result,
  and the error: `reply(job: number, seconds: number, result: T?, problem:
  string?)`.
- `as parallel` binds the handler with `BindToMessageParallel`, so the job runs
  in the parallel phase.

`@job` on the message makes `alloy check` ask for the job number and the
input. A message with no job number does not compile:

```text
error(3.11): AttributeContract: `@job` requires a field `job: number`; `Fill` declares none
```

The pool gives each job its number, and matches each answer to its job by that
number. Every `replied` handler of a message takes every answer of that
message, so two pools that share a message drop the answers of the other.

The input and the result cross between Luau VMs as a copy. A message carries
data: numbers, strings, buffers, Instances, and plain tables of them. A
struct, a function, and a `HashMap` do not arrive whole, and the compiler
reports each one at the message.

## The worker script

A package cannot ship the game's actor script. The script must sit where its
side runs scripts, and it binds the game's own messages. So the game writes
one, and it stays short:

- `.actor.` after the side in the file name puts the script in an Actor:
  `worker.client.actor.aly` builds as the Actor `worker` with a LocalScript in
  it, and `worker.server.actor.aly` as one with a Script.
- `Fill.on(serve(heights))` binds the handler. `serve(work)` runs `work` on the
  input, times it, and answers with the result. A job that throws answers with
  its error, and the worker goes on.
- `ready(script)` marks the Actor ready. The pool sends nothing to a worker
  before this call, so call it after the script binds each message.
- One worker binds as many job messages as its side sends.
- Require what the jobs need at the top of the script. A `require` cannot run
  in the parallel phase.

Each worker is a Luau VM of its own. A module that the worker requires runs
again in it, with state of its own.

## Where a worker runs

A worker runs only where its side runs scripts. Rojo builds the script of
`worker.server.actor.aly` as a Script with RunContext Legacy, and the script of
`worker.client.actor.aly` as a LocalScript. So the game must put each worker
Actor in a container that runs that class:

| Script | Runs in |
| --- | --- |
| Script, RunContext Legacy | ServerScriptService, Workspace |
| LocalScript | PlayerScripts, PlayerGui, Backpack, the character, ReplicatedFirst |

The usual mounts do this. `src/server` lands in ServerScriptService.
`src/client` lands in StarterPlayerScripts, and the engine copies it into the
PlayerScripts of each player, where the copy runs. A worker in `src/shared`
lands in ReplicatedStorage, where neither class runs. That worker never calls
`ready`, and the pool stops after its timeout.

The pool puts each clone beside the template, so each clone runs where the
template runs.

With `emitLegacyScripts = false` in a Rojo project, Rojo builds `.client.luau`
as a Script with RunContext Client. Such a script in StarterPlayerScripts runs
twice: as the template and as the copy. Keep the default.

The server has its own pool, and each client has its own. An actor message
never crosses the network.

## Pools

`Pool.start(template, count, options?)` clones the worker Actor `template`
into `count` workers beside it. The template is the first worker.

- A job goes to the ready worker with the fewest jobs out.
- `cap` is the most jobs that one worker holds at a time, 1 by default. Past
  the cap, a job waits in the queue of the pool, and it goes out when a worker
  answers. The queue is first in, first out, across the channels of a pool.
- A worker that is not ready takes nothing. The pool checks each frame, and a
  worker that turns ready takes the jobs that wait.

A worker runs all of its jobs in one parallel phase, and the frame waits for
the busiest worker. Raise the cap when the jobs are small, so that each worker
does more in each phase.

`device_workers(desktop, touch)` gives `touch` on a phone or a tablet and
`desktop` on any other device. A server has no touch screen, so it gets
`desktop`.

```aly
import { Pool, device_workers } from '@packages/foundry'

const pool = Pool.start(script.Parent->worker, device_workers(4, 2), { cap = 2, timeout = 5 })
print(pool:stats().workers)
```

`pool:channel(message, work)` gives the typed end of one job message. It binds
the replies of the message at once, before the first job goes out. A `fire`
before the first `replied` of its VM gets no answer. `channel:run(input,
done)` sends one job, and `done` gets its result in the serial phase, so it
may write Instances.

## Timeouts and the serial fallback

A job with no answer in `timeout` seconds counts as lost, 5 by default, and so
does a job that throws at its worker. A queue that waits that long while no
worker is ready stops the pool too. A lost job stops the pool:

- The pool warns once, with the reason.
- Each job out and each job in the queue runs serial at once, through the
  `work` of its channel.
- Each later `run` runs serial, and calls `done` before it returns.
- An answer that comes after the stop finds no job, and the pool drops it. So
  `done` never runs twice.

A stop is for good. `pool:stop(reason)` stops the pool from the game, as a
settings menu or a debug key may.

Keep `work` free of Instance writes. The serial path runs it in the serial
phase, where a write works, so such a bug hides until the job runs at a
worker.

## Stats

`pool:stats()` gives the numbers of the pool now, for a debug overlay:

| Field | What it counts |
| --- | --- |
| `workers` | The workers, ready or not. |
| `ready` | The workers that called `ready`. |
| `out` | The jobs out at a worker now. |
| `waiting` | The jobs that wait for a free worker. |
| `sent` | The jobs sent to a worker. |
| `answered` | The jobs that a worker answered with a result. |
| `lost` | The jobs that failed or got no answer in time. |
| `serial` | The jobs that ran here, serial. |
| `seconds` | The seconds that the workers spent in `work`. |
| `off` | Whether the pool stopped. |
| `reason` | Why the pool stopped. |

## Tests without actors

`Pool.start(nil, 0)` gives a pool that is off from the start. It warns once.
Each `run` then calls `done` at once, with no actors, so a test of game code
runs each job serial.

Foundry's own tests drive the pool with no actors. Each test builds a pool
with `new Pool { workers = ..., clock = ... }`: Actors from the test shim, a
clock of its own, and a fake message. A call to `pool:sweep()` reads the clock
again, as a frame does.

## API

| Name | What it is |
| --- | --- |
| `Pool.start(template, count, options?)` | Clones `template` into `count` workers and starts the pool. |
| `pool:channel(message, work)` | The typed end of one job message. Gives a `Channel`. |
| `channel:run(input, done)` | Sends one job. `done` gets the result once. |
| `pool:stats()` | The numbers of the pool now, as a `Stats`. |
| `pool:stop(reason)` | Stops the pool. Each job then runs serial. |
| `pool:sweep()` | Checks the timeouts and sends the queue. `start` calls it each frame. |
| `device_workers(desktop, touch)` | The workers to start on this device. |
| `serve(work)` | The handler of a job message, for the worker script. |
| `ready(script)` | Marks the Actor of a worker script ready. |
| `@job` | Marks a job message. Requires the fields `job: number` and `input`. |
| `Options` | `{ cap: number?, timeout: number? }`, for `Pool.start`. |
| `JobMessage<I, T>`, `Respond<T>` | The types of a job message and of its answer. |
| `Channel<I, T>`, `Stats`, `Pool` | The types of a channel, of the stats, and of a pool. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API.

Foundry lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `foundry` folder. Each
package there is a member of one Ember workspace and keeps its own version.
Run the commands above from `foundry`. `embr publish` from `foundry` publishes
Foundry alone.

The `how_to` and `examples` files import `@packages/foundry`. To check them,
copy them into a game that installs Foundry, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/foundry/LICENSE).
