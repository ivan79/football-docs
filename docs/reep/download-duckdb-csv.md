---
source_url: https://reep.football/data/
source_type: curated
upstream_version: null
crawled_at: 2026-10-13
---

# Reep download: DuckDB and CSV

The whole register is a free download with no key and no sign-up. It comes as
one DuckDB file, or as one gzipped CSV per table. The CSV release is the source
contract; the DuckDB file is built from it. Written from reep.football on
2026-09-22.

## Stable download links

These links always resolve to the current release:

- DuckDB: https://reep.football/downloads/duckdb (the file is `reep-register-v1.duckdb`)
- One CSV per table: `https://reep.football/downloads/csv/<table>`, for example
  https://reep.football/downloads/csv/bridges

Each release also publishes `checksums.txt` and `schema.json` next to the files;
the manifest at https://data.reep.football/releases/latest.json lists every file
for the current release.

Tables available as CSV: `bridges`, `entities`, `relationships`, `matches`,
`aliases`, `players`, `coaches`, `referees`, `teams`, `competitions`, `seasons`,
`stages`, `observed_clubs`, `redirects`, `coverage`, and the Wikidata overlay
tables `overlay_xids`, `overlay_aliases` and `overlay_links`.

Source: [downloads](https://reep.football/data/). The old page address,
https://reep.football/downloads, redirects there; the `/downloads/duckdb` and
`/downloads/csv/<table>` links above still work.

## Keeping a local copy current

A new release is cut every week, and each file carries its stamp in the
`release_metadata` table:

```sql
SELECT value FROM release_metadata WHERE key = 'source_stamp';
```

Compare it with the `stamp` field of
https://data.reep.football/releases/latest.json, which needs no key. When they
differ, download the file again from https://reep.football/downloads/duckdb over
the old copy, then resolve stored Reep IDs through the `redirects` table. Check
before any bulk matching job.

## The four tables that do the crosswalk

| Table | Columns | What it is |
|---|---|---|
| `bridges` | `provider, namespace, external_id, reep_id, rung, upstream_status` | The crosswalk: one row per provider ID. `rung` says how the bridge was corroborated; `upstream_status` says whether the provider still uses the ID. |
| `entities` | `reep_id, entity_type, status, label, gender, country, …` | One row per identity. |
| `aliases` | `reep_id, alias, kind, rank, language` | Alternate and historical names. |
| `redirects` | `from_id, to_id, reason` | IDs retired by a merge, pointing at the survivor (`reason = merged`), plus a few tombstones with an empty `to_id`. The ledger is flat: a `to_id` is never itself a `from_id`. |

The DuckDB file is self-describing: `SHOW TABLES;` and `DESCRIBE bridges;`.
Source: [get-started guide](https://reep.football/get-started).

### Live and retired provider IDs

`upstream_status` was added to `bridges` in the 2026-09-26 release, at the end of
the column list, so code that reads the first five columns by position is
unaffected. It is empty for an ID the provider still uses. In that release:

| Value | Rows | Meaning |
|---|---|---|
| empty | 7,932,209 | live provider ID |
| `retired` | 330 | the provider retired this ID in favour of another; the same Reep ID also has a live bridge to that provider |
| `removed` | 210 | the provider's own list no longer includes the ID (all SkillCorner in this release) |

Reep's published schema note describes the empty and `retired` values. Look up
by any ID, retired ones included, so that old stored IDs still resolve. When you
map *to* a provider, keep only live rows, or a retired ID can come back next to
the live one. The examples below do this with
`coalesce(b.upstream_status, '') = ''`, which works whether the empty value
arrives as NULL or as an empty string.

### Bridges can name a merged Reep ID

When two Reep IDs merge, the losing ID keeps its bridge rows, and `redirects`
points it at the survivor. So a bridge's `reep_id` can be a merged ID. In release
`20261005T180536Z`, 4,211 bridges name a merged ID. For example,
`transfermarkt / spieler / 1146905` is bridged on `rp799a2e2f9861aa` (status
`redirected`, "Samuel Maslej"), which redirects to `rp277340b925e0d2`
("Samuel Mašlej"). The survivor's own bridges have no Transfermarkt row.

A join from `bridges` straight to `entities` and back to `bridges` therefore
lands on the merged entity and misses the survivor's other provider IDs. In that
release, mapping this Transfermarkt ID to Wyscout that way returns no Wyscout ID.
Resolve every bridge to its survivor first, and join on the survivor. One join is
enough, because the ledger is flat:

```sql
WITH rb AS (   -- every bridge, with the Reep ID it resolves to
  SELECT b.*, coalesce(nullif(r.to_id, ''), b.reep_id) AS survivor_id
  FROM bridges b
  LEFT JOIN redirects r ON r.from_id = b.reep_id
)
SELECT * FROM rb LIMIT 5;
```

An entity's full set of provider IDs is every `rb` row whose `survivor_id` is
its Reep ID. A tombstone (empty `to_id`) leaves the bridge on its own ID. The
examples below use `rb`. Checked on release `20261005T180536Z`.

## Map one provider's ID to another's

Two hops through the Reep ID. Swap the two provider clauses to go the other way.

```sql
-- Wyscout player ID → Reep ID → SkillCorner player ID
WITH rb AS (
  SELECT b.*, coalesce(nullif(r.to_id, ''), b.reep_id) AS survivor_id
  FROM bridges b
  LEFT JOIN redirects r ON r.from_id = b.reep_id
)
SELECT a.external_id AS wyscout_id, e.reep_id, e.label,
       b.external_id AS skillcorner_id
FROM rb a
JOIN entities e ON e.reep_id = a.survivor_id
LEFT JOIN rb b
       ON b.survivor_id = e.reep_id
      AND b.provider = 'skillcorner' AND b.namespace = 'player'
      AND coalesce(b.upstream_status, '') = ''
WHERE a.provider = 'wyscout' AND a.namespace = 'player'
  AND a.external_id = '379209';
-- → rp1b829f1d3468c4 · Declan Rice · skillcorner 12174
```

The same query for `transfermarkt / spieler / 1146905` to Wyscout returns
`rp277340b925e0d2` · Samuel Mašlej · wyscout 908077.

## Bridge a whole file in one pass

DuckDB reads CSV directly, so your own file joins without an import step.

```sql
CREATE TABLE my_squad AS SELECT * FROM read_csv_auto('my_squad.csv');

WITH rb AS (
  SELECT b.*, coalesce(nullif(r.to_id, ''), b.reep_id) AS survivor_id
  FROM bridges b
  LEFT JOIN redirects r ON r.from_id = b.reep_id
)
SELECT m.*, e.reep_id, e.label AS reep_label,
       string_agg(DISTINCT b.external_id, '|') FILTER (WHERE b.provider = 'skillcorner') AS skillcorner_id,
       string_agg(DISTINCT b.external_id, '|') FILTER (WHERE b.provider = 'opta')        AS opta_id
FROM my_squad m
LEFT JOIN rb a
       ON a.provider = 'wyscout' AND a.namespace = 'player'
      AND a.external_id = CAST(m.wyscout_id AS VARCHAR)
LEFT JOIN entities e ON e.reep_id = a.survivor_id
LEFT JOIN rb b
       ON b.survivor_id = e.reep_id
      AND (b.provider, b.namespace) IN (('skillcorner', 'player'), ('opta', 'person'))
      AND coalesce(b.upstream_status, '') = ''
GROUP BY ALL;
```

A NULL `reep_id` means no bridge exists for that ID in this release. A NULL
target ID means the entity is in Reep but not yet bridged to that provider.

## Export flat mapping CSVs

For a spreadsheet or an import job, reshape the long `bridges` table into one row
per entity with a column per provider:

```sql
COPY (
  PIVOT (
    SELECT e.reep_id, e.label, b.provider || '_' || b.namespace AS id_column, b.external_id
    FROM bridges b
    LEFT JOIN redirects r ON r.from_id = b.reep_id
    JOIN entities e ON e.reep_id = coalesce(nullif(r.to_id, ''), b.reep_id)
    WHERE e.entity_type = 'team'
      AND coalesce(b.upstream_status, '') = ''
  )
  ON id_column USING string_agg(DISTINCT external_id, '|')
  GROUP BY reep_id, label
) TO 'teams_all_providers.csv' (HEADER);
```

The match table is larger than Excel's 1,048,576-row limit; filter by competition
or season first. Source:
[exporting flat mapping CSVs](https://reep.football/get-started#export-csv).

## No provider ID: search by name

A name match is a shortlist, not an answer. `entity_search` holds each label with
every alias in one text column, and `bridge_count` is a fair proxy for the
better-known identity:

```sql
SELECT reep_id, label, entity_type, bridge_count
FROM entity_search
WHERE search_text ILIKE '%ødegaard%'
ORDER BY bridge_count DESC
LIMIT 5;
```

Confirm a name match by team, season or nationality before trusting it.
