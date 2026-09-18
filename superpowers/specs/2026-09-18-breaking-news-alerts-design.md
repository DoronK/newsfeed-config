# On-device breaking-news alerts (roadmap #7) — v1 design

## Purpose

Opt-in local notifications when the user's followed categories/region have a
fresh story that 3+ distinct outlets are covering simultaneously — with no
server, no push backend, and nothing sent off-device. Every competitor with
breaking-news alerts (Inoreader, FeedSpot, NewsBreak, the generic
"Breaking News" apps) ships them via a server push pipeline (FCM/APNs); this
is the roadmap's deliberate bet that the gap is closable client-side. The
detection core (headline clustering over already-fetched articles) is also
the first real groundwork for roadmap #1 (cross-source story clustering /
"Full Coverage").

Honesty constraints carried from the roadmap write-up and the approved
design: this polls every ~15 min via WorkManager, so it is **not real-time**
— that is stated plainly in the Settings copy rather than glossed over.
Android-only for v1; iOS is parked with the rest of the iOS backlog (the
platform-facing surface sits behind interfaces so resuming iOS doesn't
require re-design).

Decisions locked during brainstorming (2026-09-18):

- **Detection = multi-source cluster** (headline/keyword-overlap clustering
  with distinct-outlet + freshness gates), not category-velocity and not a
  wires-only tripwire.
- **Opt-in from Settings, default-off.** The OS `POST_NOTIFICATIONS` prompt
  is triggered only by an explicit user tap on the toggle — never cold at
  first launch (a "Don't allow" on a fresh install poisons future
  re-prompts on Android 13+).
- **Approach A — standalone `BackgroundJob` with its own fetch.** The job
  registry in `core/background/` + `background_main.dart` was built by #5
  with exactly this second-job extension in mind. Accepted cost: when both
  prefetch and alerts are enabled, followed feeds are fetched twice per
  ~15-min window (both jobs' fetches are the same order of cost, concurrent
  per-target). Unifying the fetches is deferred until data usage is a real
  complaint.

## Architecture

New feature module `lib/features/breaking_news/`, mirroring the
`offline_prefetch/` layout:

| Unit | Role |
|---|---|
| `domain/breaking_news_detector.dart` | **Pure** clustering + trigger logic — no I/O, no Flutter imports. `detect(articles, ledger, config, now)` → `List<BreakingNewsStory>` |
| `domain/breaking_news_story.dart` | Value object: representative headline, outlet names + count, `link` (the freshest-unread member to open) |
| `domain/alert_notifier.dart` | Interface: `Future<bool> areNotificationsEnabled()`, `Future<void> show(BreakingNewsStory story)` — faked in tests |
| `domain/breaking_news_config.dart` | All v1 thresholds in one `const` class (see Detection) |
| `data/alert_ledger.dart` | Pure ledger over storage primitives: record/suppress/prune alerted stories, rate-limit windows (see Dedup & rate limits) |
| `data/flutter_local_alert_notifier.dart` | The **only** file importing `flutter_local_notifications` — same isolation pattern as `WorkmanagerBackgroundScheduler` (the only file importing `workmanager`) |
| `data/breaking_news_job.dart` | The `BackgroundJob` (`id: breaking_news_check`): enabled → permission → targets → fetch → detect → notify → persist |
| `presentation/breaking_news_tile.dart` + providers | Settings tile + toggle + status row, modeled on `OfflinePrefetchTile` |

Touch points outside the module (each a line or three, no structural
change):

- `background_main.dart`: add `BreakingNewsJob` to the `_registry` list —
  the extension point #5's build designed for a second job. No new
  `@pragma('vm:entry-point')` and no new native entry point: the job rides
  the existing `callbackDispatcher`, so the R8/obfuscation surface is
  unchanged from the #5 verification (2026-09-18).
- `main.dart`: cold-start re-assertion gains a sibling clause to
  `main.dart:38`'s prefetch one — re-`schedulePeriodic` when the toggle is
  on (WorkManager periodic tasks are re-registered with
  `ExistingPeriodicWorkPolicy.update`, so this is idempotent and also
  repairs a user-toggled-off-then-on state across reinstalls).
- `AndroidManifest.xml`: add `<uses-permission
  android:name="android.permission.POST_NOTIFICATIONS"/>`. No exact-alarm,
  no foreground-service permissions.
- `pubspec.yaml`: add `flutter_local_notifications`.

### Job run, step by step

1. `ctx.storage.breakingNewsEnabled == false` → `return true`.
2. `notifier.areNotificationsEnabled() == false` → `return true` (skip the
   fetch entirely — saving the network/battery is the point; alerts resume
   on re-grant without any re-scheduling step, because the periodic task
   was never cancelled).
3. Resolve targets exactly like `PrefetchJob`: `feedTargetsFor` over
   `CatalogCache` categories/regions + `lastResolvedCountryCode` (with the
   same device-locale fallback) + custom sources; muted sources excluded
   at fetch time via `excludedSourceUrls`.
4. Fetch each target through `ArticleRepositoryImpl.fetchArticlesForSources
   (forceRefresh: true)`, per-target try/catch, concurrently (same shape as
   `PrefetchJob.run` — one target failing can't sink the run); merge via
   `repository.mergeAndSort`.
5. `detect(...)` over the merged list.
6. For each story to alert (rate-limited, freshest first): `show(story)`;
   only after a successful `show()` record it in the ledger. If `show()`
   throws, do **not** record, and return `false` → WorkManager retries with
   backoff; detection re-runs and the un-recorded story is eligible again.
7. Write `breaking_news_last_check_at` (drives the Settings status row);
   return `true`.

Zero fetched articles → step 5 trivially yields nothing, write the
last-check timestamp, return `true` (nothing to check is success, not
failure).

## Detection heuristic (v1)

All thresholds live in `BreakingNewsConfig` as named constants:

| Knob | v1 value | Meaning |
|---|---|---|
| `lookbackWindow` | 12 h | Articles with `publishedAt` older than this are invisible to detection. Articles with **no** timestamp are excluded entirely (freshness can't be verified). |
| `jaccardThreshold` | 0.5 | Headline token-set similarity at/below which two articles are different stories |
| `minDistinctOutlets` | 3 | The multi-source gate — fewer outlets, no matter how fresh, is not "breaking" |
| `maxNewestAge` | 2 h | The cluster's newest article must be younger than this ("breaking now", not "still being covered") |
| `ledgerTtl` | 24 h | Alerted-cluster memory |
| `ledgerCap` | 40 entries | Ledger upper bound |
| `minGapBetweenAlerts` | 60 min | Global quiet gap between any two notifications |
| `rollingWindowCap` | 5 per 24 h | Hard noise ceiling |
| `maxAlertsPerRun` | 2 | Freshest-first within a single wake |

Algorithm (pure, deterministic given `(articles, ledger, now)`):

1. **Filter**: keep articles with `publishedAt != null` and age ≤
   `lookbackWindow`.
2. **Tokenize** each headline: lowercase, strip punctuation, drop tokens
   from a small English stopword list, drop length-1 tokens. (Cross-language
   note: local Hebrew sources tokenize fine — same-event headlines share
   proper nouns regardless of script; v1 deliberately ships without a
   Hebrew stopword list. A Hebrew-heavy cluster's residual stopwords raise
   the token floor uniformly, which the 0.5 threshold absorbs. If real-world
   precision is poor, tuning this config — not restructuring — is the
   expected first move.)
3. **Cluster**: greedy union-find over pairwise Jaccard similarity of
   headline token sets ≥ `jaccardThreshold`.
4. **Gate** each cluster: distinct outlets ≥ `minDistinctOutlets`, where
   **outlet = distinct host of the source RSS URL** (not `sourceName` —
   "BBC Technology" and "BBC Business" are one outlet, and the same outlet
   recurs across category/region/custom targets). Newest member age <
   `maxNewestAge`.
5. **Suppress** a qualifying cluster if: a ledger entry with the same
   signature is alive, or every member's link is in `readLinks` (read-only
   access — allowed by the background write rule; the *linked* article is
   the freshest member whose link is **not** in `readLinks`; if none, the
   cluster suppresses).
6. **Rank** survivors by newest-member age (freshest first), take
   `maxAlertsPerRun`.

Cluster signature (the dedup identity): hash of the cluster's 5 most
frequent tokens, sorted, joined. Chosen over hashing member links because
member sets churn as feeds scroll old items out; two unrelated stories
sharing their top-5 non-stopword tokens is rare, and the failure mode is a
missed duplicate alert (acceptable), not a false alert.

## Dedup & rate limits — state ownership

The alert job obeys #5's background-isolate write rule: it writes **only**
its own keys and never touches `read_article_links`, `read_events`,
`muted_source_urls`, `custom_sources`, `bookmarked_articles` (reads are
fine — `readLinks` and muted are read in step 3/5). New
`LocalStorageService` keys, flat `breaking_news_*` style matching
`offline_prefetch_*`:

| Key | Shape |
|---|---|
| `breaking_news_enabled` | bool (foreground writes it, background reads) |
| `breaking_news_alert_ledger` | JSON list of `{signature, alertedAt}` — TTL-pruned and size-capped on every write |
| `breaking_news_alert_times` | JSON list of notification timestamps (rolling 24 h cap window + 60-min gap check) |
| `breaking_news_last_check_at` | ISO timestamp for the Settings status row |

Ledger/rate-limit state lives behind `AlertLedger`, which takes the
storage service and exposes intent-revealing operations
(`wasAlerted`, `record`, `canNotifyNow`, `recordNotification`) — the
key-level JSON (de)serialization stays inside it, and the job never parses
these blobs itself.

## Notification & platform surface

- **One channel**, id `breaking_news`, name "Breaking news",
  importance `IMPORTANCE_HIGH` (heads-up present; the user can demote the
  channel at OS level — the app doesn't fight them). Channel description
  states the ~15-min polling cadence.
- **Content**: title = representative headline (the linked article's);
  body = `"<Outlet A>, <Outlet B> +N more reporting"` (two outlet names
  then "+N more"). No category label — a cluster legitimately spans
  targets (World + Politics covering the same election), and `Article`
  carries no category, so any label would be a guess. All text
  feed-derived and plain-text — same trust posture as feed-rendered UI.
- **Tap routing**: payload carries the article link. Warm app →
  `onDidReceiveNotificationResponse`; cold start →
  `getNotificationAppLaunchDetails` at startup. Both funnel into the same
  open-article-for-link flow `app.dart` already runs for
  `newsfeed://article?link=…` widget taps (`app.dart:56`), including
  disk-cache resolution. No new deep-link scheme, no native intent-filter
  changes.
- **Permission**: `POST_NOTIFICATIONS` is Android 13+ runtime; requested
  via the plugin's `requestNotificationsPermission()` from the Settings
  toggle tap only. Android < 13 auto-grants. The job's step-2
  `areNotificationsEnabled()` check covers post-hoc revocation from OS
  settings.
- **Posting from the background isolate**: the notifier initializes its
  own plugin instance inside the job's isolate (application context — no
  Activity needed to post), mirroring how #5 proved background-isolate
  plugin use (home_widget writes) works in this app.

## Settings & permission UX

New Settings section, modeled on the offline-prefetch tile:

- **Toggle** "Breaking news alerts", default off. Toggling on →
  `requestNotificationsPermission()`; granted → persist enabled +
  `schedulePeriodic` (any-network — no Wi-Fi-only switch in v1: this job
  fetches small XML, no images, and timeliness is the point); denied →
  toggle reverts + inline hint. Toggling off → `cancel(jobId)` +
  persist disabled.
- **Copy** (honesty constraint): one-liner under the toggle — *"Checks
  your followed topics every ~15 min and tells you when 3+ outlets break
  the same story. Not real-time."*
- **Status row**: "Last checked …" from `breaking_news_last_check_at`,
  same trust pattern as prefetch's "Saved 2m ago · N articles".
- **Revoked-permission surface**: the tile also queries
  `areNotificationsEnabled()` when built; if the toggle is on but the OS
  permission was since revoked (via system settings), it shows a hint
  ("Notifications are off in system settings") instead of the status row
  — the background already self-suspends on the same check (job step 2),
  so the UI just explains what the user observes.
- **Scope** is automatic and not configurable in v1: all followed
  categories + resolved local region + custom sources, muted excluded.
  Per-category alert granularity is a non-goal.

## Error handling

- Per-target fetch failure: isolated (prefetch pattern) — one dead feed
  degrades that target only.
- Total fetch failure (every target failed): `return false` → WorkManager
  backoff retry. Safe: nothing was alerted, nothing recorded; the retry
  re-detects from scratch.
- `show()` throws: story not recorded, `return false` → retry (see job
  step 6).
- Detector is total: malformed articles (null timestamps, empty titles,
  null URLs) are filtered, never throw; an exception escaping `detect`
  would fail the job via the existing `background_main.dart` catch, but
  the contract is that it doesn't.
- Permission revoked between runs: step 2 short-circuits to `true`; no
  network spend. Re-granting in OS settings resumes alerts on the next
  natural wake — no app open required.
- Stale/unregistered task names: handled by the existing
  `background_main.dart` unknown-job ack.

## Testing

- **Detector unit tests** (pure — the bulk of the confidence): clusters
  merge at/above 0.5 and split below; same-outlet-two-feeds counts once
  (host identity); freshness gates (old cluster, stale-newest); ledger
  suppression incl. signature stability across member churn; read-links
  suppression and freshest-unread link choice; per-run max; tokenization
  edge cases (punctuation, case, stopwords, Hebrew-script headline
  clustering on shared proper nouns).
- **AlertLedger unit tests**: TTL pruning, cap eviction, gap/cap window
  math (boundary-inclusive), JSON round-trip.
- **Job tests** with fake notifier/ledger: disabled short-circuit;
  permission-denied short-circuit (no fetch — assert via fake
  repository/interceptor); happy path posts ≤2, records ledger only after
  successful `show()`; rate-cap path; total-failure returns `false`;
  writes confined to `breaking_news_*` keys (same key-assertion approach
  as PrefetchJob's write-rule tests).
- **Settings tile widget test**: toggle on → permission requested →
  granted schedules / denied reverts + hint; toggle off cancels;
  toggle-on-but-OS-permission-revoked renders the system-settings hint.
- **`FlutterLocalAlertNotifier`**: untested, like
  `WorkmanagerBackgroundScheduler` — a thin plugin shim, faked at the
  interface everywhere else.
- **Manual/device verification** (mirrors #5's close-out): obfuscated
  release build on an emulator — toggle on, force-feed a synthetic
  multi-source story, confirm the heads-up fires from a background wake
  and the tap routes into the article screen cold and warm.

## Non-goals (named so they're deferred, not forgotten)

- **iOS** — parked with the standing iOS backlog (`AlertNotifier` +
  scheduler interfaces are the only seams iOS resumption touches; the
  roadmap's honest-copy requirement about iOS's unreliable background
  timing applies to any future iOS enablement).
- Per-category alert picking / alert-intensity settings.
- In-app quiet hours — the OS channel settings + Do Not Disturb own this;
  a second, app-level control would be worse at the same job.
- Story clustering in the *feed UI* — that's roadmap #1; this ships only
  the detection core, deliberately shaped for reuse there.
- Any server/push component, analytics on alert engagement, or custom
  notification views.
- Wi-Fi-only scheduling for this job.
