# anime-db — DreamDB offline anime catalog (data only)

Data-only snapshot backing AnimeDream's offline catalog. No code lives here —
only published JSON built on a builder machine by the (monorepo-external) P2
builder. Clients resolve chunk URLs from `manifest.json`, so a Raw → R2/S3
host migration needs zero client change.

## Sources (attributed)

- YummyAnime — `https://old.yummyani.me` (catalog JSON + one item HTML page;
  covers `static.yani.tv`). Supplies `yummy:<alias>` keys, per-episode video
  rows, `remote_ids.shikimori_id` / `myanimelist_id`.
- Shikimori — `https://shikimori.one` (ranked anime catalog + entry detail:
  name variants, genres, studios, votes, MAL id). Read-only, no auth.
- Kodik — `https://kodikplayer.com` (embed `iframe_url` only; resolved HLS is
  never stored — signatures expire same-day).
- MyAnimeList — outbound deep links only (`https://myanimelist.net/anime/<id>`);
  no MAL fetching. `malId` is kept on each row for the outbound link.

## Cadence

- Snapshot rebuilds are periodic (target: weekly). Each publish bumps
  `dbVersion` (`YYYYMMDD`) and adds/updates `meta-*.json` chunks + `videos/`
  rows. Consumers poll `manifest.json` and fetch only chunks whose `sha256`
  changed.

## Layout

| Path | Shape |
|---|---|
| `/manifest.json` | `{dbVersion, titleCount, chunks[{name, count, updatedAt, sha256, url}]}` |
| `/meta-*.json` | 500 light rows/chunk, no videos. Row = `TitleInfo` JSON subset (`id`, `slugUrl: dream:<int>`, names, `coverUrl`, `status`, `bannerUrl`, `studio`, `director`, `sourceWork`, `releaseDate`, `episodeDates`, `yummyAlias`, `shikimoriId`, `malId`, genres/tags, `description` — see policy below) |
| `/videos/<dreamId>.json` | Array of `{episode, dub, player, iframeUrl, date, durationSec}` — `iframeUrl` only |
| `/aliases.json` | `{dbVersion, yummy: {alias: dreamId}, shikimori: {id: dreamId}, mal: {id: dreamId}}` |
| `/registry.json` | Allocator, source of truth: `{dbVersion, nextId, entries[{dreamId, yummyAlias, shikimoriId, malId, addedAt}], tombstones[{dreamId, reason, removedAt}]}` |
| `/overrides/descriptions.json` | Manual own-language synopses keyed by `dreamId`. The ONLY source of non-empty `description` values |
| `build/raw/` | Builder scratch (upstream fetches). NEVER committed, NEVER published |

## ID stability promise

- `dream:<int>` ids are allocated once from `registry.json` (`nextId`
  increments, never decrements) and are append-only / never reused.
- Deletions become tombstones (`tombstones[]`), never id reuse, never
  renumbering.
- After a version pin (git tag == `dbVersion`, e.g. `20261003`), registry and
  history are frozen: no rewrites, no force-pushes, only new appends in later
  versions.

## Description policy

- Published `meta-*.json` rows MUST contain zero scraped descriptions:
  `description: ''` everywhere except rows with a manual entry in
  `overrides/descriptions.json` (own rewritten synopsis).
- The description gate (run on the builder before publish, output pasted in
  the PR) asserts: every non-empty `description` has an overrides key.
- Until a title is rewritten by hand, its `description` stays empty.

## Consuming / verifying (clean machine)

```powershell
$base = 'https://raw.githubusercontent.com/bloomstick/anime-db/20261003'
Invoke-WebRequest "$base/manifest.json" -OutFile manifest.json
# fetch each chunk URL from the manifest, then check sha256:
(Get-FileHash meta-0001.json -Algorithm SHA256).Hash.ToLower()
# description gate (needs overrides/descriptions.json + all meta-*.json locally):
# every non-empty description must have an overrides key — expect zero violations.
python3 -c "import json,glob; ov=json.load(open('overrides/descriptions.json')); n=v=0
for f in glob.glob('meta-*.json'):
  for r in json.load(open(f)):
    n+=1
    if r.get('description'):
      v+=1; assert str(r['id']) in ov or r['slugUrl'] in ov, r
print(f'rows={n} nonempty={v} overrides={len(ov)} gate=PASS')"
```

## License note

Bundled: facts/ids/URLs (titles, ids, episode numbers, iframe URLs).
Creative text only as own rewrites (`overrides/`). Attribution above +
outbound `shikimoriId`/`malId` kept on every row.

## Current snapshot — 20261003 (v0 scaffold)

- `titleCount: 0`, chunks: 0 (P2 builder spec `tools/anime_db/README.md` is
  absent from the monorepo, so no build output exists yet to publish).
- `aliases.json`: 0 entries in every map. `registry.json`: `nextId: 1`,
  no entries/tombstones, frozen at pin.
- Field emptiness: `description`/`bannerUrl`/`episodeDates`/`director` 100%
  empty (vacuous — 0 rows). Alias coverage 0%.
- `build/raw/` excluded via `.gitignore`; nothing raw was ever committed.
