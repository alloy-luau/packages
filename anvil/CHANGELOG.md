# Changelog

## 0.1.0

The first release.

- Commands: `@command(level, help)` on a `remote function`, and
  `on(Remote)(handler)`. Anvil reads the parameters from the declaration,
  and the type check holds the handler to it.
- Arguments: numbers, whole numbers by width attribute, booleans, words,
  the rest of the line, a `Player`, a `{ Player }` with `all` and
  `others`, unit enums, and optional arguments with defaults. A wrong
  word answers with the argument, what it got, and what it wanted.
- Argument types of the game's own: `@argument(wants)` on a struct, with
  a contract that asks for a static `parse(text, caller)`.
- Permissions: levels that the game gives through the resolver of
  `serve`. The server checks every line.
- The remote: a client sends a line with `run`, and the server parses,
  checks, runs, and answers. `list` gives the commands for a console UI.
  `execute` runs a line on the server.
- Help: `help` lists the commands that a player may run, and
  `help <command>` shows how to write one.
- A `how_to` tour in numbered steps, and three example games.
