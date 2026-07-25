# elin-chara-viewer-data

Archived game data for [elin-chara-viewer](https://github.com/pocke/elin-chara-viewer).

The viewer bundles only the two current versions (EA and Nightly). Every past
version lives here instead, and the viewer fetches it at request time.

## Layout

```
csv/<slug>/<table>.csv     # exported game data, one directory per version
featModifier/<slug>.json   # feat modifiers extracted from the decompiled source
ids/<slug>.json            # chara ids and element aliases, for 404s on unknown URLs
derived/                   # computed data (version diffs); not populated yet
index.json                 # catalog of every archived version
```

`index.json` entry:

```jsonc
{
  "version": "EA 23.306 Patch 1", // as reported by the game
  "slug": "23.306-patch-1",       // directory name and URL segment
  "channel": "nightly",           // "stable", "nightly", or null for builds
                                  // archived before the channel was recorded
  "date": "2026-05-14",           // when the version was first archived
  "tables": ["charas", "elements", "jobs", "races", "tactics"],
  "contentHash": "...",           // identical hashes mean identical data
  "source": "git",                // "git" or "depot"
  "featModifier": true,
  "featModifierSource": "EA 23.306" // the decompiled build it was taken from,
                                    // which is not always the same version
}
```

`source` records where the data came from: `git` for versions recovered from the
viewer repository's history, `depot` for versions downloaded from Steam with
DepotDownloader.

## Incomplete versions

41 of the versions recovered from git history contain only five tables
(`charas`, `elements`, `jobs`, `races`, `tactics`), because the exporter did not
yet dump every table when they were archived. Those five are enough for the
character and feat pages; the SQL page is limited for them. Completing them is
tracked in [elin-chara-viewer#303](https://github.com/pocke/elin-chara-viewer/issues/303).

## Regenerating

From an elin-chara-viewer checkout:

```console
$ ruby script/archive_versions.rb ../elin-chara-viewer-data
$ ruby script/extract_feat.rb --archive ../elin-chara-viewer-data
```

## Delivery

Pushing to `main` syncs the archive to Cloudflare R2, which is what the viewer
reads. Only the files that the push changed are uploaded, so a correction that
happens to keep a file's size is published too.

The viewer reads these files from the browser, so the bucket needs the CORS
policy in `cors.json`; the sync workflow applies it on every run. The `range`
header has to be allowed and `content-range` exposed because the viewer's SQL
page reads the CSVs with DuckDB-wasm, which requests them in ranges.

## License

The files under `csv/` are imported from the Elin game, and the files under
`featModifier/` are generated from its decompiled source. They are not covered
by the viewer's MIT license.
