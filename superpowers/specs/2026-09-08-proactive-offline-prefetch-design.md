# Proactive offline pre-fetch (roadmap #5)

## Purpose

FEATURE_ROADMAP.md item #5, built with the generic periodic background-task
plumbing that item #7 (on-device breaking-news alerts) will register a
second job into later. Today the app is offline-first only in the sense
that *whatever the user already opened* is cached — a commute or flight
that starts cold gets nothing more than headlines with truncated summaries
for anything not already read. This adds a Wi-Fi-by-default, opt-out
periodic background job (Android only in this PR — iOS is a deliberate
follow-up, see Non-goals) that proactively fetches full article bodies
(already a near-free win — see below) and warms lead images for the next
~20 unread articles across the user's actual followed categories, local
region, and custom feeds, so opening the app offline shows full content
instead of a dead end.

## What "full body" prefetch actually requires (read this first)

`Article.content` is already populated from `content:encoded`/Atom
`content` at RSS/Atom mapping time, already round-trips through disk-cache
JSON, and `ArticleDetailScreen` already renders it with no supplementary
fetch (`article.dart:13`, `article_detail_screen.dart:249-254`).
`ArticleRepositoryImpl.fetchArticlesForSources` already writes its result
through to disk on every call (`article_repository_impl.dart:58-68`). **So
"prefetch full article bodies" reduces entirely to: call
`fetchArticlesForSources` for every followed cache key on a timer, while
the app isn't foregrounded.** This spec does not add an article-body
fetcher — none is needed. The actual new work is: (1) a background-task
scheduling layer that can wake and call that existing method outside the
app's UI lifecycle, and (2) budgeted lead-image warming, which *is* new
work (see Phase 2 below).

## Requirements (from product-manager + architect + designer passes)

- A periodic background job, registered via `workmanager` (Android;
  ~15-minute OS-enforced minimum interval), runs independent of the app
  being foregrounded.
- Wi-Fi-only by default (`NetworkType.unmetered` at the OS scheduling
  layer, not a Dart-side check after wake) — this is what "Wi-Fi-by-default"
  in the roadmap item means; see Settings for the user-facing override.
- The job walks the user's **actual** followed cache keys — interest
  categories, resolved local-news region, custom feeds — via a single
  shared pure function, not a fourth reimplementation of the three-way
  composition that already exists four times in `feed_providers.dart`
  (see Data flow).
- Pre-fetched content write-throughs to the same disk-cache path the app
  already reads on cold start (`LocalStorageService`/
  `*DiskCacheProvider`) — verifiable fully offline (airplane mode): open
  an affected article and see full body + image with zero network call.
- Feature is opt-out (default ON), with a Settings toggle that **cancels
  the scheduled job**, not just skips work at runtime when checked.
- The background-task registration is a reusable "run this job on this
  cadence" entry point (a small job registry) so a future PR can register
  #7's alert-detection job into the same scheduled wake without a second
  platform-channel integration. This PR adds **no** notification
  permission, **no** alert-detection heuristic, and **no** notification UI
  — that is #7, entirely out of scope here.
- Settings-level UX: a toggle + nested "Only download on Wi-Fi" switch +
  a status row (last-run time/count, manual "Update now") in the existing
  Data section, plus a small UX repair to the article-detail dead end that
  makes the feature's absence (an article that wasn't prefetched) look
  like a labeled state instead of a bug.

## Corrections from the design passes, recorded so they aren't re-litigated

- **No Riverpod in the background isolate.** A `ProviderContainer` there
  would silently fire four unrequested network calls on every wake
  (`InterestCategoriesNotifier`/`LocalNewsRegionsNotifier`/
  `MinSupportedVersionNotifier`/`GeoCountryNotifier` all call their
  `_refreshOnce()`/`_resolveOnce()` from `build()`), plus
  `forYouFeedProvider`'s home-widget side effect
  (`feed_providers.dart:165`). Instead, both the foreground providers and
  the background job call the **same plain functions**
  (`feedTargetsFor`, `CatalogCache`) — stronger sharing than a provider
  graph, and it's what makes the "actual followed cache keys, not
  hardcoded" requirement true without dragging Riverpod into an isolate
  that can't construct one anyway.
- **The resolved country is never persisted today** (`GeoCountryNotifier`
  keeps it in memory only) — the background job has no way to know the
  user's region without repeating the IP lookup on every 15-minute wake,
  which defeats the point of a lightweight background job. Fixed as a
  small, independently-valuable prerequisite (Task 1): persist it, and
  seed `GeoCountryNotifier`'s locale fallback from it on next cold start
  too.
- **The `_kMaxDiskCachedArticles = 60` cap is not renegotiable in this
  PR.** It's a deliberate, commented constraint on `SharedPreferences`
  parsed synchronously at launch (`article_repository_impl.dart:23-30`).
  "Next N unread" for image-warming purposes (Phase 2) is defined
  *within* that cap (N ≈ 20), not by raising it.
- **Image prefetch is its own phase, behind its own abstraction, gated on
  a foreground-liveness heartbeat.** `flutter_cache_manager`'s own docs
  say not to run two `CacheManager` instances against the same key
  concurrently; a background isolate warming images while the foreground
  is actively scrolling risks exactly that. The guard: skip image warming
  (text prefetch still runs) if a foreground heartbeat was written in the
  last ~60 seconds.
- **No `ArticleCard`/list-item "offline ready" badge.** The designer pass
  argued this against the "mostly invisible" framing this feature should
  have (per-card inventory the user has to read, and the overlay budget
  on `ArticleCard` is already spent — category pill, bookmark, source-type
  badge). Counter-proposal adopted instead: a third `FeedStatusBanner`
  variant, list-level not card-level (see UI).
- **This PR is Android-only.** iOS needs `BGAppRefreshTask` registration
  in `AppDelegate.swift` plus `Info.plist` entries, which — like the iOS
  widget follow-up already tracked in FEATURE_ROADMAP.md #4 — wants to be
  done directly in Xcode rather than scripted blind, and iOS's lack of a
  delivery-time guarantee is a distinct enough risk to review on its own.
  `core/background`'s interfaces are platform-agnostic so that PR only
  adds a second `BackgroundScheduler` implementation, not new job logic.
  Matches the precedent already set by #4 (Android shipped, iOS a
  tracked follow-up).

## Data flow

### Task 1 — prerequisites (no behavior change)

**`LocalStorageService`** (`lib/core/storage/local_storage_service.dart`)
gains, alongside its existing key groups:

```dart
static const _kLastResolvedCountryKey = 'last_resolved_country_code';
static const _kOfflinePrefetchEnabledKey = 'offline_prefetch_enabled';
static const _kOfflinePrefetchWifiOnlyKey = 'offline_prefetch_wifi_only';
static const _kPrefetchLastRunAtKey = 'offline_prefetch_last_run_at';
static const _kPrefetchLastRunCountKey = 'offline_prefetch_last_run_count';
static const _kForegroundHeartbeatAtKey = 'foreground_heartbeat_at';

String? get lastResolvedCountryCode => _prefs.getString(_kLastResolvedCountryKey);
Future<void> setLastResolvedCountryCode(String code) =>
    _prefs.setString(_kLastResolvedCountryKey, code);

// Default true — see spec's "opt-out" requirement.
bool get offlinePrefetchEnabled =>
    _prefs.getBool(_kOfflinePrefetchEnabledKey) ?? true;
Future<void> setOfflinePrefetchEnabled(bool value) =>
    _prefs.setBool(_kOfflinePrefetchEnabledKey, value);

bool get offlinePrefetchWifiOnly =>
    _prefs.getBool(_kOfflinePrefetchWifiOnlyKey) ?? true;
Future<void> setOfflinePrefetchWifiOnly(bool value) =>
    _prefs.setBool(_kOfflinePrefetchWifiOnlyKey, value);

DateTime? get prefetchLastRunAt {
  final raw = _prefs.getString(_kPrefetchLastRunAtKey);
  return raw == null ? null : DateTime.tryParse(raw);
}
Future<void> setPrefetchLastRun(DateTime at, int itemCount) async {
  await _prefs.setString(_kPrefetchLastRunAtKey, at.toUtc().toIso8601String());
  await _prefs.setInt(_kPrefetchLastRunCountKey, itemCount);
}
int? get prefetchLastRunCount => _prefs.getInt(_kPrefetchLastRunCountKey);

DateTime? get foregroundHeartbeatAt {
  final raw = _prefs.getString(_kForegroundHeartbeatAtKey);
  return raw == null ? null : DateTime.tryParse(raw);
}
Future<void> touchForegroundHeartbeat() =>
    _prefs.setString(_kForegroundHeartbeatAtKey, DateTime.now().toUtc().toIso8601String());
```

**Background-isolate write rule** (documented as a doc comment on
`PrefetchJob`, not enforced by a type — see architect's finding): the job
may only write article-cache keys (via `fetchArticlesForSources`) and its
own `offline_prefetch_last_run_*`/`foreground_heartbeat_at` keys. It must
never write `read_article_links`/`read_events`, `muted_source_urls`,
`custom_sources`, or `bookmarked_articles` — those are read-modify-write
keys the foreground isolate also owns, and two isolates read-modify-writing
the same key independently will silently clobber each other. Reads of
those keys are safe and expected.

**`GeoCountryNotifier`** (`lib/core/providers/location_provider.dart`)
persists on successful resolution and seeds its `build()` from the
persisted value before falling back to the locale guess:

```dart
@override
String? build() {
  final localeFallback = ref.watch(deviceCountryCodeProvider);
  final persisted = ref.watch(localStorageServiceProvider).lastResolvedCountryCode;
  _resolveOnce();
  return persisted ?? localeFallback;
}

Future<void> _resolveOnce() async {
  ...
  if (code == null || !ref.mounted) return;
  state = code;
  await ref.read(localStorageServiceProvider).setLastResolvedCountryCode(code);
}
```

This is also a small standalone UX improvement (a returning user's cold
start starts from yesterday's resolved country instead of the locale
guess) — independently valuable, not just plumbing for this feature.

**New `lib/features/feed/domain/feed_targets.dart`** — the single source
of truth for "what does this user actually follow", used by both the
foreground providers (refactored onto it) and the background job:

```dart
import '../../../core/constants/interest_categories.dart';
import '../../../core/constants/local_news_sources.dart';

/// One fetchable unit: a disk-cache key plus the sources that feed it.
/// [feedTargetsFor] always returns exactly one [FeedTarget] per followed
/// category, plus exactly one for the local region (even [kGlobalNewsRegion]
/// counts — local news is unconditional, never absent), plus one for custom
/// feeds if any exist.
class FeedTarget {
  const FeedTarget({required this.cacheKey, required this.sources});
  final String cacheKey;
  final List<NewsSource> sources;
}

/// Pure composition of "what should be fetched for this user" — followed
/// categories (intersected with what the catalog actually contains, same
/// [followedCategoryIdsProvider] rule), the local region for
/// [countryCode], and any custom feeds. No Riverpod, no I/O: every input is
/// a plain value already resolved by the caller, so this is callable from
/// both foreground providers and the background isolate (which cannot
/// build a ProviderContainer — see spec's Corrections section).
List<FeedTarget> feedTargetsFor({
  required List<InterestCategory> categories,
  required List<String> followedCategoryIds,
  required List<LocalNewsRegion> regions,
  required String? countryCode,
  required List<NewsSource> customSources,
}) {
  final validIds = categories.map((c) => c.id).toSet();
  final targets = <FeedTarget>[
    for (final id in followedCategoryIds.where(validIds.contains))
      FeedTarget(
        cacheKey: id,
        sources: categoryByIdOrNull(categories, id)!.sources,
      ),
  ];
  final region = regionForCountry(regions, countryCode);
  targets.add(FeedTarget(
    cacheKey: localFeedCacheKey(region.countryCode),
    sources: region.sources,
  ));
  if (customSources.isNotEmpty) {
    targets.add(FeedTarget(cacheKey: kCustomFeedCacheKey, sources: customSources));
  }
  return targets;
}
```

Refactor `feed_providers.dart`'s four duplicated compositions
(`forYouFeedProvider:140-155`, `forYouDiskCacheProvider:196-200`,
`allArticlesFetchProvider:227-241`, `allArticlesDiskCacheProvider:260-264`)
to build their fetch/read lists from `feedTargetsFor(...).map((t) =>
repository.fetchArticlesForSources(t.cacheKey, t.sources, ...))` (or
`readDiskCache(t.cacheKey)`) instead of hand-assembling the three-way list
each time. **Behavior must not change** — this is a pure refactor; the
existing tests for these four providers are the regression check (see
Testing).

**New `lib/core/catalog/catalog_cache.dart`** — last-known catalog,
readable with no Riverpod:

```dart
import 'dart:convert';

import '../constants/interest_categories.dart';
import '../constants/local_news_sources.dart';
import '../storage/local_storage_service.dart';

/// Last-known interest-category / local-news-region catalog, read straight
/// from [LocalStorageService]'s cached remote-config JSON — no network, no
/// Riverpod. The background job uses this instead of
/// [InterestCategoriesNotifier]/[LocalNewsRegionsNotifier] because those
/// notifiers fire a remote refresh from `build()`, which the background job
/// must not do (see spec's Corrections: no unrequested network calls, and
/// the job's own network budget is feeds only). Falls back to the bundled
/// defaults exactly like the notifiers do when nothing's cached yet or the
/// cached JSON fails to parse.
class CatalogCache {
  const CatalogCache(this._storage);
  final LocalStorageService _storage;

  List<InterestCategory> categories() {
    final raw = _storage.cachedInterestCategoriesJson;
    if (raw == null) return kDefaultInterestCategories;
    try {
      final decoded = jsonDecode(raw) as List<dynamic>;
      final parsed = decoded
          .map((e) => InterestCategory.fromJson(e as Map<String, dynamic>))
          .toList();
      return parsed.isEmpty ? kDefaultInterestCategories : parsed;
    } catch (_) {
      return kDefaultInterestCategories;
    }
  }

  List<LocalNewsRegion> regions() {
    final raw = _storage.cachedLocalNewsRegionsJson;
    if (raw == null) return kDefaultLocalNewsRegions;
    try {
      final decoded = jsonDecode(raw) as List<dynamic>;
      final parsed = decoded
          .map((e) => LocalNewsRegion.fromJson(e as Map<String, dynamic>))
          .toList();
      return parsed.isEmpty ? kDefaultLocalNewsRegions : parsed;
    } catch (_) {
      return kDefaultLocalNewsRegions;
    }
  }
}
```

`InterestCategoriesNotifier._readCache`/`LocalNewsRegionsNotifier._readCache`
may optionally delegate to this (not required — they're `Notifier`s reading
via `ref.read`, this is a plain class reading via a constructor-injected
`LocalStorageService`; forcing them onto the same type isn't worth
contorting either shape). Not refactoring them is an acceptable, explicitly
recorded choice — don't let a reviewer treat the untouched duplication as
a miss.

### Task 2/3 — background-task plumbing

**New `lib/core/background/` — feature-agnostic, imports nothing from
`features/`:**

`background_job.dart`:
```dart
import 'background_job_context.dart';

/// One unit of background work, identified by [id] (the `workmanager` task
/// name it's registered under). [run] returns `true` on success (WorkManager
/// marks the run complete) or `false` to ask for a retry on the platform's
/// own backoff policy. Implementations must not throw for an expected
/// failure (no network, etc) — catch internally and return `false`; an
/// uncaught throw is still treated as failure by the dispatcher, but every
/// job should decide its own retry-worthiness explicitly rather than rely
/// on that fallback.
abstract interface class BackgroundJob {
  String get id;
  Future<bool> run(BackgroundJobContext ctx);
}
```

`background_job_context.dart`:
```dart
import 'package:dio/dio.dart';

import '../network/http_client.dart';
import '../storage/local_storage_service.dart';

/// What a [BackgroundJob] gets to work with — deliberately not a
/// ProviderContainer (see spec's Corrections: no Riverpod in the
/// background isolate). Built fresh on every wake in [background_main.dart].
class BackgroundJobContext {
  BackgroundJobContext({required this.storage, Dio? dio})
      : dio = dio ?? buildHttpClient();
  final LocalStorageService storage;
  final Dio dio;
}
```

`background_job_registry.dart`:
```dart
import 'background_job.dart';

/// Maps a `workmanager` task name to the [BackgroundJob] that handles it.
/// This is the inversion point that lets a later PR (#7) register a second
/// job without touching [BackgroundScheduler] or the callback dispatcher —
/// see `lib/background_main.dart`.
class BackgroundJobRegistry {
  BackgroundJobRegistry(Iterable<BackgroundJob> jobs)
      : _byId = {for (final j in jobs) j.id: j};
  final Map<String, BackgroundJob> _byId;
  BackgroundJob? operator [](String id) => _byId[id];
}
```

`background_scheduler.dart`:
```dart
/// Registers/cancels periodic background work. Abstracted so
/// `workmanager` (a plugin with a platform channel) never appears in a
/// widget/provider test — tests fake this interface instead.
abstract interface class BackgroundScheduler {
  Future<void> schedulePeriodic({
    required String jobId,
    required Duration interval,
    required bool unmeteredOnly,
  });
  Future<void> cancel(String jobId);
}
```

`workmanager_background_scheduler.dart` (the only file that imports
`package:workmanager`):
```dart
import 'package:workmanager/workmanager.dart';

import 'background_scheduler.dart';

class WorkmanagerBackgroundScheduler implements BackgroundScheduler {
  WorkmanagerBackgroundScheduler(this._callbackDispatcher);
  final Function _callbackDispatcher; // top-level @pragma('vm:entry-point') fn
  bool _initialized = false;

  Future<void> _ensureInitialized() async {
    if (_initialized) return;
    await Workmanager().initialize(_callbackDispatcher);
    _initialized = true;
  }

  @override
  Future<void> schedulePeriodic({
    required String jobId,
    required Duration interval,
    required bool unmeteredOnly,
  }) async {
    await _ensureInitialized();
    await Workmanager().registerPeriodicTask(
      jobId,
      jobId,
      frequency: interval,
      existingWorkPolicy: ExistingPeriodicWorkPolicy.update,
      constraints: Constraints(
        networkType: unmeteredOnly ? NetworkType.unmetered : NetworkType.connected,
      ),
    );
  }

  @override
  Future<void> cancel(String jobId) async {
    await _ensureInitialized();
    await Workmanager().cancelByUniqueName(jobId);
  }
}
```

*(`ExistingPeriodicWorkPolicy`/`Constraints`/`NetworkType` are the
`workmanager` package's own exported types — verify the exact enum/class
names and `registerPeriodicTask` signature against whatever version lands
in `pubspec.yaml` when this task starts; architect's design pass flagged
these as confirmed from published dartdoc, not a local compile, and noted
`ExistingPeriodicWorkPolicy` is a distinct type from one-off
`ExistingWorkPolicy` — easy to mis-import.)*

**New `lib/features/offline_prefetch/data/prefetch_job.dart`:**

```dart
import '../../../core/background/background_job.dart';
import '../../../core/background/background_job_context.dart';
import '../../../core/catalog/catalog_cache.dart';
import '../../../core/constants/interest_categories.dart';
import '../../feed/data/datasources/rss_remote_data_source.dart';
import '../../feed/data/repositories/article_repository_impl.dart';
import '../../feed/domain/feed_targets.dart';

/// Walks the user's followed categories/region/custom feeds and re-runs the
/// existing fetch-merge-cache pipeline for each — which already writes full
/// article bodies through to disk (see spec: no separate body-fetcher is
/// needed). Image warming is added in Phase 2 (Task 6); this class's [run]
/// only does text on introduction and gains an [ImagePrefetcher] call once
/// Task 6 lands.
class PrefetchJob implements BackgroundJob {
  const PrefetchJob();

  static const jobId = 'offline_prefetch';

  @override
  String get id => jobId;

  @override
  Future<bool> run(BackgroundJobContext ctx) async {
    if (!ctx.storage.offlinePrefetchEnabled) return true;

    final catalog = CatalogCache(ctx.storage);
    final targets = feedTargetsFor(
      categories: catalog.categories(),
      followedCategoryIds: ctx.storage.selectedInterests,
      regions: catalog.regions(),
      countryCode: ctx.storage.lastResolvedCountryCode,
      customSources: ctx.storage.customSources
          .map((m) => NewsSource(name: m['name'] as String, rssUrl: m['rssUrl'] as String))
          .toList(),
    );
    if (targets.isEmpty) return true; // nothing followed yet — nothing to do

    final repository = ArticleRepositoryImpl(RssRemoteDataSource(ctx.dio), ctx.storage);
    final muted = ctx.storage.mutedSourceUrls;
    var total = 0;
    for (final target in targets) {
      try {
        final articles = await repository.fetchArticlesForSources(
          target.cacheKey,
          target.sources,
          forceRefresh: true,
          excludedSourceUrls: muted,
        );
        total += articles.length;
      } catch (_) {
        // One source's failure (network blip, one feed down) shouldn't
        // fail the whole run — same isolation ArticleRepositoryImpl
        // already applies per-source inside fetchArticlesForSources.
      }
    }

    await ctx.storage.setPrefetchLastRun(DateTime.now(), total);
    return true;
  }
}
```

**New `lib/background_main.dart`** — the background-isolate composition
root, mirroring `main.dart`'s role for the foreground isolate:

```dart
import 'package:flutter/widgets.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:workmanager/workmanager.dart';

import 'core/background/background_job_context.dart';
import 'core/background/background_job_registry.dart';
import 'core/storage/local_storage_service.dart';
import 'features/offline_prefetch/data/prefetch_job.dart';

/// Every job a background wake can run. #7 adds its job here, alongside
/// [PrefetchJob] — this file is the one place that has to change for a
/// second job to exist; core/background and PrefetchJob itself do not.
final _registry = BackgroundJobRegistry(const [
  PrefetchJob(),
]);

/// Registered with `Workmanager().initialize` in
/// [WorkmanagerBackgroundScheduler]. Must be a top-level or static function
/// — an instance method is rejected by the plugin — and must keep this
/// exact `@pragma` so R8/obfuscation (this app builds obfuscated release
/// AABs — see CLAUDE.md) doesn't strip or rename it out from under
/// WorkManager's reflective lookup by task name.
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((taskName, _) async {
    WidgetsFlutterBinding.ensureInitialized();
    final job = _registry[taskName];
    if (job == null) return true; // unregistered/stale task name — ack, don't retry forever
    final prefs = await SharedPreferences.getInstance();
    final ctx = BackgroundJobContext(storage: LocalStorageService(prefs));
    try {
      return await job.run(ctx);
    } catch (_) {
      return false; // ask WorkManager to retry per its own backoff
    }
  });
}
```

**`WidgetsFlutterBinding.ensureInitialized()` inside the callback is
required** for `shared_preferences`'/`flutter_cache_manager`'s platform
channels to work in the background isolate — this is the app's first
background Dart entry point (see Non-goals/manual verification below for
why this needs a real-device check before merge, not just a debug run).

### Task 4 — toggle + scheduling

**New `lib/features/offline_prefetch/presentation/providers/offline_prefetch_providers.dart`:**

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../../../core/background/background_scheduler.dart';
import '../../../../core/background/workmanager_background_scheduler.dart';
import '../../../../core/providers/core_providers.dart';
import '../../data/prefetch_job.dart';
import '../../../../background_main.dart' show callbackDispatcher;

const _kPrefetchInterval = Duration(minutes: 15); // workmanager's Android floor

final backgroundSchedulerProvider = Provider<BackgroundScheduler>((ref) {
  return WorkmanagerBackgroundScheduler(callbackDispatcher);
});

/// Owns the "Save today's feed for offline" toggle. Setting it schedules or
/// cancels [PrefetchJob] via [BackgroundScheduler] as a side effect of the
/// state change itself — never deferred to a later `build()`, since a
/// cancel that only happens on next app start would leave a stale job
/// running on Android's own schedule regardless of what the user chose.
class OfflinePrefetchEnabledNotifier extends Notifier<bool> {
  @override
  bool build() => ref.watch(localStorageServiceProvider).offlinePrefetchEnabled;

  Future<void> setEnabled(bool value) async {
    state = value;
    await ref.read(localStorageServiceProvider).setOfflinePrefetchEnabled(value);
    final scheduler = ref.read(backgroundSchedulerProvider);
    if (value) {
      await scheduler.schedulePeriodic(
        jobId: PrefetchJob.jobId,
        interval: _kPrefetchInterval,
        unmeteredOnly: ref.read(offlinePrefetchWifiOnlyProvider),
      );
    } else {
      await scheduler.cancel(PrefetchJob.jobId);
    }
  }
}

final offlinePrefetchEnabledProvider =
    NotifierProvider<OfflinePrefetchEnabledNotifier, bool>(
  OfflinePrefetchEnabledNotifier.new,
);

class OfflinePrefetchWifiOnlyNotifier extends Notifier<bool> {
  @override
  bool build() => ref.watch(localStorageServiceProvider).offlinePrefetchWifiOnly;

  Future<void> setWifiOnly(bool value) async {
    state = value;
    await ref.read(localStorageServiceProvider).setOfflinePrefetchWifiOnly(value);
    // Re-register to apply the new constraint — registerPeriodicTask bakes
    // constraints in at registration time; ExistingPeriodicWorkPolicy.update
    // (see WorkmanagerBackgroundScheduler) makes this a safe re-schedule
    // rather than a duplicate.
    if (ref.read(offlinePrefetchEnabledProvider)) {
      await ref.read(backgroundSchedulerProvider).schedulePeriodic(
            jobId: PrefetchJob.jobId,
            interval: _kPrefetchInterval,
            unmeteredOnly: value,
          );
    }
  }
}

final offlinePrefetchWifiOnlyProvider =
    NotifierProvider<OfflinePrefetchWifiOnlyNotifier, bool>(
  OfflinePrefetchWifiOnlyNotifier.new,
);
```

**Initial scheduling, gated on onboarding.** In `main.dart`, after
`runApp` (or in a `NewsfeedApp.initState` via a post-frame callback,
matching the existing home-widget-tap pattern in `app.dart:37-39`): if
`storage.hasOnboarded && storage.offlinePrefetchEnabled`, call
`container.read(backgroundSchedulerProvider).schedulePeriodic(...)` once.
Zero followed categories (pre-onboarding) means nothing to prefetch, so
gating on `hasOnboarded` avoids scheduling a job that would just no-op on
every wake. Also call this same schedule step from
`SelectedInterestsNotifier.completeOnboarding()`
(`interests_provider.dart:20-23`) so a user who *just* onboarded gets the
job scheduled immediately rather than waiting for their next cold start.

**App lifecycle observer**, added to `_NewsfeedAppState`
(`lib/app.dart`) alongside its existing `initState`/`dispose`:

```dart
class _NewsfeedAppState extends ConsumerState<NewsfeedApp> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    ...
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    ...
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) {
      // The background isolate's SharedPreferences writes are invisible to
      // this isolate's own in-memory prefs cache until reload() — without
      // this, a prefetch that ran while backgrounded wouldn't show up until
      // the next full cold start.
      ref.read(localStorageServiceProvider).reload().then((_) {
        ref.read(cacheBusterProvider.notifier).bump();
      });
    }
    if (state == AppLifecycleState.resumed || state == AppLifecycleState.paused) {
      ref.read(localStorageServiceProvider).touchForegroundHeartbeat();
    }
  }
}
```

`LocalStorageService.reload()` is a thin wrapper the task below adds
(`_prefs.reload()` — a real `SharedPreferences` method; see package docs).

## UI

### Settings — Data section (per designer proposal)

New `lib/features/offline_prefetch/presentation/widgets/offline_prefetch_tile.dart`
encapsulating the whole block below, so `settings_screen.dart` gains one
widget rather than ~80 inline lines:

- **Row 1** — `SwitchListTile`: "Save today's feed for offline" /
  "Downloads about 20 unread articles so they're readable without a
  connection", bound to `offlinePrefetchEnabledProvider`.
- **Row 2** — nested, `contentPadding` left-inset to 56dp, no leading icon:
  "Only download on Wi-Fi", subtitle dynamic — "Never uses mobile data"
  (on) / "Will use mobile data when Wi-Fi isn't available" (off), bound to
  `offlinePrefetchWifiOnlyProvider`. `onChanged: null` (disabled, not
  hidden) when the parent toggle is off, wrapped in
  `Semantics(hint: 'Turn on Save today's feed for offline to change this')`.
- **Row 3** — status row (not a `ListTile` — a `Padding`+`Row` at the same
  visual weight as `FeedStatusBanner`), hidden entirely when the parent
  toggle is off:
  - Never run: `schedule_rounded` / "Not saved yet — runs automatically on
    Wi-Fi" / trailing "Save now" `TextButton`.
  - Has run: `offline_pin_rounded` (filled) / "Saved {timeAgo} · {count}
    articles" (reuse `lib/core/utils/date_format_utils.dart`'s `timeAgo`
    verbatim — do not invent a second relative-time format) / trailing
    "Update now".
  - Manual run in progress: 14px `CircularProgressIndicator` / "Saving
    articles…" / trailing button disabled.
  - Manual run failed: `cloud_off_rounded` / "Couldn't save — check your
    connection" / trailing "Try again".
  - `Semantics(liveRegion: true)` on this row so a screen-reader user gets
    an announcement when "Update now" changes its text, not silence.
  - "Update now"/"Save now" runs `PrefetchJob().run(...)` directly
    in-process (building a `BackgroundJobContext` from the already-live
    `localStorageServiceProvider`/`dioProvider`) rather than going through
    `workmanager` — a manual run is a foreground action with its own
    loading state, not a background wake. If Wi-Fi-only is on and the
    device is currently on mobile data, confirm first via an `AlertDialog`
    ("Use mobile data? Wi-Fi isn't available. Saving now will use mobile
    data." [Cancel] / [Save anyway]) matching the existing Clear-cache
    dialog pattern (`settings_screen.dart:103-121`) — this PR does not
    need a new connectivity-checking dependency if `Dio`'s own request
    failure is treated as "assume it's fine, let the fetch fail like any
    other offline fetch" instead; **implementer's call, record whichever
    is chosen in the plan's Self-Review Notes** rather than silently
    picking one.
- Existing "Clear cache" row stays immediately below, unchanged.
- Section label may stay "Data" or become "Offline" — designer's call was
  a soft preference for "Offline", not a requirement; keep "Data" if
  renaming touches enough surrounding text to feel like scope creep.

Accessibility bar: icons next to their own label text are decorative —
wrap in `ExcludeSemantics` (matching the existing pattern at
`source_type_badge.dart`); "Update now"/"Save now"/"Try again" buttons
must hit the ≥48dp tap-target minimum; status-row text must use
`Wrap`/`Flexible` so 200% dynamic type reflows instead of overflowing
(matching the existing fix already applied at
`article_detail_screen.dart:169-176`).

### `FeedStatusBanner` — new offline-with-content variant

`lib/features/feed/presentation/widgets/feed_status_banner.dart` gains a
third rendering: when offline and the articles being shown include
prefetched content, show `offline_pin_rounded` (filled) + "Offline — N
articles saved in full" instead of the current "Showing saved articles —
pull to retry". Computed by the caller from the already-in-hand articles
list (`articles.where((a) => a.hasFullContent).length`) — no new provider,
no new state; `Article.hasFullContent` already exists
(`article.dart:56`). Both call sites
(`for_you_screen.dart:198`, `category_feed_screen.dart:141`) pass this
count through. Suggested signature:

```dart
const FeedStatusBanner({super.key, this.isOffline = false, this.savedCount = 0});
```

`savedCount > 0 && isOffline` selects the new copy/icon; `isOffline` alone
(0 saved) keeps today's "pull to retry" copy; refreshing-online is
unchanged.

### `ArticleDetailScreen` — the dead-end fallback

`article_detail_screen.dart:249-254`'s bare
`'No preview available for this article.'` string is replaced with a
labeled state (reusing whatever the app's existing empty-state visual
idiom is, e.g. `lib/core/widgets/empty_state.dart`'s compact form if it
has one, otherwise a small inline `Icon` + two lines of text matching this
screen's own type scale): `cloud_off_rounded` / "Not saved for offline" /
"This one wasn't downloaded. It'll open in full once you're back online."
This is scope-limited to the fallback text/icon — **not** relabeling or
disabling the "Read full article" button based on live connectivity (that
needs a `connectivity_plus` dependency this PR doesn't add; explicitly
deferred, see Non-goals).

## Loading / error / empty states

- **Job run with nothing followed** (`targets.isEmpty`, e.g. a user who
  unfollowed every category): `PrefetchJob.run` returns `true` (success,
  nothing to do) without writing a last-run timestamp — the status row
  stays in "Not saved yet" rather than falsely claiming a 0-article save.
- **Every source in a run fails** (e.g. fully offline when the job
  happens to fire, which the Wi-Fi-only OS constraint should make rare
  but not impossible): `total` stays 0, the run still records
  `setPrefetchLastRun(now, 0)` — an honest "Saved just now · 0 articles"
  is more truthful than silently not updating the timestamp, and next
  wake will very likely succeed given the constraint that scheduled it.
- **Manual "Update now"/"Save now" failure** (e.g. Dio throws): status row
  shows the "failed" state from the UI section above; does not throw
  through to a crash or a generic error screen.
- **Toggle flipped off mid-run**: the in-flight run (either scheduled or
  manual) completes normally — `run()` doesn't watch the toggle mid-flight,
  it only checks it at the top (`if (!ctx.storage.offlinePrefetchEnabled)
  return true;`). The *next* wake won't happen because
  `setEnabled(false)` already cancelled the scheduled job.
- **Image prefetch skipped by the foreground-liveness guard** (Phase 2,
  Task 6): text prefetch and its status-row update still happen normally;
  this is silent by design (see Task 6) — no separate "images skipped"
  status state in this PR.

## Testing

- **`test/local_storage_service_test.dart`** (existing or new — check
  first): round-trip tests for every new key
  (`lastResolvedCountryCode`, `offlinePrefetchEnabled`/`WifiOnly` default
  values and persistence, `prefetchLastRunAt`/`Count`,
  `foregroundHeartbeatAt`), plus a `reload()` test if one doesn't already
  exist for the underlying `SharedPreferences` wrapper.
- **`test/feed_targets_test.dart`** (new): `feedTargetsFor` — followed
  categories intersected with the catalog (an id not in `categories` is
  dropped, matching `followedCategoryIdsProvider`'s existing rule); local
  region target is always present, including the `kGlobalNewsRegion`
  fallback for an unresolved/null country code; custom-feeds target only
  appears when `customSources` is non-empty; cache keys match
  `localFeedCacheKey`/`kCustomFeedCacheKey`'s existing conventions exactly
  (this is the regression check that the refactor in Task 1 didn't change
  any cache key any existing disk-cache read depends on).
- **`feed_providers_test.dart`** (existing, extended if needed): confirm
  `forYouFeedProvider`/`forYouDiskCacheProvider`/`allArticlesFetchProvider`/
  `allArticlesDiskCacheProvider` behave identically before/after the Task 1
  refactor onto `feedTargetsFor` — same inputs, same merged output. This
  is the primary safety net for "behavior must not change" above.
- **`test/catalog_cache_test.dart`** (new): returns bundled defaults when
  nothing cached; returns parsed cache when present; falls back to
  defaults on malformed JSON (mirrors the existing notifier tests'
  fallback coverage).
- **`test/prefetch_job_test.dart`** (new): construct a `PrefetchJob`
  against a `BackgroundJobContext` built from a fake/in-memory
  `LocalStorageService` and a `Dio` wired to a fake adapter (same pattern
  `rss_remote_data_source_test.dart` already uses) — assert: (a) the exact
  set of cache keys fetched for a given followed-categories + region +
  custom-feeds fixture, including a muted source being excluded and an
  unknown followed id being skipped; (b) `offlinePrefetchEnabled = false`
  short-circuits with no fetch calls at all; (c) `prefetchLastRunAt`/`Count`
  are written correctly, including the "0 targets → no write" and
  "targets present but every fetch fails → writes count 0" cases from
  Loading/error states above; (d) one source's fetch failure doesn't stop
  the others from running.
- **`test/offline_prefetch_providers_test.dart`** (new): a fake
  `BackgroundScheduler` (implementing the interface, recording calls) —
  assert `setEnabled(true)` calls `schedulePeriodic` with
  `unmeteredOnly` matching the current Wi-Fi-only state, `setEnabled(false)`
  calls `cancel` and *not* `schedulePeriodic`, and `setWifiOnly` re-calls
  `schedulePeriodic` only when already enabled (not when the toggle is
  off). This is the test that actually enforces the spec's "disabling
  cancels the job, not just skips work" requirement — `workmanager` itself
  is never touched by any test.
- **Widget tests** for `offline_prefetch_tile.dart`'s status-row states
  (never-run / has-run / in-progress / failed) and the nested switch's
  disabled+`Semantics(hint:)` behavior when the parent is off.
- **`feed_status_banner_test.dart`** (existing or new): the new
  `savedCount` parameter selects the right copy/icon at the right
  thresholds (0 vs >0, combined with `isOffline`).
- `flutter analyze && flutter test` both pass at the end of every task,
  and again at the end of the plan.
- **Manual, non-automatable gate** (per the plan's phasing — do not skip
  silently, carry into the PR body as an explicit outstanding item if it
  can't be done in this session): build a real obfuscated release AAB/APK,
  install on a physical Android device, confirm the scheduled job
  actually fires (WorkManager's own `adb shell dumpsys jobscheduler` or
  Android Studio's Background Task Inspector) and that
  `@pragma('vm:entry-point')` survived R8 + Dart obfuscation. This is the
  same class of "only surfaces on a real device" risk already recorded in
  FEATURE_ROADMAP.md #4's shipped-scope note about the `RemoteViews`
  `<View>` failure — a debug run or the automated test suite alone cannot
  confirm this.

## Non-goals (named so they're deferred, not forgotten)

- **iOS.** No `BGAppRefreshTask`/`Info.plist`/`AppDelegate.swift` changes
  in this PR — a tracked follow-up, same shape as #4's Android-first /
  iOS-follow-up split.
- **#7 (breaking-news alerts).** No notification permission, no
  alert-detection heuristic, no notification UI, no `flutter_local_notifications`
  or FCM/APNs dependency. `core/background`'s job registry exists
  specifically so #7 is "add one more `BackgroundJob` to the registry",
  not a second platform-channel integration — if a future PR finds that
  isn't true, this design's abstraction was wrong, which is a useful
  falsification signal for whoever picks up #7.
- **Raising `_kMaxDiskCachedArticles` past 60.** Explicitly rejected —
  see Corrections.
- **A storage-size cap/slider in Settings.** ~20 text bodies + images is
  single-digit MB; a Netflix-style GB stepper would misrepresent the
  scale of what this stores.
- **A first-run dialog, onboarding step, or notification for this
  feature.** Default-on, Wi-Fi-only-by-default, and a plainly-worded
  Settings subtitle is the proportionate disclosure for a client-only app
  that already doesn't do accounts/tracking; a modal would fight this
  app's fast onboarding for no commensurate benefit.
- **Connectivity-aware relabeling of the "Read full article"/"View
  original" button** on the detail screen (only the fallback *text* changes
  — see UI section). Would need `connectivity_plus`; not required to ship
  the core feature honestly.
- **`clearArticleCache()` also clearing the `flutter_cache_manager`
  image-disk cache**, or correcting its subtitle copy to mention images.
  Flagged as a real, small, pre-existing-adjacent gap by the designer pass
  but out of scope for this PR — noted here so it isn't silently
  rediscovered as a regression this PR caused.
- **Unifying `ManageSourcesScreen`'s badge rendering, `RelatedArticleTile`,
  or the Android home-screen widget** onto anything this PR touches — none
  of those are in scope.

## Phase 2 (Task 6) — image prefetch, added detail

`lib/features/offline_prefetch/domain/image_prefetcher.dart`:

```dart
abstract interface class ImagePrefetcher {
  Future<void> warm(Iterable<String> imageUrls);
}
```

`lib/features/offline_prefetch/data/cache_manager_image_prefetcher.dart`:
wraps `DefaultCacheManager().downloadFile(url)` per URL (same cache
manager + same implicit `cacheKey: url` the app's three existing
`CachedNetworkImage` call sites already rely on — no key mismatch), each
call wrapped in its own try/catch so one bad image URL doesn't abort the
batch.

`lib/features/offline_prefetch/domain/prefetch_policy.dart`: pure
selection — from the merged, deduped article list `PrefetchJob` already
built this run, drop already-read links (`ctx.storage.readLinks`), drop
any `imageUrl` that isn't a valid http(s) URL (reuse `safeArticleUri` from
`lib/core/utils/safe_link.dart` — it's generic despite its name saying
"article"), take the first N (N = 20) newest-first.

Wire into `PrefetchJob.run`, after the existing per-target fetch loop:

```dart
if (imagePrefetcher != null) {
  final heartbeat = ctx.storage.foregroundHeartbeatAt;
  final foregroundLikelyActive = heartbeat != null &&
      DateTime.now().toUtc().difference(heartbeat) < const Duration(seconds: 60);
  if (!foregroundLikelyActive) {
    final urls = selectImageUrlsToWarm(allFetchedArticles, ctx.storage.readLinks, limit: 20);
    await imagePrefetcher!.warm(urls);
  }
}
```

`PrefetchJob` takes `ImagePrefetcher? imagePrefetcher` as a constructor
parameter (default `null` in `background_main.dart`'s real registration →
supply `CacheManagerImagePrefetcher()`; tests pass a fake or omit it to
exercise text-only behavior without touching `flutter_cache_manager`).
