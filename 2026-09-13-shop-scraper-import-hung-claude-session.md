# 2026-09-13 - Claude Code session hung on a 30-minute import run in the foreground

**Status:** Resolved
**Area:** Claude Code (agent workflow), valiora shop-scraper, structlog logging

## Summary

A Claude Code session implementing the valiora online-shop scraper started the
first full import (35k products from four shop APIs) as a foreground shell
command inside the session, to test it. The import ran for about 27 minutes
and, at INFO level, wrote two to three log lines per item - on the order of
100k JSON lines. The command exceeded the session's tool timeout and its output
flooded the context; the session became unusable. The import itself completed
correctly. A second session verified the data, added a `--limit` dev mode,
moved the per-item log lines to DEBUG and documented a testing ladder.

## Impact

- One Claude Code session lost (had to be abandoned; work continued in a new
  session). No code or data lost: all changes were in the working tree, the
  import committed cleanly.
- ~30 minutes of wall time plus the re-derivation of context in the new session.

## Environment

| Component | Detail |
|---|---|
| Project | valiora, branch `189-getting-whole-online-shop-data`, `apps/shop-scraper` (new) |
| Runtime | docker compose service `shop-scraper` (python 3.12-slim), Postgres 16, Redis 7 on the dev machine |
| Logging | structlog JSON to stdout, `LOG_LEVEL=INFO` |
| Claude Code | Bash tool, default timeout 2 min, maximum 10 min per call |

## Timeline

Local time (CEST). Times marked *db* come from `flyer_offers.created_at`
(transaction start) and `suppliers.last_scraped_at`; the rest is estimated.

- ~20:55 *db* - SPAR import transaction starts (22 276 offers).
- 21:13 *db* - SPAR committed (`last_scraped_at`), ~18 min.
- 21:15-21:21 *db* - BILLA (12 288 offers), ~6 min.
- 21:21-21:22 *db* - Penny (385) and Lidl (270), 34 s and 20 s.
- Around the same time - the session that launched the run stopped responding
  usefully (estimated; observed by the user as "aufgehängt").
- 21:27-22:00 - new session: DB state inspected, all four snapshot flyers
  complete, gates passed, unit/API tests green.
- 22:00-22:30 - `--limit`, DEBUG log levels, compose comment fix, docs.

## Root cause

Two independent limits were both exceeded by one command:

1. **Duration.** A full run takes ~27 min. The time is not HTTP (SPAR: 23
   requests ≈ 25 s) but the importer's per-item product resolution: for every
   item without an exact match it ORM-loads all products of the same unit
   (and brand) and fuzzy-scores them; ≈ 50 ms × 35k items. The Bash tool
   allows at most 10 min per call.
2. **Output volume.** `product_created`, `canonical_created`, `brand_created`,
   `fuzzy_match_found` and `category_tag_inferred` were INFO, i.e. 2-3 JSON
   lines per item → roughly 100k lines for one run, all of it returned into
   the session context.

Either alone would have been survivable (a timeout returns; a big log from a
short command can be tailed). Together, the session waited the full timeout
and then received a context-sized blob.

## Evidence

- Import completed: `select storage_path, count(*) from flyers join
  flyer_offers using (flyer_id) where storage_path like 'shop-api/%' group by 1`
  → spar 22 276, billa 12 288, penny 385, lidl 270; `price_history` has
  34 212 rows with `price_type='shop'`.
- Durations: `min(created_at)` / `last_scraped_at` per supplier as in the
  timeline.
- Log volume: the per-item INFO calls are visible in the diff of
  `importer.py` / `canonical.py` (five events fired per new product). After
  moving them to DEBUG, `--source penny --limit 20` produced 15 lines total.
  The original run's line count was not captured - see *Unverified*.

## What didn't work

- Nothing was tried in the hung session itself; it was abandoned.
- Running pytest inside the `shop-scraper` image fails (`No module named
  pytest`) - pytest is not in `requirements.txt`. Installing with
  `pip install --user` also fails because the container user has no home
  (`/nonexistent`); `HOME=/tmp PYTHONUSERBASE=/tmp/py` plus `PYTHONPATH`
  works (documented in the cheatsheet).
- `docker compose --profile base --profile workers run shop-scraper` (the
  command in the compose comment) fails: `service "scraper" depends on
  undefined service "api-migrator"`. `--profile all` is needed.

## Resolution

- `apps/shop-scraper`: `--limit N` dev mode (first N items only; skips gates,
  delisting and alerts - `save_extracted_products(delist_missing=False)`).
- `valiora_common/services/importer.py`, `canonical.py`: per-item log lines
  → DEBUG; batch summaries stay INFO.
- Compose comment corrected to `--profile all` with detached run + grep.
- Documented ladder: unit tests → `--dry-run` → `--limit` → detached full run
  with `docker logs | grep 'shop_source_finished|shop_ingestion_failed'` →
  verify in SQL. Lives in valiora `docs/vault/Shop Scraper.md` §Testing and
  `docs/guides/command-cheatsheet.md`.

## Unverified

- The exact log-line count of the original run was not captured (the session
  is gone); "~100k" is extrapolated from lines per item × items.
- Whether the session died from the timeout, the output size, or both is not
  distinguishable after the fact; both limits were exceeded.

## Prevention

- **Agent rule of thumb:** a batch that takes more than a few minutes or
  writes more than a few hundred lines is never run in the foreground of a
  session. Start it detached (`docker compose run -d` / `run_in_background`),
  read only summary events with `grep`, verify results in SQL.
- **Code:** `--limit` and `--dry-run` exist so the write path can be tested
  in seconds (`apps/shop-scraper/app/main.py`).
- **Logging rule:** one line per item is DEBUG, one line per batch is INFO
  (valiora `docs/vault/Logging And Observability.md`).
- **Where it acts:** the compose file comment next to the service, the CLI
  docstring, the vault note and the cheatsheet all carry the detached-run
  commands, so the next person copies the safe form.
