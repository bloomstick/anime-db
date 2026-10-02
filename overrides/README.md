# overrides — manual own-language synopses only

`descriptions.json` maps `"<dreamId>"` → own rewritten synopsis text.
Scraped text never enters this directory or any published `meta-*.json`.

Gate: every published row with non-empty `description` MUST have a key here
(`"<id>"` or `"dream:<id>"`). Until a title is rewritten by hand, its
published `description` stays `''`.
