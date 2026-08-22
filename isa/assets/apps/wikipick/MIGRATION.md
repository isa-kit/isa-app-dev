# wikipick (Wikipick) — schema migrations

## 2026-08-21 — crawl-fetch throttle + failure reasons (additive only)

seedVersion `2026-08-20.wikiparity-wp2` -> `2026-08-21.crawl-throttle` — the
build guard (`tool/check_schema_seedversion.sh`) requires a bump on ANY
`columns.json` change so existing installs actually re-read the new columns.
This is schema-read only, distinct from `content.json` seed re-planting (no
`content.json` touched here, so no demo rows are resurrected on existing
installs — see the seed-replant SCAR this repo tracks separately).
`tool/check_compat.sh` needs no acknowledgement — additive only, nothing
removed/retyped.

**The bug**: native crawl-fetch failures (screenshot: 29 pending / 2 fetched
/ 10 FAILED, macOS v0.7.4). Reproduced live via a real curl burst against
`en.wikipedia.org/w/api.php?action=parse`: the first ~10 rapid, unthrottled,
identical anonymous requests return HTTP 200, every one after that returns
HTTP 429 (Wikimedia's edge/Envoy rate limiter — `x-envoy-ratelimited: true`,
plain-text body, `retry-after: 11`). The crawl drain in `wikiFetchLayer`
(wikipick_net.dart) fired up to 25 fetches back-to-back with zero spacing —
far faster than any browser session — so a batch tripped the limiter, and a
tripped fetch was marked `failed` PERMANENTLY: the wrong classification for
a transient server throttle.

**The fix**: (1) a minimum inter-fetch spacing in the drain loop
(`_kInterFetchDelayMs`, test-overridable via `debugWikiThrottleMs`); (2)
failure classification + in-run retry-with-backoff for TRANSIENT reasons
only (HTTP 429 / network exceptions — `_fetchArticleHtmlForCrawl`,
backoff schedule test-overridable via `debugWikiCrawlBackoff`); a genuine
404/missing-page response is still immediately `failed`, unchanged; (3) a
"Retry failed" queue-screen action (`wikiRetryFailedQueue`) that re-queues
transient-reason failed rows (never `notFound` rows) and kicks a resume
drain, respecting every existing budget/depth cap.

**Schema**: `crawl_queue` gains `failureReason` (enum, no default —
`rateLimited`/`networkError`/`notFound`, set only on a `failed` row, cleared
back to empty by "Retry failed"). `queueStatus` gains `failedCount` (integer,
default 0), recomputed alongside `pendingCount` everywhere that already
existed — drives the new button's visibility without needing per-value
label templating. Existing rows read the new columns as absent/0 via the
same `?? ''`/`?? 0` pattern every other optional column in this codebase
already uses — no backfill needed, no data lost.

### Tests covering this change

- `test/wikipick_crawl_throttle_test.dart` (new) — throttle spacing honored
  (real timestamps), 429 -> in-run retry -> success (budget charged once,
  not per attempt), retries-exhausted -> failed with reason (`rateLimited`,
  `networkError`), `notFound` never retried (one call only, matching the
  pre-existing correct behavior for that class), and the "Retry failed"
  action (re-queues transient rows, leaves `notFound` alone, no-op when
  nothing failed). Real network never touched — `debugWikiFetchClient`
  (`package:http/testing.dart` `MockClient`), the same seam
  `wikipick_fetch_jump_test.dart` already established.
- Full existing wikipick suite re-run green, including
  `wikipick_crawl_scale_test.dart` (its 5000-row drain is dedup-hit only, so
  it never reaches a real fetch and is unaffected by the new throttle) and
  `applet_lint_all_test.dart`/`schema_compat_test.dart` (new column +
  `wikiRetryFailedQueue` action registered and lint-clean).

## 2026-08-20 — W7 images + cache depth (additive only)

seedVersion `2026-08-19.seedsaved-flag.1` -> `2026-08-20.w7-images-depth.1`.

**Additive, non-breaking** — no table/column removed or retyped, so
`tool/check_compat.sh` needs no acknowledgement; this entry is a record, not
an audit trail for a break. New: `articleImages` table (`articleId`, `mime`,
`b64`, `w`, `h`, `bytes` — one lead-image payload per article, per the W2
design of record) and `saveOptions` table (single-row scratch: `includeImages`
default false, `cacheDepth` default 1 — the per-save controls). `articles`
gains `imageBytes` (default 0) and `totalBytes` (default 0, = extractBytes +
imageBytes). Existing rows read these as 0/absent via the same `?? 0` pattern
every other optional column in this codebase already uses — no backfill
needed, no data lost. `crawls.depthCap` (already existed) is now genuinely
driven by the per-save "Cache depth" choice instead of always being hardcoded
to 1.

## 2026-08-18 — timestamp columns `date` -> `dateTime` (polish1-0818, finding 2)

seedVersion `2026-08-18.home.1` -> `2026-08-18.datetime-fix.1`.

**Acknowledged BREAKING schema change** (retype): `tool/check_compat.sh`
reports six columns retyped `date -> dateTime`, and this file is the audit
trail that accepts it. The applet id (`wikipick`) and store namespace
(`isan_wikipick`) are UNCHANGED.

### The bug this fixes

A polish-loop tester round reported the reader's freshness badge reading
"3h ago / Cached" for an article saved seconds earlier in the same live
session — reproduced twice (immediately after save, and again after a full
reload), and re-reproduced live during this fix round ("5h ago" immediately
after a fresh save). It looked like a UTC-vs-local timezone bug in the
relative-time formatting. It was not.

**Root cause:** `articles.fetchedAt`, `articles.lastCheckedAt`,
`crawl_queue.discoveredAt`, `crawls.startedAt`, `liveArticle.fetchedAt` and
`recentReads.when` were all declared `"type": "date"`, but every write site
(`wikipick_net.dart`) stores a full `DateTime.now().toIso8601String()`
timestamp. The ENGINE's own column coercion (`datakit`'s `coerceValue`,
`DataType.date` case) silently truncates any value written to a `date`
column down to its bare calendar day (`_ymd`, e.g. `"2026-08-18"`) on every
`addRow`/`setField` — the time-of-day is dropped before it ever reaches
storage. The reader's "Cached" `stat` (table `liveArticle`, `format:
"relative"`) then parses that bare date string as **midnight local time**
and diffs it against the real `DateTime.now()` — so the badge always shows
"however many hours have passed since midnight today", regardless of when
the save actually happened. That number happens to look exactly like a
timezone-offset artifact (a few hours, non-zero, seemingly arbitrary),
which is why it read as a UTC/local bug rather than a column-type bug.

### The fix

Retype all six columns to `"dateTime"` (`coerceValue`'s `DataType.dateTime`
case keeps the full ISO-8601 string, time-of-day included) — no code change
needed in `wikipick_net.dart`; it was always writing correct full timestamps,
they were just getting truncated on the way into storage. A new unit test
(`test/wikipick_freshness_test.dart`) locks the exact failure mode: writes a
known "now", asserts `relativeTime` reads back "just now" (not several hours
off), and separately asserts a `date`-typed column WOULD truncate (documenting
why the retype, not a formatting tweak, was the fix).

### What happens to existing data

**Not lossy, but historically imprecise for rows saved before this fix.** A
`dateTime` column still parses a bare `"2026-08-18"` string fine (`DateTime.
tryParse` reads it as local midnight) — no row disappears, no value is
rejected. An article saved *before* this fix keeps showing its (wrong,
midnight-anchored) freshness badge until it is re-fetched or re-checked
("Cache linked pages" / "Check for updates" both call `DateTime.now().
toIso8601String()` again and overwrite the column with a real timestamp).
No migration code re-writes old rows — the bug was cosmetic (a mis-reported
"how long ago", nothing computed off it gates a real feature: `staleDays`
freshness banners work off `lastCheckedAt` too, so a pre-fix install may show
"stale" a little early/late until its next check, never destructively).

### Tests covering this change

- `test/wikipick_freshness_test.dart` (new) — the falsifying regression:
  saves a known "now", asserts the reader's relative-time reads "just now"
  through the real `relativeTime` helper against a real `dateTime`-coerced
  value; a second case demonstrates the OLD `date`-typed coercion actually
  truncating (proving the root cause, not just patching the symptom).
- `test/wikipick_reading_test.dart`, `test/wikipick_save_persistence_test.dart`,
  `test/wikipick_queue_resume_test.dart` — full existing wikipick suite
  re-run green against the retyped schema (no other behavior changed).

## 2026-08-20.perf-scale.1 — crawl-scale performance fix + cosmetic labels

Additive-only schema change: `depthCap` (integer, default 1) added to
`articles` and `liveArticle`. Pre-existing rows read the default (1),
matching the pre-W7 always-depth-1 behavior — no data is reinterpreted or
lost, and no migration code is needed.

### What changed

- **Perf**: `wikiFetchLayer`'s crawl-resume drain, `_commitSeedSave`'s
  outgoing-link/crawl bootstrap, and `_deleteArticleRefCounted`'s delete +
  cascade + closing sweep now batch their mutations through
  `BoardState.runAsSingleUndo` instead of leaving every `addRow`/`setField`/
  `deleteRow` call to snapshot the whole database independently. A resume
  against a crawl_queue that had grown into the thousands (a hub article's
  depth-2 discovery) used to freeze the tab for 1-3 minutes; see
  `wikiFetchLayer`'s doc comment for the full root-cause writeup.
  `wikiFetchLayer` also replaces its old O(table size) `_findArticle`/
  `_hasMembership` linear scans with in-memory Sets/Maps built once per
  drain and kept in sync with rows the run itself adds.
- **Cosmetic**: the reader's "Cache linked pages" button used to say
  "— 1 layer" unconditionally, even for a depth-2 save; it now reads the
  article's actual `depthCap` (two showIf-gated button variants, since the
  engine's button label has no per-value templating). The Library/home
  caption's stale "text only, no images" line (pre-W7 wording) is updated to
  "images are optional per save" across en+ko.

### What happens to existing data

Not lossy. `depthCap` defaults to 1 for every row saved before this change
(matching what those rows' crawls actually did, pre-W7), so no article's
apparent behavior changes; the corrected label simply becomes accurate going
forward for any NEW depth-2 save.

### Tests covering this change

- `test/wikipick_crawl_scale_test.dart` (new) — synthesizes a 5000-row
  pending queue + 500-article library and asserts a resume drain and a full
  delete each complete under a generous CI-safe time budget; guards against
  the O(rows × db size) regression this round fixes.
- Full existing wikipick suite re-run green against the additive schema (no
  other behavior changed) — delete-arc adversarial semantics (five prior
  rounds) preserved exactly; only the batching/notification-timing changed,
  not what gets purged/kept.
