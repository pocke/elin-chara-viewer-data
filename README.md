# elin-chara-viewer-data

Archived game data for [elin-chara-viewer](https://github.com/pocke/elin-chara-viewer).

The viewer bundles only the two current versions (EA and Nightly). Every other
version lives here instead, and the viewer fetches it at request time.

## Layout

```
index.json                    catalog of every version, generated from the files below
v/<slug>/meta.json            version, channel, releaseDate, source
v/<slug>/csv/<table>.csv      exported game data
v/<slug>/featModifier.json    feat modifiers, and the decompiled build they came from
v/<slug>/ids.json             chara ids and element aliases, so unknown URLs can 404
v/<slug>/derived/             computed data (version diffs); not populated yet
```

`<slug>` is the version name with its spaces replaced: `EA 23.306 Patch 1`
becomes `EA-23.306-patch-1`.

`meta.json`:

```jsonc
{
  "version": "EA 23.306 Patch 1", // as reported by the game
  "channel": "nightly",           // "stable", "nightly", or null when it was
                                  // archived before the channel was recorded
  "releaseDate": "2026-05-14",
  "source": "git"                 // "git" or "depot"
}
```

Each version directory is self-contained: everything about a version that cannot
be computed from its own files is in its own `meta.json`. `index.json` is a
generated aggregate, and exists because neither R2 nor raw.githubusercontent.com
can list a directory while the viewer needs to enumerate the versions.

`source` records where the data came from: `git` for the versions recovered from
the viewer repository's history, `depot` for versions downloaded from Steam with
DepotDownloader.

`releaseDate` currently holds the day the version was archived, which is close to
but not always the release date. The Steam backfill will correct it.

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
$ ruby script/archive_release.rb ../elin-chara-viewer-data 'EA 23.306' --source depot
$ ruby script/extract_feat.rb --archive ../elin-chara-viewer-data --version 'EA 23.306'
```

`archive_release.rb` regenerates `index.json`, so committing and pushing here is
all that is left.

## Delivery

Pushing to `main` syncs the archive to Cloudflare R2, which is what the viewer
reads. Only the files that the push changed are uploaded.

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
