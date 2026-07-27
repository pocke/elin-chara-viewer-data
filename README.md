# elin-chara-viewer-data

Archived game data for [elin-chara-viewer](https://github.com/pocke/elin-chara-viewer).

The viewer bundles only the two current versions (EA and Nightly). Every other
version lives here instead, and the viewer fetches it at request time.

## Layout

```
index.json                    catalog of every version, generated from the files below
v/<slug>/meta.json            version, channel, releaseDate
v/<slug>/csv/<table>.csv      exported game data
v/<slug>/featModifier.json    feat modifiers, and the decompiled build they came from
v/<slug>/ids.json             chara ids and element aliases, so unknown URLs can 404
history/charas/<page>.json    what changed about one character page, version by version
history/manifest.json         how much history the last build produced
```

`<slug>` is the version name with its spaces replaced: `EA 23.306 Patch 1`
becomes `EA-23.306-patch-1`.

`meta.json`:

```jsonc
{
  "version": "EA 23.306 Patch 1", // as reported by the game
  "channel": "nightly",           // "stable" or "nightly"
  "releaseDate": "2026-05-14"
}
```

Each version directory is self-contained: everything about a version that cannot
be computed from its own files is in its own `meta.json`. `index.json` is a
generated aggregate, and exists because neither R2 nor raw.githubusercontent.com
can list a directory while the viewer needs to enumerate the versions.

`history/` is not filed under a version because a history spans all of them: one
file holds every change to one character detail page, newest first, and a page
fetches only its own. It is derived from `v/`, not a source of truth — the
viewer's `npm run build:history` rebuilds all of it from scratch on every
release, so a version added out of order or a corrected `releaseDate` is picked
up without anything else to do.

`releaseDate` currently holds the day the version was archived, which is close to
but not always the release date. The Steam backfill will correct it.

## The shape of a history

The file name is the identifier the detail page uses: `putty.json` for a
character, `bit---eleFire.json` for one of its element variants.

```jsonc
{
  "schemaVersion": 1,
  "key": "putty",
  "isVariant": false,          // true for an `id---element` page
  "names": {                   // element alias -> the newest name it was seen under
    "CHA": { "ja": "魅力", "en": "Charisma" }
  },
  "entries": [ /* newest first */ ]
}
```

A reader that does not recognise `schemaVersion` hides the history rather than
guessing; fields are only ever added within a version, since an unknown field is
ignored.

`names` exists because an alias can be renamed, or stop existing. A page names an
alias from its own version's `elements.csv` and falls back here, so one alias
reads as one name all the way down the list.

Each entry is one version in which something happened. Versions where nothing
did are not listed at all.

```jsonc
{
  "version": "EA 23.269",
  "slug": "EA-23.269",         // the URL segment, and the directory under v/
  "channel": "nightly",
  "releaseDate": "2026-02-11",
  "kind": "changed",
  "changes": [ /* what the detail page shows */ ],
  "raw": [ /* what only the raw-data section shows */ ]
}
```

`kind` is one of:

| | |
|---|---|
| `origin` | Where the record starts. The character was already there — either this is the oldest archived version, or the versions before it could not be computed. |
| `added` | The page did not exist in the version before. |
| `removed` | It stopped existing. |
| `changed` | Values moved; `changes` and `raw` say which. |
| `unavailable` | The version has the character but the viewer cannot compute it, because the row points at a race or a job that version does not have. `reason` carries what went wrong. |

Only `changed` carries anything in `changes` or `raw`. An `origin` or `added`
entry marks the place and no more: the values themselves are the character
rather than a change, and that version's own page shows them.

`changes` are the things the detail page puts on screen. `field` names one of
ten groups and `key` picks a member out of it, or is absent where the group
holds one value:

| `field` | `key` | `from` / `to` |
|---|---|---|
| `name` | `ja`, `en` | string |
| `mainElement` | — | element alias, or null |
| `race`, `job` | `id`, `nameJa`, `nameEn` | string |
| `tactics` | `id`, `nameJa`, `nameEn`, `distance`, `moveFrequency`, `party`, `taunt`, `melee`, `range`, `spell`, `heal`, `summon`, `buff`, `debuff`, `partyBuff` | number, string, or boolean for `partyBuff` |
| `level` | — | number |
| `geneSlot` | — | number, after the feats |
| `bodyParts` | `hand`, `head`, `torso`, `back`, `waist`, `arm`, `foot`, `neck`, `finger` | number |
| `elements` | element alias | array of the powers merged into it, sorted; null where the character does not have it |
| `abilities` | `<name>\|<element>` | `{ name, chance, party, element }`, or null |

Everything an element carries is one group, so a power change reads the same
whether the page files it under attributes, resistances or skills. An element
being renamed is not a change to a character and is not recorded; the alias is
the identity.

`elements` holds the powers rather than their total because a skill is shown as
base potential, which counts only the powers that are not 1.

`raw` is the rest of the four rows the page prints raw — `charas`, `races`,
`jobs`, `tactics` — as they read after parsing, so a column the viewer's schemas
do not know is not in here either. A column is listed only when `changes` does
not already account for it, which is why a reordered `elements` string shows up
here while a change to what it grants does not. Values are strings, or null for
a column with nothing in it. `swapped` marks a row that changed because the
character started pointing at a different one.

```jsonc
{ "table": "races", "column": "playable", "from": "8", "to": "4" }
```

`history/manifest.json` records how much the last build produced, and is what the
next one compares against before it agrees to write.

## Incomplete versions

41 versions contain only five tables (`charas`, `elements`, `jobs`, `races`,
`tactics`), because the exporter did not yet dump every table when they were
archived. Those five are enough for the character and feat pages; the SQL page is
limited for them. Completing them is tracked in
[elin-chara-viewer#303](https://github.com/pocke/elin-chara-viewer/issues/303).

## How versions get here

A released version is added by the viewer repository's `archive.yml` workflow
once the release is merged. Versions outside that flow — the ones restored from
git history, and the ones that will be downloaded from Steam — are added by
running the scripts from an elin-chara-viewer checkout:

```console
$ ruby script/archive_release.rb ../elin-chara-viewer-data 'EA 23.306' \
    --channel stable --release-date 2026-05-10 --db /path/to/exported/csv
$ ruby script/extract_feat.rb --archive ../elin-chara-viewer-data --version 'EA 23.306'
```

`--db` points at the directory holding `<version>/*.csv`, and `--channel` is
required for a version that no `versions/` file names. `archive_release.rb`
regenerates `index.json`. A version added this way also belongs in the history,
which the same checkout rebuilds:

```console
$ npm run build:history -- ../elin-chara-viewer-data
```

It writes nothing when the result would lose pages or move the entry count by
more than a quarter, which a version added in the wrong place would cause;
`--allow-large-change` says the difference is meant. Committing and pushing here
is all that is left.

## Delivery

Pushing to `main` syncs the archive to Cloudflare R2, which is what the viewer
reads. A push uploads the files it changed; the whole `v/` and `history/` trees
and `index.json` go up on any run that is not a push — started by hand (the way
to seed an empty bucket) or the weekly one, so that a push whose run was
superseded while queued cannot leave the bucket behind.

`v/` is cached for a day because an archived version's files only change when
the archive is rebuilt to correct them. `index.json` and `history/` are cached
for five minutes, because a release changes both.

A bucket served directly to browsers needs the CORS policy in `cors.json`, which
is applied by hand because changing bucket configuration requires an
admin-scoped R2 token while the sync only needs object access:

```console
$ npx wrangler r2 bucket cors set <bucket> --file cors.json
```

The `range` header has to be allowed and `content-range` exposed because the
viewer's SQL page reads the CSVs with DuckDB-wasm, which requests them in ranges.
A bucket served through a Worker sets these headers in the Worker instead.

## License

The files under `v/*/csv/` are imported from the Elin game, and the files under
`v/*/featModifier.json` are generated from its decompiled source. They are not
covered by the viewer's MIT license.
