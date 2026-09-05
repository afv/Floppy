# Performance and reliability review — September 2026

Work is isolated on `codex/performance-reliability-pass`, starting at `2bd503d70`
on `latest`. Existing edits in the original checkout are excluded. Each change
is intended to be reviewable and usable independently; nothing is merged or deployed.

## Keep statistics visible during refresh

- Problem: invalidating a warm statistics page could replace it with an empty
  page while its background job waited to run.
- Cause: invalidation deleted results unless a worker already held a refresh
  lock, even though readers already support serving a previous result.
- Change: advance the existing history version and retain the payload. Readers
  schedule a refresh and the completed job replaces it.
- Alternative: another fallback cache would duplicate an existing capability.
  Removing loading indicators alone would conceal missing data.
- Validation: existing statistics tests plus regressions for retained results,
  successful replacement, and unavailable workers. Final counts recorded below.
- Limit: the previous result may briefly lag edits; it is marked stale. This
  does not create results on first use or preserve Redis data after a flush.

## Load more home cards without rebuilding other shelves

- Problem: asking for another page of one shelf rebuilt unrelated home rows.
- Cause: the request built all groups before selecting the requested row.
- Change: pass the existing single-row filter for HTMX append requests.
  Initial page and ordinary navigation retain their existing behavior.
- Evidence: the regression fails on baseline because another shelf is touched;
  with the fix it returns the requested cards without touching that shelf.
  This removes work proportional to unrelated shelves; no wall-clock speedup
  is claimed without profiling a populated installation.
- Validation: all 28 home-view tests and scoped Ruff passed.
- Alternative: another cache is unnecessary; the builder already supports
  selecting a row before it performs expensive work.
- Follow-up: a cold individual shelf still materializes and sorts matching
  library entries before pagination. Optimizing that requires preserving
  smart filters, provider deduplication, and heterogeneous sorting.

## Review coverage and follow-ups

The review follows Django request views, shared media helpers, Redis range/day
caches, database-backed Discover rows, Celery scheduling, and existing tests.
The repository already contains substantial performance work; this pass builds
on it rather than replacing the architecture.

Open PRs were inspected through GitHub. On September 4 (US Central), the open
list contained #1080, #1041–#1035, #1031, #993, and #760; none was explicitly a
performance PR. Performance changes #653 (first-run context queries), #920
(compact media ordering), and #833 (bounded cache polling) are already merged.
No PR was merged or modified by this pass. The open show-status, season, and
TMDB-credit changes need their own correctness reviews before adoption; these
patches do not imply approval of them.

High-value remaining leads, not established fixes:

- Statistics highlight image normalization explicitly permits provider access
  while serving cached results. Existing #211 regression tests require that
  behavior to recover landscape artwork after Redis loss. Moving it off the
  request path needs a replacement artwork refresh path, not just disabling it.
- History uses indexed day payloads and targeted invalidation; Discover has
  durable row caches but Redis tab snapshots. Cold starts and queue backlogs
  need realistic large-library/worker measurements before promising that every
  loading banner can disappear.
