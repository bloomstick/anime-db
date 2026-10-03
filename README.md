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

## Current snapshot — v0-partial (tag `v0-partial`)

- `titleCount: 22733`, chunks: 46 (`meta-0001.json` … `meta-0045.json` at
  500 rows, `meta-0046.json` at 233 rows), `videos/` rows: 22733 files,
  `linked/` rows: 22733 files (12789 titles with links, rest
  `{"links": []}`),
  `aliases.json`: 23719 entries (yummy 999, shikimori 22720),
  `registry.json`: `nextId: 22734`.
- Merge joins (from `build/merged.json` stats): by shikimoriId 967,
  by MAL id 3, by exact name 1; yummy-only 31, shiki-only 21731;
  Manami attached 26606, unmatched 14931 (unmatched rows are counted
  and dropped — Manami never allocates `dream:<id>`).
- Builder `verify` PASS on this snapshot (single note only:
  `dream:2000` genre 38 → 24 truncation under the +24 cap).
- Known gaps (v1 reserved for the full-detail rebuild off this same
  registry): `bannerUrl` always empty (no banner source);
  shiki-only rows are thin (details fetched for 2897/24005 so far —
  list-row fields only for the rest); franchise links are partial
  (yummy view-list side plus Shikimori related where details exist);
  every `description` is empty (own-only policy, no manual rewrites yet).
- Sources + attribution: YummyAnime `https://old.yummyani.me`, Shikimori
  `https://shikimori.one`, Kodik embed `iframe_url` only (no stored HLS),
  MyAnimeList outbound links only, plus
  `manami-project/anime-offline-database` 2026-27 (ODbL-1.0, sha256
  `8a631897…861b1aa`). Manami-derived fields (`enNames` unions, `linked`
  edges, fill-when-empty `type`/`status`/`malId`) ship with this
  attribution; ODbL share-alike fit stays a flagged open question.
- ID stability: `dream:<id>` allocation is append-only from this tag
  (`registry.json` committed here); v1 rebuilds off the same registry,
  never renumbering.
