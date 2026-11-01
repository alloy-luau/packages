# Decree

Typed console commands for Roblox games, in
[Alloy](https://github.com/alloy-luau/alloy).

A command is a `remote function` marked `@command`. Decree reads its
declaration: the name of each parameter, its type, and its default. From
that it parses each line, writes the help and the usage, and gives the handler
each argument as its declared type. The type check holds the handler to the
declaration.

The server is the authority. A client sends a line as text, and the server
parses it, checks the player's level, runs the command, and answers. Decree
ships no UI: a console of your own, or one such as Conch, sends the lines.

Decree depends on no other package.

Decree needs Alloy 0.1.5 or later and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/decree/how_to) is a
  tour of Decree in numbered steps, from `001_first_command.aly` to
  `030_help.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/decree/examples)
  holds small games: admin commands with kick, ban, and unban, developer spawn
  commands, and teleport commands.

## Install

Add Decree to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
decree = { name = "alloy/decree", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

## Quick start

A server script declares a command, gives it a handler, and starts Decree:

```aly
import { command, fail, on, serve } from '@packages/decree'

--- The level of a moderator.
const MODERATOR = 1

--- The user ids of the moderators.
const MODERATORS: { [number]: boolean } = { [156] = true }

--- Kicks a player out of the server.
@command(MODERATOR, 'Kicks a player out of the server.')
remote function Kick(target: Player, reason: string = 'You were kicked.') -> string from client

on(Kick)(function(caller, target, reason)
  if target == caller then
    return fail('You cannot kick yourself.')
  end

  target:Kick(reason)

  return `Kicked {target.Name}.`
end)

-- The resolver gives each player a level: a moderator 1, anyone else 0.
serve(function(player)
  return MODERATORS[player.UserId] ? MODERATOR : 0
end)
```

A client script sends each line and shows the answer:

```aly
import { run } from '@packages/decree'

async do
  const reply = await run('kick builderman griefing')
  print(reply.ok, reply.text)
end
```

A moderator who writes `kick builderman griefing` reads `Kicked Builderman.`
The handler gets `target` as a `Player` and `reason` as a `string`. The last
string takes the rest of the line.

## Why a remote

An attribute on a plain function keeps its own arguments alone, so it cannot
see the function's parameters. A `remote` declaration keeps them: the compiler
writes the name, the default, and the type of each parameter into the remote's
`spec`. Decree reads that `spec` and never binds the remote. No RemoteFunction is
made for a command, and any use of the command as a remote raises.

The declaration must read `remote function Name(...) -> string from client`.
`from client` gives the handler the caller first, and `-> string` makes the
handler answer with the line the caller reads.

`on(Kick)` reads the declaration, and the second call takes the handler. The
two calls let the type check type the handler's parameters from the
declaration, so the handler needs no annotations.

The command is the remote's name in lower case: `Kick` runs as `kick`.

## Arguments

Each parameter is one argument. Its type says how Decree reads the word:

| Type | A line writes |
| --- | --- |
| `number` | A number: `2`, `0.5`, `-3`. |
| `@u8 number` | A whole number from 0 to 255. `@u16`, `@u32`, `@i8`, `@i16`, and `@i32` set other ranges. |
| `boolean` | `true`, `yes`, `on`, `1`, or `false`, `no`, `off`, `0`. |
| `string` | One word, or words in double quotes. |
| the last `string` | The rest of the line. |
| `Player` | `me`, a name, the start of a name, or a display name. |
| `{ Player }` | The same, or `all`, or `others`, with commas between. |
| a unit enum | The name of one variant, in any case. |
| a struct marked `@argument` | What the struct's `parse` reads. |

A parameter with a default, or with a type that ends in `?`, is optional. Each
optional parameter comes after the ones a line must give.

A wrong word answers in the player's words. It names the argument, what it
got, and what it wanted:

```text
roll 300     sides wants a whole number from 0 to 255, but got "300".
wave s       target: "s" could be Shedletsky, StickMasterLuke. Write more of the name.
kick         kick needs target: a player: a name, the start of one, a display name, or me. Write it as: kick <target> [reason]
```

A type that Decree cannot read, such as a `Vector3`, raises when the handler
arrives. So does a required parameter after an optional one.

## Argument types of your own

A struct marked `@argument` is an argument type. The attribute's contract asks
for a static `parse(text, caller)`, so the compiler reports a struct that has
none. `parse` gives the value, or nil for a word that is not one. It can also
call `fail` with a reason of its own.

```aly
import { argument, command, on } from '@packages/decree'

--- The seconds of each unit of a length.
const UNITS: { [string]: number } = { m = 60, h = 3600, d = 86400 }

--- The length of a mute, in seconds.
@argument('a length: 30m, 2h, or 7d')
struct Length
  seconds: number
end

impl Length
  --- The length a word names.
  function parse(text: string, _caller: Player) -> Length?
    const count, unit = string.match(string.lower(text), '^(%d+)(%a)$')
    const each = UNITS[unit ?? '']

    return (count ~= nil and each ~= nil)
      ? new Length { seconds = (tonumber(count) ?? 0) * each }
      : nil
  end
end

--- Mutes a player.
@command(252, 'Mutes a player for a time.')
remote function Mute(target: Player, length: Length) -> string from client

on(Mute)(function(_, target, length)
  target:SetAttribute('MutedUntil', os.time() + length.seconds)

  return `Muted {target.Name} for {length.seconds} seconds.`
end)
```

The struct must be data that a remote can carry, since the declaration is a
remote.

## Permissions

A level is a number that the game gives each player. `@command(level, help)`
names the lowest level that may run the command. `serve(resolver)` takes the
game's resolver, which gives each player a level. Strata uses the ranks of its
group: a moderator holds rank 252 or above, and a developer rank 254 or above.

- The server checks the level of every line. The client only asks.
- A player below a command's level does not see it in the help. A line that
  names it answers that no such command exists.
- A resolver that raises gives level 0, so a fault in it lets no one in.
- `level_of(player)` gives the level that the resolver gives. A handler uses
  it for a check that reads the target, such as "no moderator kicks a
  moderator".

The step
[010_permissions.aly](https://github.com/alloy-luau/packages/blob/main/decree/how_to/010_permissions.aly)
reads the levels from the ranks of a group, with
`GroupService:GetRolesInGroupAsync`.

## The client and a console UI

`run(line)` sends a line to the server and gives a Future of the `Reply`.
`ok` is false when the line did not run, and `text` is the answer to show.
`list()` gives the commands that the player may run, as `Entry` values with a
`name`, a `usage`, and a `help`, for a UI that lists them or completes a line.

```aly
import { list, run } from '@packages/decree'

async do
  for _, entry in await list() do
    print(entry.usage, entry.help)
  end

  print((await run('help kick')).text)
end
```

The server drops a flood of lines: 20 in 10 seconds for each player. `run`
then answers that the player must wait. A line holds 500 characters at most.

On the server, `execute(player, line)` runs a line as that player and gives the
`Reply`. A server hook, such as a chat command, calls it.

## Help

Every player may run `help`. With no word it lists the commands that the
player may run, each with its usage and its line of help. With a command's
name it shows how to write that command, and what each argument wants:

```text
help kick
  kick <target> [reason]
  Kicks a player out of the server.
    target: a player: a name, the start of one, a display name, or me
    reason: the rest of the line; "You were kicked." when left out
```

`<name>` is an argument that a line must give, and `[name]` one that it may
leave out.

## API

| Name | What it is |
| --- | --- |
| `@command(level, help)` | Marks a `remote function` as a command. |
| `on(Remote)(handler)` | Gives a command its handler. The handler gets the caller, then each argument, and returns the answer. |
| `fail(text)` | Ends the command, and answers with `text`. |
| `@argument(wants)` | Marks a struct as an argument type. Requires a static `parse(text, caller)`. |
| `serve(resolver)` | Starts Decree on the server. `resolver` gives each player a level. |
| `level_of(player)` | The level that the resolver gives, or 0 when it raises. |
| `execute(player, line)` | Runs a line as a player on the server. Gives a `Reply`. |
| `entries(player)` | The commands a player may run, as `Entry` values, on the server. |
| `run(line)` | Sends a line from the client. Gives a Future of the `Reply`. |
| `list()` | The commands that this player may run, from the client. Gives a Future. |
| `Reply` | `{ ok: boolean, text: string }`: the answer to a line. |
| `Entry` | `{ name: string, usage: string, help: string }`: one command. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They run
on lest, with Alloy's test shim for the Roblox API.

Decree lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `decree` folder. Each
package there is a member of one Ember workspace and keeps its own version.
Run the commands above from `decree`. `embr publish` from `decree` publishes
Decree alone.

The `how_to` and `examples` files import `@packages/decree`. To check them, copy
them into a game that installs Decree, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/decree/LICENSE).
