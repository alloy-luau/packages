# Gloss

Localization for [Alloy](https://github.com/alloy-luau/alloy), with string
keys that the compiler checks.

A game writes its strings in TOML, one file for each language. Alloy imports
a TOML file as a typed table, and Gloss makes the table type of the base
language the shape of every other language. A key that a language lacks is a
type error at the line that adds it. A key that the code misspells is a type
error at the line that reads it. Neither reaches the game as a blank label.

Gloss also fills placeholders, picks plural forms by the CLDR rules, writes
numbers with the separators of each language, reads the language of the
player, and switches the language at run time. It depends on no other
package.

Gloss needs Alloy 0.1.5 or later and embr 0.8.1 or later.

## Learn

- [how_to](https://github.com/alloy-luau/packages/tree/main/gloss/how_to) is a
  tour of Gloss in numbered steps, from `001_strings_from_toml.aly` to
  `020_checking_in_a_test.aly`. Read them in order.
- [examples](https://github.com/alloy-luau/packages/tree/main/gloss/examples)
  holds small games: a shop, a settings page with a language picker, and a
  quest log with plurals in four languages.

## Install

Add Gloss to the game's `ember.toml`, then run `embr install`:

```toml
[dependencies]
gloss = { name = "alloy/gloss", version = "^0.1.0" }
```

Mount the packages folder in `alloy.toml`:

```toml
[mount]
packages = ["packages/roblox", "@game/ReplicatedStorage/Packages"]
```

## Quick start

Write the base language in TOML. English is the base here:

```toml
# src/shared/lang/en.toml
[shop]
title = "Shop"
buy = "Buy for {price} coins"

[quest.kills]
one = "{count} kill"
other = "{count} kills"
```

Write each other language with the same keys:

```toml
# src/shared/lang/fr.toml
[shop]
title = "Boutique"
buy = "Acheter pour {price} pièces"

[quest.kills]
one = "{count} victime"
other = "{count} victimes"
```

Keep one Gloss in a shared module:

```aly
-- src/shared/strings.aly
import { Gloss } from '@packages/gloss'
import en from './lang/en.toml'
import fr from './lang/fr.toml'

--- The languages of the game. English is the base.
export const lang = new Gloss('en', en)
lang:add('fr', fr)
```

A client script shows the language of the player, and each label follows a
change:

```aly
-- src/client/main.client.aly
import Players from '@game/Players'
import { lang } from '@shared/strings'

lang:follow(Players.LocalPlayer)

local title = new Instance('TextLabel') {
  Size = new UDim2(0, 200, 0, 40),
  Parent = new Instance('ScreenGui') { Parent = Players.LocalPlayer->'PlayerGui'! },
}

lang:watch(function(t)
  title.Text = lang:format(t.shop.buy, { price = 1500 })
end)
```

The label shows "Buy for 1,500 coins" for an English account, and "Acheter
pour 1 500 pièces" for a French one.

## Keys the compiler checks

`new Gloss(code, strings)` takes the base language. `Strings<typeof(en)>` is
the table type of the base with each key read only, and with each plural
table widened to `Plural`. `lang.t` has this type, and `add` asks for it.

Take `title` out of `fr.toml`, and `lang:add('fr', fr)` does not compile. The
message names the table where the key is missing:

```text
error(luau): TypeError: Expected this to be '{ read quest: ..., read shop:
  { read buy: string, read title: string } }' but got '{ quest: ..., shop:
  { buy: string } }'; accessing `shop` results in `{ buy: string }` in the
  latter type and `{ read buy: string, read title: string }` in the former
  type
```

Read `lang.t.shop.titel`, and that does not compile either:

```text
error(luau): TypeError: Key 'titel' not found in table
  '{ read buy: string, read title: string }'
```

What Luau checks, and what it does not:

- A missing key is an error, at each depth.
- An extra key is not. Luau takes a table with more keys where it asks for
  fewer. `lang:check()` lists extra keys.
- A plural table may hold other forms than the base. Russian adds `few` and
  `many`, and Japanese has `other` alone.
- The text of a string is not part of its type, so Luau cannot see a
  placeholder. `lang:check()` compares the placeholders of each language with
  the base.
- An array keeps the type of the base, and its items must match it exactly.
- A key in a variable, `t.items[id]`, does not type check, because Luau does
  not index a table type with a string. Give each item a function that reads
  its key, as the shop example does.
- `alloy check` runs no Luau type check. The check runs under `alloy flux`
  and in the editor.

The shape comes from a Luau type function, so it follows the TOML file. Add a
key to `en.toml`, and each language that lacks it stops compiling.

## Placeholders

A placeholder is a name in braces, as in a Roblox LocalizationTable:
`"Buy for {price} coins"`. A translator moves it where the grammar of the
language puts it, so the code never joins pieces of a sentence.

```aly
import { Gloss } from '@packages/gloss'
import en from './lang/en.toml'

const lang = new Gloss('en', en)

print(lang:format(lang.t.shop.buy, { price = 1500 })) -- Buy for 1,500 coins
print(lang:format('{name} #{id}', { name = 'Sword', id = '10024' })) -- Sword #10024
```

- A number value takes the separators of the language. Pass a string for a
  code or an id.
- A name with no value stays in the text, so a missing value shows as
  `{price}` in the label. It does not stop the game.
- Braces around anything other than a name, such as `{ }`, stay as they are.

## Plurals

A plural is a TOML table with one string for each form of a count, as CLDR
names them: `zero`, `one`, `two`, `few`, `many`, and `other`. A plural must
have `other`, and the other forms are optional. A table whose keys are all form names, and that holds `other`, is a
plural.

```toml
# src/shared/lang/ru.toml
[shop]
title = "Магазин"
buy = "Купить за {price} монет"

[quest.kills]
one = "{count} убийство"
few = "{count} убийства"
many = "{count} убийств"
other = "{count} убийства"
```

`lang:plural(forms, count, args?)` picks the form by the rule of the language
that shows, and fills `{count}`:

```aly
import { Gloss } from '@packages/gloss'
import en from './lang/en.toml'
import ru from './lang/ru.toml'

const lang = new Gloss('en', en)
lang:add('ru', ru)

print(lang:plural(lang.t.quest.kills, 1)) -- 1 kill
lang:set('ru')
print(lang:plural(lang.t.quest.kills, 21)) -- 21 убийство
print(lang:plural(lang.t.quest.kills, 5)) -- 5 убийств
```

Gloss knows the rules of English, German, Dutch, Swedish, Finnish, Norwegian,
Italian, Spanish, Portuguese, French, Turkish, Greek, Hungarian, Russian,
Ukrainian, Polish, Czech, Slovak, Arabic, Japanese, Korean, Chinese, Thai,
Vietnamese, Indonesian, and Malay. Another language takes the English rule. A
form that the table lacks shows `other`.

CLDR gives French, Spanish, Italian, and Portuguese a `many` form for whole
millions ("1 million de"). Gloss leaves it out, so those counts take `other`,
and a translator writes one form less for each plural.

## Numbers

`lang:number(value, decimals?)` writes a number with the separators of the
language that shows: 1,234,567.5 in English, 1.234.567,5 in German, and
1 234 567,5 in French. `format` and `plural` use it for each number value.

- Without `decimals`, the fraction has at most two digits, and no zeros at
  the end.
- With `decimals`, the fraction has that many digits.
- A language that groups with a space uses a no-break space, U+00A0, so a
  number stays on one line.
- Spanish and Polish group from five digits: 1500, but 15.000.

## Languages and fallback

`add(code, strings)` takes a language with each key. `draft(code, strings)`
takes a language that a translator has not finished: each key is optional,
and the base fills each gap.

The base fills a gap in each language:

- A key that the language lacks shows the base string.
- An empty string shows the base string too. A translation tool often leaves
  one where no one has translated yet.
- A plural table and an array come whole from the language, so a French count
  never shows an English form.

The merge runs once, when `add` or `draft` takes the language. `lang.t` is a
plain table, so a read costs no more than any table read.

`set(locale)` takes a Roblox locale, such as `fr-ca`. It tries the locale,
then its language, then the base, and returns the code that it picked. A code
is lowercase with a dash, so `en_US` reads `en-us`.

## The player's language

`Player.LocaleId` holds the locale of the player's Roblox account.
`lang:follow(player)` shows that language now, and again when the player
changes it in the Roblox settings. It returns the connection.

When the player picks a language in the game, the pick wins. Disconnect the
connection, then call `set`:

```aly
import Players from '@game/Players'
import { lang } from '@shared/strings'

const account = lang:follow(Players.LocalPlayer)

--- Shows the language that the player picks in the game's settings.
export function pick(code: string) -> ()
  account:Disconnect()
  lang:set(code)
end
```

Most games translate on the client. The server sends keys and values, and
each client formats them in its own language.

## Switching at run time

`set` replaces `lang.t` and fires `lang.changed`, a std `Signal<string>`,
with the new code. It fires only when the code changes.

`lang:watch(fn)` runs `fn` with the strings now, and again after each change.
It returns a `SignalConnection`; disconnect it when the label goes. A local
that holds `lang.t` keeps the old language, so read the `t` that `watch`
passes.

`lang:languages()` lists the codes, the base first. `lang:strings(code)`
gives the strings of one language, so a language picker can show each name in
its own language.

## Checking in a test

Some problems are not in the types: an extra key, an empty string, a
placeholder that differs from the base, and a plural form that a language
uses but lacks. `lang:check()` returns one line for each:

```text
fr: shop.stale is not in the base
de: shop.title is empty, so the base shows
fr: shop.buy has {cost}, and the base has {price}
ru: quest.kills has no few form, so few counts show the other form
```

Assert in a test that the list is empty, so `alloy test --run` fails when a
translation falls behind:

```aly
import { lang } from './strings'

@test
function each_language_is_complete() -> ()
  $assert_eq(table.concat(lang:check(), '\n'), '')
end
```

## Roblox LocalizationTable

Gloss 0.1.0 does not read or write a LocalizationTable. A table feeds
`AutoLocalize` and the `Translator`, which set text by their own rules. A game
that uses both has two copies of each string to keep in step. A table that
Gloss reads at run time has no type, so the key check would not see it. Turn
off `AutoLocalize` on each label that Gloss sets, so Roblox does not
translate the text again.

## API

| Name | What it is |
| --- | --- |
| `new Gloss(code, strings)` | A Gloss with the base language. The base shows first. |
| `lang.t` | The strings of the language that shows, the base in each gap. |
| `lang.code`, `lang.base` | The code of the language that shows, and of the base. |
| `lang.changed` | A `Signal<string>` that fires with the new code. |
| `lang:add(code, strings)` | Adds a language with each key of the base. |
| `lang:draft(code, strings)` | Adds a language that may lack keys. |
| `lang:set(locale)` | Shows the language for a locale. Returns the code. |
| `lang:resolve(locale)` | The code that `set` picks for a locale. |
| `lang:follow(player)` | Shows the language of the player, and follows it. |
| `lang:watch(fn)` | Runs `fn` with the strings now and after each change. |
| `lang:languages()` | The codes, the base first. |
| `lang:strings(code)` | The strings of one language, or nil. |
| `lang:format(text, args?)` | Fills each `{name}`. |
| `lang:plural(forms, count, args?)` | Picks the form for `count`, then fills it. |
| `lang:number(value, decimals?)` | Writes a number with the separators of the language. |
| `lang:check()` | Lists each problem in the languages. |
| `Strings<T>`, `Draft<T>` | The shape of a language, and of a draft. |
| `Plural` | A table of plural forms. It must have `other`. |
| `Args` | The values of the placeholders: `{ [string]: string \| number }`. |

## Development

```sh
alloy check          # the compiler and the lints
alloy flux           # the same, plus luau-lsp
alloy test --run     # the tests, on lest
alloy fmt            # the layout
```

The tests are the `@test` namespaces at the end of each file in `src`. They
run on lest, with Alloy's test shim for the Roblox API.

Gloss lives in [alloy-luau/packages](https://github.com/alloy-luau/packages),
the repository of the official Alloy packages, in the `gloss` folder. Each
package there is a member of one Ember workspace and keeps its own version.
Run the commands above from `gloss`. `embr publish` from `gloss` publishes
Gloss alone.

The `how_to` and `examples` files import `@packages/gloss`. To check them,
copy them into a game that installs Gloss, then run `alloy flux` there.

## License

MIT. See [LICENSE](https://github.com/alloy-luau/packages/blob/main/gloss/LICENSE).
