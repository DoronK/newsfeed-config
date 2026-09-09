# Proactive Offline Pre-fetch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship roadmap #5 — a Wi-Fi-by-default, opt-out Android background
job that keeps the next ~20 unread articles (full body + lead image) ready
offline — built on a reusable background-task plumbing layer that a future
PR (#7) registers a second job into.

**Architecture:** `core/background/` (platform-agnostic scheduler + job +
registry interfaces) + `lib/background_main.dart` (the background-isolate
composition root, mirroring `main.dart`) + `features/offline_prefetch/`
(the actual `PrefetchJob`, its Settings toggle, and its UI). Text prefetch
is almost entirely a scheduling problem, not a fetching problem —
`ArticleRepositoryImpl.fetchArticlesForSources` already writes full
article bodies through to disk on every call; `PrefetchJob` just calls it
on a timer for every followed cache key, resolved via a new pure
`feedTargetsFor()` function shared with the foreground providers (no
Riverpod in the background isolate). Image prefetch is a separate, later
task behind its own `ImagePrefetcher` abstraction, gated on a
foreground-liveness heartbeat to avoid two `CacheManager` instances
fighting over the same disk cache concurrently.

**Tech Stack:** Flutter, `flutter_riverpod` (foreground only),
`workmanager` (new dependency — Android), `cached_network_image`/
`flutter_cache_manager` (already present, reused for image warming).
Tests use `flutter_test` + fakes, matching this repo's existing pattern of
never hitting a real platform channel or network call in a test.

**Spec:** `docs/superpowers/specs/2026-09-08-proactive-offline-prefetch-design.md`
— every code block referenced below (`«see spec: <section>»`) is written
out in full there; this plan sequences it into tasks and tests rather than
re-deriving it.

## Global Constraints

- **Android only in this PR.** No `Info.plist`/`AppDelegate.swift`/
  `BGAppRefreshTask` changes — iOS is a tracked follow-up (see spec
  Non-goals), same shape as roadmap #4's Android-first split.
- **No Riverpod inside the background isolate.** `PrefetchJob`,
  `CatalogCache`, and `feedTargetsFor` all take plain constructor/function
  arguments — never `ref.read`/`ref.watch`. Only
  `offline_prefetch_providers.dart` (foreground) is Riverpod-aware.
- **The background-isolate write rule** (spec, Task 1 section): `PrefetchJob`
  only ever writes article-cache keys (via `fetchArticlesForSources`) and
  its own `offline_prefetch_last_run_*`/`foreground_heartbeat_at` keys.
  Never `read_article_links`, `read_events`, `muted_source_urls`,
  `custom_sources`, or `bookmarked_articles` — those are foreground
  read-modify-write keys; writing them from the background isolate risks
  silently clobbering a concurrent foreground write. This constraint gets
  a doc comment on `PrefetchJob` itself, not just a mention here.
- **`_kMaxDiskCachedArticles = 60` (`article_repository_impl.dart:30`)
  is not renegotiated.** Any "next N" budget (image prefetch) is defined
  within that cap (N ≈ 20), never by raising it.
- **This PR ships zero notification code.** No `flutter_local_notifications`,
  no `POST_NOTIFICATIONS` permission, no notification UI — that's #7,
  entirely deferred.
- **Task 1's provider refactor must not change observable behavior.**
  `forYouFeedProvider`/`forYouDiskCacheProvider`/`allArticlesFetchProvider`/
  `allArticlesDiskCacheProvider`'s existing tests are the regression gate —
  if any of their assertions need to change to pass, the refactor changed
  behavior and that's a bug, not a test update.

---

### Task 1: Prerequisite refactors (persisted country, `feedTargetsFor`, `CatalogCache`)

**Files:**
- Modify: `lib/core/storage/local_storage_service.dart`
- Modify: `lib/core/providers/location_provider.dart`
- Create: `lib/features/feed/domain/feed_targets.dart`
- Create: `lib/core/catalog/catalog_cache.dart`
- Modify: `lib/features/feed/presentation/providers/feed_providers.dart`
  (refactor `forYouFeedProvider`, `forYouDiskCacheProvider`,
  `allArticlesFetchProvider`, `allArticlesDiskCacheProvider` onto
  `feedTargetsFor`)
- Modify (or create): `test/local_storage_service_test.dart`
- Create: `test/feed_targets_test.dart`
- Create: `test/catalog_cache_test.dart`
- Modify: `test/feed_providers_test.dart` (extend — regression coverage
  for the refactor; do not weaken existing assertions)
- Modify (if a test exists): `test/location_provider_test.dart` (or
  create one if `GeoCountryNotifier` has no dedicated test file yet —
  check first)

**Interfaces:**
- Produces: `LocalStorageService.lastResolvedCountryCode` (get/set),
  `.offlinePrefetchEnabled`/`.offlinePrefetchWifiOnly` (get/set, default
  `true`), `.prefetchLastRunAt`/`.prefetchLastRunCount` (get) +
  `.setPrefetchLastRun(DateTime, int)`, `.foregroundHeartbeatAt` (get) +
  `.touchForegroundHeartbeat()`, and a `.reload()` wrapper over the
  underlying `SharedPreferences.reload()`.
- Produces: `FeedTarget` + `List<FeedTarget> feedTargetsFor({...})`
  (pure — see spec, Task 1 section for the exact signature/body).
- Produces: `CatalogCache` (see spec, Task 1 section for the exact class).
- Consumed by: Task 3 (`PrefetchJob` uses `feedTargetsFor` + `CatalogCache`
  + the new storage keys), Task 4 (toggle notifiers read/write
  `offlinePrefetchEnabled`/`WifiOnly`; `app.dart`'s lifecycle observer
  uses `.reload()`/`.touchForegroundHeartbeat()`).

**Surface:** app

- [x] **Step 1: Write failing tests for the new `LocalStorageService` keys**

  In `test/local_storage_service_test.dart` (check whether this file
  already exists — extend it if so, matching its existing setup/style;
  create it with `SharedPreferences.setMockInitialValues({})` +
  `LocalStorageService.create()` if not), add cases for:
  - `lastResolvedCountryCode` is `null` before any set; round-trips a code.
  - `offlinePrefetchEnabled`/`offlinePrefetchWifiOnly` both default to
    `true`; each round-trips `false`.
  - `prefetchLastRunAt`/`prefetchLastRunCount` are both `null`/`null`
    before any run; `setPrefetchLastRun(dt, 18)` makes them round-trip
    (allow for UTC/ISO8601 precision — compare via
    `isAtSameMomentAs` or truncate to seconds, matching however this repo
    already compares `DateTime` in its other storage tests).
  - `foregroundHeartbeatAt` is `null` before any touch;
    `touchForegroundHeartbeat()` sets it to (approximately) now.
  - `reload()` doesn't throw against the mocked `SharedPreferences`
    backend used in this test file.

- [x] **Step 2: Run and confirm failure**

  `flutter test test/local_storage_service_test.dart` — FAIL, none of
  these members exist yet.

- [x] **Step 3: Implement the new `LocalStorageService` members**

  Add the constants and get/set pairs exactly as written in the spec's
  "Task 1 — prerequisites" section (`_kLastResolvedCountryKey` through
  `_kForegroundHeartbeatAtKey` and their accessors), plus:

  ```dart
  Future<void> reload() => _prefs.reload();
  ```

- [x] **Step 4: Run and confirm pass**

  `flutter test test/local_storage_service_test.dart` — PASS.

- [x] **Step 5: Write failing test(s) for persisted country in `GeoCountryNotifier`**

  (`test/geo_country_test.dart` already exists as this notifier's dedicated
  test file — extended in place rather than creating
  `test/location_provider_test.dart`.)

  Add (or extend, if `test/location_provider_test.dart` already exists —
  read it first and match its existing `ProviderContainer`/override
  pattern) a case: given a `LocalStorageService` pre-seeded with
  `lastResolvedCountryCode = 'IL'` and a locale fallback that differs
  (e.g. `deviceCountryCodeProvider` overridden to `'US'`), the notifier's
  initial `build()` value is `'IL'`, not the locale fallback — persisted
  beats locale-guessed. Also confirm (may already be covered by an
  existing test) that a successful `geoCountryResolverProvider` resolution
  still calls `setLastResolvedCountryCode` on the fake/mock storage — a
  `Mocktail`/manual fake with a verifiable call, or a real
  `LocalStorageService` backed by mocked `SharedPreferences` whose value
  you re-read afterward, whichever pattern this repo's other provider
  tests already use.

- [x] **Step 6: Run and confirm failure**

  Expected: FAIL — `build()` doesn't read `lastResolvedCountryCode` yet,
  `_resolveOnce` doesn't persist it yet.

- [x] **Step 7: Update `GeoCountryNotifier`**

  Exactly as written in the spec's Task 1 section: `build()` reads
  `ref.watch(localStorageServiceProvider).lastResolvedCountryCode` and
  prefers it over the locale fallback; `_resolveOnce()` calls
  `ref.read(localStorageServiceProvider).setLastResolvedCountryCode(code)`
  right after `state = code`.

- [x] **Step 8: Run and confirm pass**

  `flutter test test/location_provider_test.dart` (or wherever these
  live) — PASS, and confirm no other existing test in this file broke.

- [x] **Step 9: Write failing tests for `feedTargetsFor`**

  Create `test/feed_targets_test.dart` covering:
  - Two followed category ids that both exist in `categories` → two
    targets, cache keys equal to those ids, sources equal to each
    category's `sources`.
  - A followed id that does **not** exist in `categories` (stale/removed
    remotely) → silently dropped, matching
    `followedCategoryIdsProvider`'s existing behavior — no throw.
  - `countryCode` resolving to a known region → one target with
    `cacheKey == localFeedCacheKey(region.countryCode)` and that region's
    `sources`.
  - `countryCode` null, or not in `regions` → one target using
    `kGlobalNewsRegion` (`cacheKey == localFeedCacheKey('GLOBAL')`) — the
    local target is **never absent**, even with zero followed categories.
  - `customSources` empty → no custom-feed target at all (not a target
    with an empty `sources` list); non-empty → exactly one target with
    `cacheKey == kCustomFeedCacheKey`.
  - Zero followed categories + empty custom sources → exactly one target
    (the local-region one) — never an empty list.

- [x] **Step 10: Run and confirm failure**

  Expected: FAIL — the file doesn't exist yet.

- [x] **Step 11: Implement `feed_targets.dart`**

  Exactly as written in the spec's Task 1 section (with one addition the
  spec's own code block omitted: `kCustomFeedCacheKey` isn't defined in
  either of the two imports the spec lists — it lives in
  `custom_sources_provider.dart` — so an explicit `show kCustomFeedCacheKey`
  import of that file was added).

- [x] **Step 12: Run and confirm pass**

  `flutter test test/feed_targets_test.dart` — PASS.

- [x] **Step 13: Write failing tests for `CatalogCache`**

  Create `test/catalog_cache_test.dart`: no cached JSON → returns
  `kDefaultInterestCategories`/`kDefaultLocalNewsRegions`; valid cached
  JSON → returns the parsed list (assert on a specific field, e.g. a
  category id, to confirm it's the parsed value and not the default);
  malformed JSON → falls back to defaults without throwing.

- [x] **Step 14: Run and confirm failure, then implement, then confirm pass**

  Implement `lib/core/catalog/catalog_cache.dart` exactly as written in
  the spec. Run `flutter test test/catalog_cache_test.dart` — PASS.

- [x] **Step 15: Refactor `feed_providers.dart`'s four duplicated compositions onto `feedTargetsFor`**

  No test in the existing suite directly exercised these four providers
  before this task (`feed_providers_test.dart` only covered the
  `withoutMuted`/`fetchCategoriesForIds`/`followedCategoryIdsProvider`
  helpers). Added a new regression group covering all four providers
  first, ran it against the **pre-refactor** implementation to confirm it
  passes as a baseline (`flutter test test/feed_providers_test.dart` — 9/9
  passed, including the `allArticlesDiskCacheProvider` vs.
  `forYouDiskCacheProvider` custom-feed-disk-cache asymmetry case), then
  refactored `forYouFeedProvider`, `forYouDiskCacheProvider`,
  `allArticlesFetchProvider`, and `allArticlesDiskCacheProvider` onto
  `feedTargetsFor(...)`. `allArticlesFetchProvider`/
  `allArticlesDiskCacheProvider` pass every category id as
  `followedCategoryIds`. `allArticlesDiskCacheProvider` passes
  `customSources: const []` to `feedTargetsFor` and separately keeps its
  unconditional `repository.readDiskCache(kCustomFeedCacheKey)` call, to
  preserve the pre-existing asymmetry vs. `forYouDiskCacheProvider`'s
  gated version (documented inline at its call site).

- [x] **Step 16: Run and confirm no regression**

  `flutter test test/feed_providers_test.dart` — PASS, all 9 assertions in
  the new regression group unchanged from the pre-refactor baseline.

- [x] **Step 17: Run the full suite so far**

  `flutter test` — PASS (115 tests).

- [x] **Step 18: Run static analysis**

  `flutter analyze` — no issues.

- [x] **Step 19: Commit**

  ```bash
  git add lib/core/storage/local_storage_service.dart lib/core/providers/location_provider.dart lib/features/feed/domain/feed_targets.dart lib/core/catalog/catalog_cache.dart lib/features/feed/presentation/providers/feed_providers.dart test/local_storage_service_test.dart test/feed_targets_test.dart test/catalog_cache_test.dart test/feed_providers_test.dart test/geo_country_test.dart docs/superpowers/plans/2026-09-08-proactive-offline-prefetch.md
  git commit -m "Prerequisite refactors for offline pre-fetch: persist resolved country, extract feedTargetsFor/CatalogCache"
  ```

  (Adjusted from the plan's original file list: `test/geo_country_test.dart`
  is this repo's existing dedicated test file for `GeoCountryNotifier` —
  extended in place instead of creating `test/location_provider_test.dart`.
  Also includes this plan file itself, since its checkboxes were updated in
  the same commit as the work they track.)

---

### Task 2: `core/background/` scaffolding + `workmanager` dependency

**Files:**
- Modify: `pubspec.yaml` (add `workmanager`)
- Create: `lib/core/background/background_job.dart`
- Create: `lib/core/background/background_job_context.dart`
- Create: `lib/core/background/background_job_registry.dart`
- Create: `lib/core/background/background_scheduler.dart`
- Create: `lib/core/background/workmanager_background_scheduler.dart`
- Create: `test/background_job_registry_test.dart`

**Interfaces:**
- Produces: `BackgroundJob` (abstract), `BackgroundJobContext`,
  `BackgroundJobRegistry`, `BackgroundScheduler` (abstract),
  `WorkmanagerBackgroundScheduler`.
- Consumed by: Task 3 (`PrefetchJob implements BackgroundJob`,
  `background_main.dart` builds a `BackgroundJobRegistry` and a
  `BackgroundJobContext`), Task 4 (`offline_prefetch_providers.dart`
  depends on `BackgroundScheduler`, provides
  `WorkmanagerBackgroundScheduler` as the real implementation).

**Surface:** app

- [x] **Step 1: Add the `workmanager` dependency**

  In `pubspec.yaml`, under `dependencies:`, add
  `workmanager: ^0.10.0` (check `flutter pub outdated`/pub.dev for the
  current latest 0.10.x at implementation time — architect's design pass
  confirmed 0.10.10 as of this session but that may have moved). Run
  `flutter pub get` and confirm it resolves with no conflicts against the
  existing `dio`/`shared_preferences`/`flutter_riverpod` constraints.

- [x] **Step 2: Implement `background_job.dart`, `background_job_context.dart`, `background_scheduler.dart`**

  Exactly as written in the spec's "Task 2/3 — background-task plumbing"
  section (three small files, no tests needed for the plain interfaces
  themselves — they have no behavior to test beyond what Step 4 below
  covers for the registry).

- [x] **Step 3: Write failing test for `BackgroundJobRegistry`**

  Create `test/background_job_registry_test.dart`: a registry built from
  two fake `BackgroundJob`s with distinct `id`s resolves each by its id
  via `registry[id]`; an unknown id returns `null`.

- [x] **Step 4: Run and confirm failure, implement, confirm pass**

  Implement `background_job_registry.dart` exactly as in the spec.
  `flutter test test/background_job_registry_test.dart` — PASS.

- [x] **Step 5: Implement `WorkmanagerBackgroundScheduler`**

  Exactly as written in the spec, **but first verify the exact
  `workmanager` API surface against the version pinned in Step 1** —
  `Workmanager().initialize(callback, {isInDebugMode})`,
  `registerPeriodicTask(uniqueName, taskName, {frequency, constraints,
  existingWorkPolicy, ...})`, `cancelByUniqueName(String)`,
  `Constraints(networkType: ...)`, `NetworkType.unmetered`/`.connected`,
  and — importantly — that `ExistingPeriodicWorkPolicy` (not
  `ExistingWorkPolicy`, a different type for one-off tasks) is the correct
  enum for `registerPeriodicTask`'s `existingWorkPolicy` parameter. Adjust
  parameter names to match the installed version's actual signature if
  dartdoc for the pinned version differs from the spec's sketch — this is
  flagged in the spec itself as unverified against a real compile.

  No dedicated unit test for this class — it's a thin wrapper over a
  plugin with a platform channel that can't run in `flutter test` without
  a real device/emulator; Task 4's `offline_prefetch_providers_test.dart`
  tests against the `BackgroundScheduler` *interface* via a fake, which is
  the actual behavioral contract that matters. This class gets exercised
  for real only by Task 3's manual on-device verification step.

- [x] **Step 6: Run the full suite**

  `flutter test` — PASS. `flutter analyze` — no new issues (in particular,
  confirm `WorkmanagerBackgroundScheduler` compiles against the real
  plugin API — a compile error here is the first real signal the spec's
  sketch drifted from the installed version).

- [x] **Step 7: Commit**

  ```bash
  git add pubspec.yaml pubspec.lock lib/core/background/ test/background_job_registry_test.dart
  git commit -m "Add core/background scheduling plumbing (workmanager-backed)"
  ```

---

### Task 3: `PrefetchJob`, `background_main.dart`, initial scheduling wiring

**Files:**
- Create: `lib/features/offline_prefetch/data/prefetch_job.dart`
- Create: `lib/background_main.dart`
- Modify: `lib/main.dart` (schedule the job on startup if already
  onboarded + enabled)
- Modify: `lib/features/onboarding/presentation/providers/interests_provider.dart`
  (`completeOnboarding` also schedules the job)
- Modify: `lib/app.dart` (`WidgetsBindingObserver` — reload +
  heartbeat on lifecycle changes)
- Create: `test/prefetch_job_test.dart`

**Interfaces:**
- Consumes: `BackgroundJob`/`BackgroundJobContext`/`BackgroundJobRegistry`
  (Task 2), `feedTargetsFor`/`CatalogCache` (Task 1).
- Produces: `PrefetchJob` (`static const jobId = 'offline_prefetch'`),
  `callbackDispatcher` (top-level, `@pragma('vm:entry-point')`, in
  `lib/background_main.dart`).
- Consumed by: Task 4 (`offline_prefetch_providers.dart` imports
  `callbackDispatcher` to construct `WorkmanagerBackgroundScheduler`;
  `offline_prefetch_tile.dart`'s manual "Update now" constructs a
  `PrefetchJob` directly).

**Surface:** app

- [x] **Step 1: Write failing tests for `PrefetchJob`**

  Create `test/prefetch_job_test.dart`. Build a `BackgroundJobContext`
  from an in-memory `LocalStorageService` (mocked `SharedPreferences`,
  `SharedPreferences.setMockInitialValues({...})`) and a `Dio` wired to a
  fake `HttpClientAdapter` (same pattern as
  `test/rss_remote_data_source_test.dart`'s `_FakeAdapter`, extended or
  reused to serve different XML per-URL if the fixture needs more than one
  source). Cover, per the spec's Testing section:
  - Given followed categories = `['technology']`, a custom source, and a
    resolved country in `regions`: the exact set of URLs fetched matches
    every source across all three targets, and a muted source's URL
    (`storage.mutedSourceUrls` pre-seeded) is excluded from the fetch set.
  - A followed category id that isn't in the catalog is silently skipped
    (no fetch attempted for it, no throw).
  - `offlinePrefetchEnabled = false` → `run()` returns `true` immediately,
    zero HTTP calls made (assert on the fake adapter's call count/log),
    and no `prefetchLastRunAt` write.
  - Zero followed categories (but local region always present) → still
    fetches the local-region target (per `feedTargetsFor`'s guarantee)
    and records a last-run.
  - Every source fetch fails (fake adapter throws) → `run()` still
    returns `true` (not a hard failure — see spec's error-state
    reasoning) and records `prefetchLastRunAt` with count `0`.
  - One source failing among several succeeding doesn't stop the others —
    assert the successful ones' articles are still counted.

- [x] **Step 2: Run and confirm failure**

  `flutter test test/prefetch_job_test.dart` — FAIL, `PrefetchJob`
  doesn't exist.

- [x] **Step 3: Implement `PrefetchJob`**

  Exactly as written in the spec's Task 2/3 section — text-only for now
  (no `ImagePrefetcher` parameter yet; that's added in Task 5 without
  breaking this constructor, since it'll be an optional named parameter
  defaulting to `null`).

- [x] **Step 4: Run and confirm pass**

  `flutter test test/prefetch_job_test.dart` — PASS.

- [x] **Step 5: Implement `lib/background_main.dart`**

  Exactly as written in the spec, registering `PrefetchJob()` in
  `_registry`. No unit test for `callbackDispatcher` itself (it's a
  `workmanager`-plugin entry point, exercised only by the manual on-device
  verification step below) — but do add one focused test confirming the
  registry lookup logic (`_registry[taskName]` returning the right job /
  `null` for an unknown name) if that logic is factored out testably;
  otherwise this is covered transitively by Task 2's
  `background_job_registry_test.dart`.

- [x] **Step 6: Wire initial scheduling into `main.dart` and onboarding completion**

  In `main.dart`, after `runApp` returns (or via a small helper called
  right after building `storage`, before/independent of `runApp` — either
  is fine as long as it doesn't block first paint): if
  `storage.hasOnboarded && storage.offlinePrefetchEnabled`, construct a
  `WorkmanagerBackgroundScheduler(callbackDispatcher)` and call
  `schedulePeriodic(jobId: PrefetchJob.jobId, interval: const
  Duration(minutes: 15), unmeteredOnly: storage.offlinePrefetchWifiOnly)`.
  This is a direct, non-Riverpod call (no `ProviderScope` exists yet at
  this point in `main()`) — do not try to route it through
  `offlinePrefetchEnabledProvider` here; that provider (Task 4) is for the
  Settings toggle's own runtime scheduling, this is cold-start
  re-assertion in case a previous cancel/schedule was interrupted.

  In `SelectedInterestsNotifier.completeOnboarding()`
  (`interests_provider.dart:20-23`), after
  `ref.read(hasOnboardedProvider.notifier).markOnboarded()`: if
  `ref.read(localStorageServiceProvider).offlinePrefetchEnabled` (true by
  default), call the same `schedulePeriodic` — either via
  `ref.read(backgroundSchedulerProvider)` if Task 4 already exists when
  this step runs (it doesn't yet in-order — see note below) or via a
  direct `WorkmanagerBackgroundScheduler(callbackDispatcher)` construction
  matching `main.dart`'s approach. **Since Task 4 (which defines
  `backgroundSchedulerProvider`) comes after this task in the plan's
  order, implement this step's onboarding-completion scheduling with the
  direct construction now, then in Task 4 Step 6 (below) revisit this one
  call site to route it through the provider instead** — leaving a
  `// TODO(Task 4): route through backgroundSchedulerProvider` comment
  here is acceptable and expected, not a shortcut to avoid.

- [x] **Step 7: Add the `WidgetsBindingObserver` to `app.dart`**

  Exactly as written in the spec's Task 4 section — `with
  WidgetsBindingObserver`, `addObserver`/`removeObserver` in
  `initState`/`dispose`, `didChangeAppLifecycleState` reloading storage +
  bumping the cache buster on resume, and touching the foreground
  heartbeat on both resume and pause.

  Add a widget test (extend `test/app_test.dart`) confirming: pumping
  `NewsfeedApp` and driving `didChangeAppLifecycleState(resumed)` (via
  `WidgetsBinding.instance` test hooks / `tester.binding` lifecycle
  simulation — match whatever mechanism this repo's existing tests
  already use for lifecycle, or `TestWidgetsFlutterBinding`'s
  `handleAppLifecycleStateChanged` if none does yet) results in
  `localStorageServiceProvider`'s `touchForegroundHeartbeat` having been
  called (assert via a fake/spy `LocalStorageService`, not the real
  `SharedPreferences`-backed one, to avoid a real platform-channel call in
  a widget test).

- [x] **Step 8: Run and confirm pass**

  `flutter test` — PASS (full suite, not just this task's new files —
  confirm the `app.dart` change didn't break any existing
  `app_test.dart`/`screens_smoke_test.dart` case that pumps `NewsfeedApp`).

- [x] **Step 9: Run static analysis**

  `flutter analyze` — no new issues.

- [x] **Step 10: Commit**

  ```bash
  git add lib/features/offline_prefetch/data/prefetch_job.dart lib/background_main.dart lib/main.dart lib/features/onboarding/presentation/providers/interests_provider.dart lib/app.dart test/prefetch_job_test.dart test/app_test.dart
  git commit -m "Add PrefetchJob, background isolate entry point, and initial job scheduling"
  ```

---

### Task 4: Settings toggle + scheduling wiring

**Files:**
- Create: `lib/features/offline_prefetch/presentation/providers/offline_prefetch_providers.dart`
- Modify: `lib/features/onboarding/presentation/providers/interests_provider.dart`
  (revisit Task 3 Step 6's TODO — route `completeOnboarding`'s scheduling
  through `backgroundSchedulerProvider`)
- Create: `test/offline_prefetch_providers_test.dart`

**Interfaces:**
- Consumes: `BackgroundScheduler`/`WorkmanagerBackgroundScheduler` (Task
  2), `PrefetchJob.jobId` + `callbackDispatcher` (Task 3).
- Produces: `backgroundSchedulerProvider`, `offlinePrefetchEnabledProvider`
  (+ `.setEnabled`), `offlinePrefetchWifiOnlyProvider` (+
  `.setWifiOnly`).
- Consumed by: Task 5 (`offline_prefetch_tile.dart` reads/writes both
  toggle providers and the status-row data).

**Surface:** app

- [x] **Step 1: Write failing tests against a fake `BackgroundScheduler`**

  Create `test/offline_prefetch_providers_test.dart`. Define a small fake
  implementing `BackgroundScheduler` that records every
  `schedulePeriodic`/`cancel` call (arguments included). Using a
  `ProviderContainer` with `backgroundSchedulerProvider` overridden to the
  fake and `localStorageServiceProvider` overridden to a real
  mocked-`SharedPreferences`-backed `LocalStorageService`:
  - `offlinePrefetchEnabledProvider.notifier.setEnabled(true)` calls
    `schedulePeriodic` exactly once with `jobId: PrefetchJob.jobId` and
    `unmeteredOnly` matching the current `offlinePrefetchWifiOnlyProvider`
    state; persists `true` to storage.
  - `setEnabled(false)` calls `cancel(PrefetchJob.jobId)` and does **not**
    call `schedulePeriodic`.
  - `offlinePrefetchWifiOnlyProvider.notifier.setWifiOnly(false)` while
    enabled → calls `schedulePeriodic` again with `unmeteredOnly: false`.
  - `setWifiOnly(...)` while **disabled** → does not call
    `schedulePeriodic` at all (nothing to re-register).

- [x] **Step 2: Run and confirm failure**

  `flutter test test/offline_prefetch_providers_test.dart` — FAIL.

- [x] **Step 3: Implement `offline_prefetch_providers.dart`**

  Exactly as written in the spec's Task 4 section.

- [x] **Step 4: Run and confirm pass**

  `flutter test test/offline_prefetch_providers_test.dart` — PASS.

- [x] **Step 5: Route `completeOnboarding`'s scheduling through the new provider**

  Replace Task 3 Step 6's direct `WorkmanagerBackgroundScheduler(...)`
  construction inside `SelectedInterestsNotifier.completeOnboarding()`
  with `ref.read(backgroundSchedulerProvider).schedulePeriodic(...)` (or,
  more simply, `ref.read(offlinePrefetchEnabledProvider.notifier)
  .setEnabled(true)` if the user's stored preference is already `true` by
  default — prefer this simpler call since it also keeps
  `offlinePrefetchEnabledProvider`'s own state consistent, rather than
  scheduling the job without the provider knowing it's "on"). Remove the
  `// TODO(Task 4)` comment.

- [x] **Step 6: Run and confirm pass**

  `flutter test` — PASS (full suite).

- [x] **Step 7: Run static analysis**

  `flutter analyze` — no new issues.

- [x] **Step 8: Commit**

  ```bash
  git add lib/features/offline_prefetch/presentation/providers/offline_prefetch_providers.dart lib/features/onboarding/presentation/providers/interests_provider.dart test/offline_prefetch_providers_test.dart
  git commit -m "Wire the offline pre-fetch toggle to schedule/cancel the background job"
  ```

---

### Task 5: Settings UI — toggle, Wi-Fi switch, status row

**Files:**
- Create: `lib/features/offline_prefetch/presentation/widgets/offline_prefetch_tile.dart`
- Modify: `lib/features/settings/presentation/screens/settings_screen.dart`
  (Data section)
- Create: `test/offline_prefetch_tile_test.dart`

**Interfaces:**
- Consumes: `offlinePrefetchEnabledProvider`/`offlinePrefetchWifiOnlyProvider`
  (Task 4), `LocalStorageService.prefetchLastRunAt`/`Count` (Task 1),
  `PrefetchJob`/`BackgroundJobContext` (Task 3, for the manual "Update
  now"/"Save now" in-process run), `timeAgo`
  (`lib/core/utils/date_format_utils.dart`, existing).
- Produces: `OfflinePrefetchTile` widget, dropped into
  `settings_screen.dart`'s Data section.

**Surface:** ui

- [x] **Step 1: Write failing widget tests for `OfflinePrefetchTile`'s states**

  Create `test/offline_prefetch_tile_test.dart` covering, per the spec's
  UI section: never-run state renders "Not saved yet…" + a "Save now"
  button; has-run state (storage pre-seeded with a `prefetchLastRunAt`/
  `Count`) renders "Saved {timeAgo} · N articles" + "Update now"; the
  nested Wi-Fi-only switch is disabled (`onChanged == null`) and carries a
  `Semantics(hint: ...)` when the parent toggle is off; tapping "Save
  now"/"Update now" shows the in-progress state (spinner, button
  disabled) while the run is in flight, then resolves to the has-run
  state on success or the failed state (with a "Try again" action) if the
  underlying fetch throws — inject a fake/failing `Dio`/repository path so
  this doesn't hit a real network call (match whatever seam Task 3's
  `PrefetchJob` test used to fake HTTP, or accept a constructor-injectable
  `PrefetchJob` instance as a test parameter on the tile if that's the
  cleaner seam — implementer's call, but the test must not make a real
  HTTP request).

- [x] **Step 2: Run and confirm failure**

  `flutter test test/offline_prefetch_tile_test.dart` — FAIL.

- [x] **Step 3: Implement `OfflinePrefetchTile`**

  Per the spec's UI section in full: `SwitchListTile` for the main
  toggle; a visually-nested (56dp left inset, no leading icon)
  `SwitchListTile` for Wi-Fi-only, with dynamic subtitle text and the
  disabled+`Semantics(hint:)` treatment when the parent is off; a status
  row (`Padding`+`Row`, not a `ListTile`) with the four states (never-run
  / has-run / in-progress / failed) described in the spec, using
  `timeAgo()` for the relative-time string, `Semantics(liveRegion: true)`
  wrapping the row so a screen reader announces state changes, and
  `ExcludeSemantics` around the row's leading icon. The "Save now"/"Update
  now" action runs `PrefetchJob().run(BackgroundJobContext(storage: ...,
  dio: ...))` in-process using the already-live foreground
  `localStorageServiceProvider`/`dioProvider` — decide (and record in this
  plan's Self-Review Notes at the end) whether a Wi-Fi-only + currently-
  metered situation gets a confirmation `AlertDialog` (matching the
  existing Clear-cache dialog pattern) before running, or whether the
  manual run simply proceeds and lets a failed fetch surface as the
  "failed" status state — the spec left this as an implementer's call
  rather than over-specifying it.

- [x] **Step 4: Run and confirm pass**

  `flutter test test/offline_prefetch_tile_test.dart` — PASS.

- [x] **Step 5: Wire `OfflinePrefetchTile` into `settings_screen.dart`'s Data section**

  Insert it above the existing "Clear cache" `ListTile`
  (`settings_screen.dart:96-131`), inside the same `_Card`, separated by
  the same `Divider` pattern already used between rows elsewhere on this
  screen (e.g. the "Your interests"/"Manage sources" pair at lines
  68-90). Keep the section label as "Data" (the spec records the
  "Offline" rename as a soft preference, not a requirement — do not churn
  the label unless it's a clean one-line change).

  Extend `test/screens_smoke_test.dart` (or add a focused test) asserting
  `SettingsScreen` renders the new tile's toggle without breaking any of
  the screen's existing assertions (e.g. that "Clear cache" is still
  present and still functions).

- [x] **Step 6: Run the full suite**

  `flutter test` — PASS.

- [x] **Step 7: Run static analysis**

  `flutter analyze` — no new issues.

- [x] **Step 8: Commit**

  ```bash
  git add lib/features/offline_prefetch/presentation/widgets/offline_prefetch_tile.dart lib/features/settings/presentation/screens/settings_screen.dart test/offline_prefetch_tile_test.dart test/screens_smoke_test.dart
  git commit -m "Add offline pre-fetch Settings UI (toggle, Wi-Fi switch, status row)"
  ```

---

### Task 6: Image prefetch (Phase 2)

**Files:**
- Create: `lib/features/offline_prefetch/domain/image_prefetcher.dart`
- Create: `lib/features/offline_prefetch/domain/prefetch_policy.dart`
- Create: `lib/features/offline_prefetch/data/cache_manager_image_prefetcher.dart`
- Modify: `lib/features/offline_prefetch/data/prefetch_job.dart` (add the
  optional `ImagePrefetcher?` parameter and the foreground-liveness-gated
  call)
- Modify: `lib/background_main.dart` (register `PrefetchJob` with a real
  `CacheManagerImagePrefetcher()`)
- Create: `test/prefetch_policy_test.dart`
- Modify: `test/prefetch_job_test.dart` (extend for the new parameter)

**Interfaces:**
- Produces: `ImagePrefetcher` (abstract), `CacheManagerImagePrefetcher`,
  `selectImageUrlsToWarm(...)` (or equivalent name — see spec's Phase 2
  section for the intended shape).
- Consumes: `safeArticleUri` (`lib/core/utils/safe_link.dart`, existing —
  reused for image-URL validation), `Article.hasFullContent`/`imageUrl`
  (existing).

**Surface:** app

- [x] **Step 1: Write failing tests for the pure selection policy**

  Create `test/prefetch_policy_test.dart` for the function described in
  the spec's Phase 2 section (`selectImageUrlsToWarm` or whatever name is
  chosen — pick one and use it consistently across this task): given a
  list of articles with a mix of read/unread links and valid/invalid/
  missing `imageUrl`s, and a `limit`, assert: already-read links (present
  in a passed-in `readLinks` set) are excluded; a `null`/`http`-invalid
  (e.g. `javascript:`/`file:`) `imageUrl` is excluded (reuse
  `safeArticleUri` for the check — assert this by testing a URL that
  `safeArticleUri` itself would reject, not by re-deriving the rule);
  results are capped at `limit`; results preserve newest-first order (the
  input is assumed already sorted, matching `mergeAndSort`'s output — the
  function itself doesn't re-sort).

- [x] **Step 2: Run and confirm failure, implement, confirm pass**

  Implement `prefetch_policy.dart`.
  `flutter test test/prefetch_policy_test.dart` — PASS.

- [x] **Step 3: Implement `ImagePrefetcher` + `CacheManagerImagePrefetcher`**

  `image_prefetcher.dart`: the interface as in the spec.
  `cache_manager_image_prefetcher.dart`: wraps
  `DefaultCacheManager().downloadFile(url)` per URL from `warm(urls)`,
  each call in its own try/catch (one bad URL must not abort the batch —
  assert this with a unit test using two URLs where one is deliberately
  malformed/unreachable via a fake `BaseCacheManager`/injected function
  rather than a real network call, if `flutter_cache_manager`'s API
  allows injecting one; if it doesn't cleanly support faking in a unit
  test, this class may go untested directly and instead be covered by
  `PrefetchJob`'s test in Step 5 below via a fake `ImagePrefetcher` — note
  which approach was taken in this plan's Self-Review Notes).

- [x] **Step 4: Add the optional `ImagePrefetcher` parameter to `PrefetchJob`**

  Constructor gains `{ImagePrefetcher? imagePrefetcher}` (default `null`
  — existing call sites/tests from Task 3 keep compiling unchanged). `run()`
  gains the foreground-liveness-gated call from the spec's Phase 2 section,
  after the existing per-target text-fetch loop, only when
  `imagePrefetcher != null`.

- [x] **Step 5: Extend `prefetch_job_test.dart` for image prefetch**

  Add cases: with a fake `ImagePrefetcher` injected and no recent
  foreground heartbeat (`storage.foregroundHeartbeatAt` unset or old) →
  `warm(...)` is called with the expected URL set (respecting the N=20
  cap and read-link exclusion from Step 1's policy); with a **recent**
  heartbeat (< 60s old) → `warm(...)` is **not** called at all (text
  prefetch still runs and still records a last-run) — this is the
  regression test for the "don't fight the foreground for the image
  cache" guard. `imagePrefetcher == null` (Task 3's original tests) →
  unaffected, still pass unchanged.

- [x] **Step 6: Run and confirm pass**

  `flutter test test/prefetch_job_test.dart test/prefetch_policy_test.dart` — PASS.

- [x] **Step 7: Wire the real `CacheManagerImagePrefetcher` into `background_main.dart`**

  `_registry`'s `PrefetchJob()` becomes
  `PrefetchJob(imagePrefetcher: CacheManagerImagePrefetcher())`.

- [x] **Step 8: Run the full suite**

  `flutter test` — PASS.

- [x] **Step 9: Run static analysis**

  `flutter analyze` — no new issues.

- [x] **Step 10: Commit**

  ```bash
  git add lib/features/offline_prefetch/domain/image_prefetcher.dart lib/features/offline_prefetch/domain/prefetch_policy.dart lib/features/offline_prefetch/data/cache_manager_image_prefetcher.dart lib/features/offline_prefetch/data/prefetch_job.dart lib/background_main.dart test/prefetch_policy_test.dart test/prefetch_job_test.dart
  git commit -m "Add image prefetch behind a foreground-liveness guard (Phase 2)"
  ```

---

### Task 7: `FeedStatusBanner` offline-with-content variant + detail-screen dead-end fix

**Files:**
- Modify: `lib/features/feed/presentation/widgets/feed_status_banner.dart`
- Modify: `lib/features/feed/presentation/screens/for_you_screen.dart`
- Modify: `lib/features/feed/presentation/screens/category_feed_screen.dart`
- Modify: `lib/features/feed/presentation/screens/article_detail_screen.dart`
- Modify (or create): `test/feed_status_banner_test.dart`
- Modify: `test/screens_smoke_test.dart` (or wherever
  `ArticleDetailScreen`'s existing smoke tests live)

**Interfaces:**
- Consumes: `Article.hasFullContent` (existing).
- Produces: `FeedStatusBanner`'s new `savedCount` parameter (backward
  compatible — default `0`, existing call sites without it keep today's
  behavior unless updated in this task).

**Surface:** ui

- [x] **Step 1: Write failing tests for `FeedStatusBanner`'s new variant**

  Extend/create `test/feed_status_banner_test.dart`: `isOffline: true,
  savedCount: 0` → today's "Showing saved articles — pull to retry" text
  + `cloud_off_rounded`; `isOffline: true, savedCount: 12` → "Offline —
  12 articles saved in full" + `offline_pin_rounded` (filled); `isOffline:
  false` (refreshing) → unchanged spinner + "Refreshing…" regardless of
  `savedCount`.

- [x] **Step 2: Run and confirm failure**

  Expected: FAIL — `savedCount` parameter doesn't exist yet.

- [x] **Step 3: Implement the new variant**

  Add `this.savedCount = 0` to `FeedStatusBanner`'s constructor; branch
  the offline case on `savedCount > 0` per the spec's UI section.

- [x] **Step 4: Run and confirm pass**

  `flutter test test/feed_status_banner_test.dart` — PASS.

- [x] **Step 5: Wire `savedCount` through both call sites**

  In `for_you_screen.dart:198` and `category_feed_screen.dart:141`,
  compute `articles.where((a) => a.hasFullContent).length` from the
  already-in-hand `articles` list passed to `_ArticlesSliverList` (or
  wherever the count is most naturally computed alongside `isOffline` —
  match each file's existing structure) and pass it as `savedCount`.

- [x] **Step 6: Run and confirm pass**

  `flutter test` — PASS (full suite; confirm neither screen's existing
  offline-banner tests, if any, broke).

- [x] **Step 7: Write failing test for the detail-screen fallback**

  Extend the detail-screen's existing smoke test file: an article with
  empty `content` (and empty `summary`, so the fallback branch is
  actually hit — check `Article.fromJson`'s summary-fallback behavior
  doesn't mask this in the fixture) renders the new fallback treatment
  ("Not saved for offline" title text, or whatever exact copy is chosen —
  match it to the spec's UI section) instead of the old bare string.

- [x] **Step 8: Run and confirm failure, implement, confirm pass**

  Replace `article_detail_screen.dart`'s
  `article.content.isNotEmpty ? article.content : 'No preview available
  for this article.'` fallback branch with the labeled empty-state
  treatment from the spec's UI section (reusing
  `lib/core/widgets/empty_state.dart` if it has a compact/inline form
  suitable for inline use within the body `SelectableText`'s position —
  otherwise a small `Icon` + two `Text`s matching this screen's own type
  scale, since `SelectableText` itself can't render an icon). Do **not**
  change the "Read full article"/"View original" button below it — that
  connectivity-aware relabeling is explicitly out of scope (see spec
  Non-goals).

  `flutter test` — PASS.

- [x] **Step 9: Run static analysis**

  `flutter analyze` — no new issues.

- [x] **Step 10: Run the full suite one final time for this plan**

  `flutter test` — PASS, every test from every task.

- [x] **Step 11: Commit**

  ```bash
  git add lib/features/feed/presentation/widgets/feed_status_banner.dart lib/features/feed/presentation/screens/for_you_screen.dart lib/features/feed/presentation/screens/category_feed_screen.dart lib/features/feed/presentation/screens/article_detail_screen.dart test/feed_status_banner_test.dart test/screens_smoke_test.dart
  git commit -m "Show saved-offline count in FeedStatusBanner; label the detail-screen not-saved state"
  ```

---

## Self-Review Notes

- **Spec coverage:** every spec section maps to a task — prerequisites
  (Task 1), background plumbing (Task 2), `PrefetchJob` + entry point +
  initial scheduling (Task 3), toggle/scheduling providers (Task 4),
  Settings UI (Task 5), image prefetch (Task 6), banner variant +
  detail-screen fix (Task 7). Non-goals (iOS, #7, storage cap, connectivity-
  aware button relabeling, `clearArticleCache()` image-cache scope) are
  intentionally not tasked.
- **Ordering rationale:** Task 1 must land first — Tasks 3/4 both depend
  on its storage keys and `feedTargetsFor`/`CatalogCache`. Task 2
  (scheduling plumbing) has no dependency on Task 1 and could theoretically
  run in parallel, but is sequenced second for a simpler linear commit
  history; an implementer following `subagent-driven-development` may
  reorder Tasks 1 and 2 relative to each other if that agent finds it
  cleaner, but Task 3 must not start before both are committed, and Tasks
  4–7 must not start before Task 3.
- **Known open implementer decisions, flagged rather than silently
  resolved** (both explicitly permitted by the spec): (a) whether manual
  "Update now"/"Save now" shows a Wi-Fi confirmation dialog on metered data
  (Task 5, Step 3); (b) how `CacheManagerImagePrefetcher` gets unit-tested
  given `flutter_cache_manager`'s API constraints (Task 6, Step 3). Record
  whichever choice is made for each in this section once decided, so a
  reviewer isn't left guessing why the code doesn't match a literal reading
  of the spec.
- **Decision (a), resolved in Task 5 — no metered-data confirmation
  dialog.** "Save now"/"Update now" runs immediately, with no "Use mobile
  data?" `AlertDialog`. Reasons: (1) knowing whether the current connection
  is metered requires `connectivity_plus`, a dependency this PR explicitly
  defers in Non-goals — without it the dialog would have to be shown
  unconditionally, which turns a one-tap action into a two-tap one for the
  common Wi-Fi case; (2) an explicit tap on "Save now" *is* the user asking
  for this transfer right now, so a confirm step is friction on a funnel
  step the user just deliberately chose; (3) the transfer is small (RSS
  text for the followed targets, no images until Task 6). Follow-on
  consequence, deliberately taken: the Wi-Fi-only subtitle copy is scoped
  to automatic runs — "Automatic saves never use mobile data" / "Automatic
  saves can use mobile data when Wi-Fi isn't available" instead of the
  spec's literal "Never uses mobile data" / "Will use mobile data when
  Wi-Fi isn't available" — because the unmetered constraint genuinely only
  applies to the WorkManager-scheduled runs, and the original wording would
  be a false promise the moment a user taps "Save now" on cellular. If
  `connectivity_plus` ever lands for another reason, revisiting this with a
  real metered check is cheap.
- **Decision (b), resolved in Task 6 — `CacheManagerImagePrefetcher` is
  unit-tested directly, not only transitively.** `flutter_cache_manager`'s
  concrete `CacheManager`/`DefaultCacheManager` needs a real cache
  directory (`path_provider`, a platform channel) to construct, but the
  `BaseCacheManager` abstract interface it implements has no such
  constructor requirement and is a clean, fully-abstract surface —
  straightforward to implement as a hand-written fake. So
  `CacheManagerImagePrefetcher` took an injectable `BaseCacheManager?`
  constructor parameter (defaulting to `DefaultCacheManager()` for the real
  registration in `background_main.dart`), and
  `test/cache_manager_image_prefetcher_test.dart` exercises it directly
  against a `_FakeCacheManager implements BaseCacheManager`, including the
  "one bad URL doesn't abort the batch" case the plan specifically calls
  out. This is *in addition to* the transitive coverage
  `test/prefetch_job_test.dart` already gets via a fake `ImagePrefetcher` —
  belt and suspenders, not a substitute for one or the other. Consequence:
  `flutter_cache_manager` (previously only a transitive dependency of
  `cached_network_image`) and `file` (the package `BaseCacheManager`'s
  `downloadFile` return type depends on) were promoted to direct
  dependencies in `pubspec.yaml` (`flutter_cache_manager` under
  `dependencies`, `file` under `dev_dependencies`, both pinned to the
  versions already resolved transitively — no version bump to anything
  else), so `pubspec.yaml`/`pubspec.lock` are part of Task 6's commit even
  though the plan's literal Step 10 file list didn't enumerate them (the
  code doesn't compile without that pin once `flutter_cache_manager`/`file`
  symbols are imported directly rather than only through
  `cached_network_image`).
- **The one-PR scope boundary:** this plan is Tasks 1–7, Android only, no
  notification code. Phase 7 of the SDLC pipeline running this plan
  updates `FEATURE_ROADMAP.md`/`README.md`/`CLAUDE.md` directly — no task
  in this plan touches those docs itself.
- **Manual verification gate, not automatable by any task above:** before
  merge, build a real obfuscated release AAB, install on a physical
  Android device, and confirm the scheduled job actually fires with
  `@pragma('vm:entry-point')` surviving R8 + Dart obfuscation (see spec's
  Testing section). If this session has no way to perform that check (no
  physical device / no real release-signing credentials available in this
  environment), the plan's Step 5/6 checkbox stays unflipped, one sentence
  explaining why is added directly under this bullet at execution time,
  and it's carried into the PR body as an explicit outstanding item — not
  silently skipped or claimed as passing.
