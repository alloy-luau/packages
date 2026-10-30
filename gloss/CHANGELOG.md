# Changelog

## 0.1.0

The first release.

- `Gloss`: the languages of a game, and the one that shows now. The table
  type of the base language is the shape of each other language.
- Keys that the compiler checks: `add` asks for each key of the base, so a
  missing key is a type error. A misspelled read is a type error too.
- `draft` for a language that is not finished. The base fills each gap,
  and an empty string shows the base string.
- Placeholders: `format` fills each `{name}`. A number value takes the
  separators of the language.
- Plurals: `plural` picks the CLDR form of a count, for 26 languages.
- Numbers: `number` writes the group and decimal marks of the language.
- The player's language: `set` reads a Roblox locale, and `follow` reads
  `Player.LocaleId` and follows its change.
- Switching at run time: `set` fires `changed`, and `watch` runs a
  function with the strings now and after each change.
- `check` lists what the types do not catch: extra keys, empty strings,
  placeholders that differ, and missing plural forms.
- A `how_to` tour in numbered steps, and three example games.
