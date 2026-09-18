# Breaking-News Alerts (roadmap #7) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Opt-in local notifications when 3+ distinct outlets cover the same fresh story in the user's followed feeds — no backend, Android-only v1.

**Architecture:** A second `BackgroundJob` (`breaking_news_check`) in the existing `core/background/` registry fetches all followed targets every ~15 min (same machinery as `PrefetchJob`), runs a pure headline-clustering detector over the fresh results, and posts via `flutter_local_notifications` behind an `AlertNotifier` interface. Cluster dedup/rate-limit state lives in `breaking_news_*` prefs keys owned by the job.

**Tech Stack:** Flutter/Riverpod, workmanager (existing), flutter_local_notifications (new), shared_preferences.

**Spec:** `docs/superpowers/specs/2026-09-18-breaking-news-alerts-design.md` — the plan argues from the spec; executors read both.

## Global Constraints

- **Android-only v1.** No iOS code paths; the `AlertNotifier`/`BackgroundScheduler` interfaces are the only seams iOS resumption may later touch.
- **No new `@pragma('vm:entry-point')`** — the job rides the existing `callbackDispatcher` in `lib/background_main.dart`.
- **Background write rule:** the job writes ONLY `breaking_news_*` keys on `LocalStorageService`. Reads of `read_article_links`/`muted_source_urls`/etc. are fine. (Tests assert this.)
- **Plugin isolation:** `package:flutter_local_notifications` may be imported only by `lib/features/breaking_news/data/flutter_local_alert_notifier.dart` (mirrors the `workmanager` rule).
- **Honest copy, verbatim** (spec §Settings): subtitle *"Checks your followed topics every ~15 min and tells you when 3+ outlets break the same story. Not real-time."*
- **No Wi-Fi-only switch** for this job; scheduled with `unmeteredOnly: false`.
- **Interval:** `const Duration(minutes: 15)` (workmanager's Android floor), matching prefetch.
- **v1 thresholds** (spec table): lookback 12 h · Jaccard ≥ 0.5 · ≥ 3 distinct outlets (RSS host identity) · newest member < 2 h · ledger TTL 24 h / cap 40 · min gap 60 min · cap 5 per rolling 24 h · max 2 alerts per run.
- **Tap-through reuses the existing flow:** disk-cache article lookup → `router.push('/article', extra: article)` → else `safeArticleUri(link)` + external launch. Never open a non-http(s) link.
- Commit messages end with `Co-Authored-By: Claude Code <noreply@anthropic.com>`.
- Every task: `flutter analyze` clean and full `flutter test` green before committing.

## Deviations from spec (recorded, don't re-litigate)

- Spec says `detect(articles, ledger, config, now)`. Implemented as `detect(..., {required bool Function(String signature) wasAlerted, ...})` — a callback, so the detector stays free of the storage-facing `AlertLedger`. The job wires `ledger.wasAlerted` in. Same behavior, purer detector.
- Spec correction made during planning: `Article` *does* carry `categoryId`; the no-category-label decision still holds (mixed-target clusters make any label a guess) — spec §Notification already updated.

## File Structure

```
lib/core/storage/local_storage_service.dart                          # modify: 4 new keys + accessors
lib/features/breaking_news/domain/breaking_news_config.dart          # create: thresholds
lib/features/breaking_news/domain/headline_text.dart                 # create: tokenizer + Jaccard (pure)
lib/features/breaking_news/domain/breaking_news_story.dart           # create: value object
lib/features/breaking_news/domain/breaking_news_detector.dart        # create: clustering + gates (pure)
lib/features/breaking_news/domain/alert_notifier.dart                # create: interface
lib/features/breaking_news/data/alert_ledger.dart                    # create: dedup + rate-limit state
lib/features/breaking_news/data/flutter_local_alert_notifier.dart    # create: plugin shim (only plugin import)
lib/features/breaking_news/data/breaking_news_job.dart               # create: the BackgroundJob
lib/features/breaking_news/presentation/providers/breaking_news_providers.dart  # create
lib/features/breaking_news/presentation/widgets/breaking_news_tile.dart          # create
lib/background_main.dart                                             # modify: register job
lib/main.dart                                                        # modify: re-assertion clause
lib/app.dart                                                         # modify: notification tap routing
android/app/src/main/AndroidManifest.xml                             # modify: POST_NOTIFICATIONS
android/app/build.gradle.kts                                         # modify: desugaring
pubspec.yaml                                                         # modify: plugin
docs/superpowers/specs, FEATURE_ROADMAP.md, CLAUDE.md                # modify (final task)
test/local_storage_service_test.dart, test/alert_ledger_test.dart,
test/breaking_news_detector_test.dart, test/breaking_news_job_test.dart,
test/breaking_news_tile_test.dart, test/app_test.dart                # test files
```

---

### Task 1: Storage keys on `LocalStorageService`

**Files:**
- Modify: `lib/core/storage/local_storage_service.dart`
- Test: `test/local_storage_service_test.dart` (append)

**Interfaces:**
- Consumes: nothing new.
- Produces (later tasks rely on these exact members):
  - `bool get breakingNewsEnabled` (default `false`) / `Future<void> setBreakingNewsEnabled(bool)`
  - `DateTime? get breakingNewsLastCheckAt` / `Future<void> setBreakingNewsLastCheck(DateTime at)`
  - `String? get breakingNewsLedgerJson` / `Future<void> setBreakingNewsLedgerJson(String json)`
  - `List<String> get breakingNewsAlertTimes` (ISO-8601 strings) / `Future<void> setBreakingNewsAlertTimes(List<String>)`

- [ ] **Step 1: Write the failing tests**

Append to `test/local_storage_service_test.dart` (follow the file's existing `SharedPreferences.setMockInitialValues({})` style):

```dart
group('breaking news keys', () {
  test('breakingNewsEnabled defaults to false', () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    expect(storage.breakingNewsEnabled, isFalse);
  });

  test('breakingNewsEnabled round-trips', () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    await storage.setBreakingNewsEnabled(true);
    expect(storage.breakingNewsEnabled, isTrue);
  });

  test('last-check timestamp round-trips and parses', () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    expect(storage.breakingNewsLastCheckAt, isNull);
    final at = DateTime.utc(2026, 9, 18, 10, 30);
    await storage.setBreakingNewsLastCheck(at);
    expect(storage.breakingNewsLastCheckAt, at);
  });

  test('ledger json round-trips', () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    expect(storage.breakingNewsLedgerJson, isNull);
    await storage.setBreakingNewsLedgerJson('[{"signature":"a b","alertedAt":"2026-09-18T10:00:00Z"}]');
    expect(storage.breakingNewsLedgerJson, contains('a b'));
  });

  test('alert times round-trip as a string list', () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    expect(storage.breakingNewsAlertTimes, isEmpty);
    await storage.setBreakingNewsAlertTimes(['2026-09-18T10:00:00Z']);
    expect(storage.breakingNewsAlertTimes, ['2026-09-18T10:00:00Z']);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `flutter test test/local_storage_service_test.dart`
Expected: FAIL — `breakingNewsEnabled` etc. not defined.

- [ ] **Step 3: Implement**

In `lib/core/storage/local_storage_service.dart`, add to the key constants block:

```dart
  static const _kBreakingNewsEnabledKey = 'breaking_news_enabled';
  static const _kBreakingNewsLedgerKey = 'breaking_news_alert_ledger';
  static const _kBreakingNewsAlertTimesKey = 'breaking_news_alert_times';
  static const _kBreakingNewsLastCheckKey = 'breaking_news_last_check_at';
```

and a section after the offline-prefetch section:

```dart
  // ---- Breaking-news alerts (roadmap #7) ----

  /// Whether the periodic breaking-news job may run at all. Default
  /// `false` — alerts are opt-in (unlike pre-fetch), and the OS
  /// notification permission is requested from the Settings toggle.
  bool get breakingNewsEnabled =>
      _prefs.getBool(_kBreakingNewsEnabledKey) ?? false;

  Future<void> setBreakingNewsEnabled(bool value) =>
      _prefs.setBool(_kBreakingNewsEnabledKey, value);

  /// Raw JSON of alerted-cluster signatures; (de)serialization and
  /// TTL/cap rules live in `AlertLedger`, not here (same split as
  /// custom sources / the remote-catalog blobs).
  String? get breakingNewsLedgerJson =>
      _prefs.getString(_kBreakingNewsLedgerKey);

  Future<void> setBreakingNewsLedgerJson(String json) =>
      _prefs.setString(_kBreakingNewsLedgerKey, json);

  /// ISO-8601 timestamps of recent notifications — the rate-limit
  /// windows (60-min gap, 5-per-24h cap) read this.
  List<String> get breakingNewsAlertTimes =>
      _prefs.getStringList(_kBreakingNewsAlertTimesKey) ?? const [];

  Future<void> setBreakingNewsAlertTimes(List<String> times) =>
      _prefs.setStringList(_kBreakingNewsAlertTimesKey, times);

  /// When the breaking-news job last completed a run (Settings status row).
  DateTime? get breakingNewsLastCheckAt {
    final raw = _prefs.getString(_kBreakingNewsLastCheckKey);
    return raw == null ? null : DateTime.tryParse(raw);
  }

  Future<void> setBreakingNewsLastCheck(DateTime at) => _prefs.setString(
        _kBreakingNewsLastCheckKey,
        at.toUtc().toIso8601String(),
      );
```

- [ ] **Step 4: Run to verify pass**

Run: `flutter test test/local_storage_service_test.dart`
Expected: PASS (all, including pre-existing).

- [ ] **Step 5: Commit**

```bash
git add lib/core/storage/local_storage_service.dart test/local_storage_service_test.dart
git commit -m "feat(breaking-news): storage keys for alert state

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 2: `AlertLedger` — dedup + rate-limit state

**Files:**
- Create: `lib/features/breaking_news/data/alert_ledger.dart`
- Test: `test/alert_ledger_test.dart`

**Interfaces:**
- Consumes: `LocalStorageService.breakingNewsLedgerJson` / `breakingNewsAlertTimes` + setters (Task 1).
- Produces:
  ```dart
  class AlertLedger {
    AlertLedger(LocalStorageService storage);
    bool wasAlerted(String signature, {required DateTime now, required Duration ttl});
    Future<void> record(String signature, {required DateTime now, required Duration ttl, required int cap});
    bool canNotifyNow({required DateTime now, required Duration minGap, required Duration window, required int windowCap});
    Future<void> recordNotification({required DateTime now, required Duration window});
  }
  ```

- [ ] **Step 1: Write the failing tests**

Create `test/alert_ledger_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/storage/local_storage_service.dart';
import 'package:newsfeed/features/breaking_news/data/alert_ledger.dart';
import 'package:shared_preferences/shared_preferences.dart';

const _ttl = Duration(hours: 24);
const _gap = Duration(minutes: 60);
const _window = Duration(hours: 24);
const _cap = 5;

void main() {
  late LocalStorageService storage;
  late AlertLedger ledger;
  final now = DateTime.utc(2026, 9, 18, 12, 0);

  setUp(() async {
    SharedPreferences.setMockInitialValues({});
    storage = LocalStorageService(await SharedPreferences.getInstance());
    ledger = AlertLedger(storage);
  });

  group('signature ledger', () {
    test('unrecorded signature is not alerted', () {
      expect(ledger.wasAlerted('a b c', now: now, ttl: _ttl), isFalse);
    });

    test('recorded signature is alerted within ttl', () async {
      await ledger.record('a b c', now: now, ttl: _ttl, cap: 40);
      expect(ledger.wasAlerted('a b c', now: now, ttl: _ttl), isTrue);
    });

    test('signature expires after ttl', () async {
      await ledger.record('a b c', now: now, ttl: _ttl, cap: 40);
      final later = now.add(_ttl + const Duration(minutes: 1));
      expect(ledger.wasAlerted('a b c', now: later, ttl: _ttl), isFalse);
    });

    test('recording prunes expired entries', () async {
      await ledger.record('old', now: now, ttl: _ttl, cap: 40);
      final later = now.add(_ttl + const Duration(minutes: 1));
      await ledger.record('new', now: later, ttl: _ttl, cap: 40);
      // 'old' is gone from storage; 'new' survives.
      expect(ledger.wasAlerted('old', now: later, ttl: _ttl), isFalse);
      expect(ledger.wasAlerted('new', now: later, ttl: _ttl), isTrue);
    });

    test('cap evicts the oldest entries', () async {
      for (var i = 0; i < 42; i++) {
        final at = now.add(Duration(minutes: i));
        await ledger.record('sig-$i', now: at, ttl: _ttl, cap: 40);
      }
      expect(ledger.wasAlerted('sig-0', now: now, ttl: _ttl), isFalse);
      expect(ledger.wasAlerted('sig-1', now: now, ttl: _ttl), isFalse);
      expect(ledger.wasAlerted('sig-41', now: now, ttl: _ttl), isTrue);
    });

    test('corrupt ledger json is treated as empty, not thrown', () async {
      await storage.setBreakingNewsLedgerJson('not json at all');
      expect(ledger.wasAlerted('a b c', now: now, ttl: _ttl), isFalse);
    });
  });

  group('notification rate limits', () {
    test('first notification is always allowed', () {
      expect(
        ledger.canNotifyNow(now: now, minGap: _gap, window: _window, windowCap: _cap),
        isTrue,
      );
    });

    test('blocked within the min gap', () async {
      await ledger.recordNotification(now: now.subtract(const Duration(minutes: 30)), window: _window);
      expect(
        ledger.canNotifyNow(now: now, minGap: _gap, window: _window, windowCap: _cap),
        isFalse,
      );
    });

    test('allowed once the gap has passed', () async {
      await ledger.recordNotification(now: now.subtract(_gap), window: _window);
      expect(
        ledger.canNotifyNow(now: now, minGap: _gap, window: _window, windowCap: _cap),
        isTrue,
      );
    });

    test('blocked at the rolling-window cap', () async {
      for (var i = 1; i <= _cap; i++) {
        await ledger.recordNotification(
          now: now.subtract(Duration(hours: i)),
          window: _window,
        );
      }
      expect(
        ledger.canNotifyNow(now: now, minGap: _gap, window: _window, windowCap: _cap),
        isFalse,
      );
    });

    test('old timestamps leave the window and free a slot', () async {
      for (var i = 1; i <= _cap; i++) {
        await ledger.recordNotification(
          now: now.subtract(Duration(hours: 24 + i)),
          window: _window,
        );
      }
      expect(
        ledger.canNotifyNow(now: now, minGap: _gap, window: _window, windowCap: _cap),
        isTrue,
      );
    });

    test('recordNotification prunes timestamps outside the window', () async {
      await ledger.recordNotification(now: now.subtract(_window * 2), window: _window);
      await ledger.recordNotification(now: now, window: _window);
      expect(storage.breakingNewsAlertTimes.length, 1);
    });
  });
}
```

- [ ] **Step 2: Run to verify failure**

Run: `flutter test test/alert_ledger_test.dart`
Expected: FAIL — `alert_ledger.dart` doesn't exist.

- [ ] **Step 3: Implement**

Create `lib/features/breaking_news/data/alert_ledger.dart`:

```dart
import 'dart:convert';

import '../../../core/storage/local_storage_service.dart';

/// Alerted-cluster memory and notification rate-limit windows, over the
/// `breaking_news_*` storage keys. Pure with respect to time — every call
/// takes `now` — so callers (job, tests) control the clock.
///
/// Belongs to the background job's owned-key set (see PrefetchJob's
/// write rule): the foreground never writes these.
class AlertLedger {
  AlertLedger(this._storage);
  final LocalStorageService _storage;

  static const _kSignature = 'signature';
  static const _kAlertedAt = 'alertedAt';

  /// Whether [signature] has an alive (non-expired) ledger entry.
  bool wasAlerted(String signature, {required DateTime now, required Duration ttl}) {
    return _entries(now: now, ttl: ttl).any((e) => e[_kSignature] == signature);
  }

  /// Records [signature], dropping expired entries first and evicting the
  /// oldest once [cap] alive entries are exceeded.
  Future<void> record(String signature, {required DateTime now, required Duration ttl, required int cap}) async {
    final alive = _entries(now: now, ttl: ttl).toList()
      ..removeWhere((e) => e[_kSignature] == signature)
      ..add({_kSignature: signature, _kAlertedAt: now.toUtc().toIso8601String()});
    final evicted = alive.length > cap ? alive.sublist(alive.length - cap) : alive;
    await _write(evicted);
  }

  /// Whether the min-gap and rolling-window cap both allow a notification
  /// right now. Boundary-inclusive: exactly [minGap] since the last
  /// notification, or exactly [windowCap] notifications in [window, blocks.
  bool canNotifyNow({
    required DateTime now,
    required Duration minGap,
    required Duration window,
    required int windowCap,
  }) {
    final times = _times(now: now, window: window);
    if (times.length >= windowCap) return false;
    final last = times.isEmpty ? null : times.last;
    if (last != null && now.toUtc().difference(last) < minGap) return false;
    return true;
  }

  /// Records a notification, pruning timestamps that left [window].
  Future<void> recordNotification({required DateTime now, required Duration window}) async {
    final times = _times(now: now, window: window).toList()..add(now.toUtc());
    await _storage.setBreakingNewsAlertTimes(
        times.map((t) => t.toIso8601String()).toList());
  }

  Iterable<Map<String, dynamic>> _entries({required DateTime now, required Duration ttl}) sync* {
    final raw = _storage.breakingNewsLedgerJson;
    if (raw == null || raw.isEmpty) return;
    final List<dynamic> decoded;
    try {
      decoded = jsonDecode(raw) as List<dynamic>;
    } catch (_) {
      return; // corrupt blob — treat as empty; the next record() overwrites it
    }
    for (final e in decoded) {
      if (e is! Map<String, dynamic>) continue;
      final at = DateTime.tryParse(e[_kAlertedAt] as String? ?? '');
      if (at == null) continue;
      if (now.toUtc().difference(at.toUtc()) > ttl) continue;
      yield e;
    }
  }

  List<DateTime> _times({required DateTime now, required Duration window}) {
    final cutoff = now.toUtc().subtract(window);
    final times = _storage.breakingNewsAlertTimes
        .map(DateTime.tryParse)
        .whereType<DateTime>()
        .map((t) => t.toUtc())
        .where((t) => t.isAfter(cutoff))
        .toList()
      ..sort();
    return times;
  }

  Future<void> _write(List<Map<String, dynamic>> entries) =>
      _storage.setBreakingNewsLedgerJson(jsonEncode(entries));
}
```

- [ ] **Step 4: Run to verify pass**

Run: `flutter test test/alert_ledger_test.dart`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/features/breaking_news/data/alert_ledger.dart test/alert_ledger_test.dart
git commit -m "feat(breaking-news): alert ledger with dedup + rate limits

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 3: Config + tokenizer + similarity (pure)

**Files:**
- Create: `lib/features/breaking_news/domain/breaking_news_config.dart`
- Create: `lib/features/breaking_news/domain/headline_text.dart`
- Test: `test/breaking_news_detector_test.dart` (new file; detector tests land here in Task 4)

**Interfaces:**
- Produces:
  ```dart
  class BreakingNewsConfig {
    const BreakingNewsConfig({
      this.lookbackWindow = const Duration(hours: 12),
      this.jaccardThreshold = 0.5,
      this.minDistinctOutlets = 3,
      this.maxNewestAge = const Duration(hours: 2),
      this.ledgerTtl = const Duration(hours: 24),
      this.ledgerCap = 40,
      this.minGapBetweenAlerts = const Duration(minutes: 60),
      this.rollingWindow = const Duration(hours: 24),
      this.rollingWindowCap = 5,
      this.maxAlertsPerRun = 2,
    });
    // ... final fields, spec's v1 defaults
  }
  Set<String> headlineTokens(String title);
  double jaccard(Set<String> a, Set<String> b);
  ```

- [ ] **Step 1: Write the failing tests**

Create `test/breaking_news_detector_test.dart` with just the tokenizer/similarity group for now:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/features/breaking_news/domain/headline_text.dart';

void main() {
  group('headlineTokens', () {
    test('lowercases, strips punctuation, drops stopwords', () {
      expect(headlineTokens('The Central Bank Raises Rates, Again!'),
          {'central', 'bank', 'raises', 'rates'});
    });

    test('drops single-character and pure-numeric tokens', () {
      expect(headlineTokens('A G7 win by 10 players, I saw it'),
          {'g7', 'win', 'players', 'saw'});
    });

    test('handles Hebrew-script headlines without throwing', () {
      final tokens = headlineTokens('ראש הממשלה נפגש עם הנשיא בירושלים');
      expect(tokens, containsAll(['ראש', 'הממשלה', 'נפגש']));
    });

    test('empty/garbage titles yield empty sets', () {
      expect(headlineTokens(''), isEmpty);
      expect(headlineTokens('... !!! ???'), isEmpty);
    });
  });

  group('jaccard', () {
    test('identical sets score 1.0', () {
      expect(jaccard({'a', 'b'}, {'b', 'a'}), 1.0);
    });

    test('disjoint sets score 0.0', () {
      expect(jaccard({'a', 'b'}, {'c', 'd'}), 0.0);
    });

    test('empty set scores 0.0 regardless', () {
      expect(jaccard(<String>{}, {'a'}), 0.0);
      expect(jaccard(<String>{}, <String>{}), 0.0);
    });

    test('boundary: 2 of 4 shared tokens scores exactly 0.5', () {
      expect(jaccard({'a', 'b', 'c'}, {'a', 'b', 'd'}), 0.5);
    });
  });
}
```

- [ ] **Step 2: Run to verify failure**

Run: `flutter test test/breaking_news_detector_test.dart`
Expected: FAIL — `headline_text.dart` doesn't exist.

- [ ] **Step 3: Implement**

`lib/features/breaking_news/domain/breaking_news_config.dart`:

```dart
/// All v1 detection/rate thresholds in one place (spec: "Detection
/// heuristic"). Tuning expected — restructuring not — if real-world
/// precision is off; see spec's cross-language note.
class BreakingNewsConfig {
  const BreakingNewsConfig({
    this.lookbackWindow = const Duration(hours: 12),
    this.jaccardThreshold = 0.5,
    this.minDistinctOutlets = 3,
    this.maxNewestAge = const Duration(hours: 2),
    this.ledgerTtl = const Duration(hours: 24),
    this.ledgerCap = 40,
    this.minGapBetweenAlerts = const Duration(minutes: 60),
    this.rollingWindow = const Duration(hours: 24),
    this.rollingWindowCap = 5,
    this.maxAlertsPerRun = 2,
  });

  /// Articles older than this are invisible to detection.
  final Duration lookbackWindow;

  /// Headline token-set similarity at/above which two articles are the
  /// same story.
  final double jaccardThreshold;

  /// Distinct RSS-host outlets a cluster needs to count as "breaking".
  final int minDistinctOutlets;

  /// The cluster's newest article must be younger than this.
  final Duration maxNewestAge;

  final Duration ledgerTtl;
  final int ledgerCap;
  final Duration minGapBetweenAlerts;
  final Duration rollingWindow;
  final int rollingWindowCap;
  final int maxAlertsPerRun;
}
```

`lib/features/breaking_news/domain/headline_text.dart`:

```dart
/// Headline tokenization + similarity for the breaking-news detector.
/// Pure string math — no article knowledge, no I/O.
library;

/// Small English stopword list. Deliberately minimal: cross-language
/// clusters share proper nouns regardless of script (spec), and a longer
/// list risks eating meaningful headline words.
const _stopwords = {
  'the', 'a', 'an', 'and', 'or', 'but', 'of', 'in', 'on', 'at', 'to', 'for',
  'from', 'by', 'with', 'as', 'is', 'are', 'was', 'were', 'be', 'been',
  'am', 'it', 'its', 'this', 'that', 'these', 'those', 'his', 'her', 'their',
  'after', 'before', 'over', 'under', 'into', 'about', 'again', 'not', 'no',
};

final _splitRe = RegExp(r'[^\p{L}\p{N}]+', unicode: true);
final _numericRe = RegExp(r'^\d+$');

/// Title → comparable token set: lowercased, punctuation-split,
/// stopwords/single characters/pure numbers dropped. Unicode-aware, so
/// Hebrew/Arabic/CJK headlines tokenize on their own letters.
Set<String> headlineTokens(String title) {
  return {
    for (final raw in title.toLowerCase().split(_splitRe))
      if (raw.length > 1 && !_stopwords.contains(raw) && !_numericRe.hasMatch(raw))
        raw,
  };
}

/// |A∩B| / |A∪B| — 1.0 identical, 0.0 disjoint, 0.0 if either side empty.
double jaccard(Set<String> a, Set<String> b) {
  if (a.isEmpty || b.isEmpty) return 0.0;
  return a.intersection(b).length / a.union(b).length;
}
```

- [ ] **Step 4: Run to verify pass**

Run: `flutter test test/breaking_news_detector_test.dart`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/features/breaking_news/domain/breaking_news_config.dart \
  lib/features/breaking_news/domain/headline_text.dart test/breaking_news_detector_test.dart
git commit -m "feat(breaking-news): config, headline tokenizer, jaccard similarity

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 4: `BreakingNewsStory` + the detector (pure clustering + gates)

**Files:**
- Create: `lib/features/breaking_news/domain/breaking_news_story.dart`
- Create: `lib/features/breaking_news/domain/breaking_news_detector.dart`
- Test: `test/breaking_news_detector_test.dart` (append)

**Interfaces:**
- Consumes: `Article` (`title`, `link`, `sourceName`, `sourceUrl`, `publishedAt`), `headlineTokens`/`jaccard` (Task 3), `BreakingNewsConfig` (Task 3).
- Produces:
  ```dart
  class BreakingNewsStory {
    final String headline;      // newest member's title
    final String link;          // freshest unread member's link
    final List<String> outletNames; // one per distinct host, newest-first
    final String signature;     // dedup identity (top-5 tokens joined)
    String get bodyText;        // "<A>, <B> +N more reporting"
  }
  class BreakingNewsDetector {
    const BreakingNewsDetector();
    List<BreakingNewsStory> detect({
      required List<Article> articles,
      required Set<String> readLinks,
      required bool Function(String signature) wasAlerted,
      required DateTime now,
      BreakingNewsConfig config = const BreakingNewsConfig(),
    });
  }
  ```

- [ ] **Step 1: Write the failing tests**

Append to `test/breaking_news_detector_test.dart` (add imports for `Article`, `BreakingNewsStory`, `BreakingNewsDetector`, `BreakingNewsConfig`):

```dart
// --- helpers ---
Article _article({
  required String title,
  required String link,
  required String sourceName,
  required String sourceUrl,
  DateTime? publishedAt,
}) {
  return Article(
    id: link,
    title: title,
    summary: '',
    content: '',
    link: link,
    imageUrl: null,
    sourceName: sourceName,
    sourceUrl: sourceUrl,
    categoryId: 'world',
    publishedAt: publishedAt,
    language: 'en',
    sourceType: SourceType.international,
  );
}

DateTime _ago(DateTime now, Duration d) => now.subtract(d);

group('BreakingNewsDetector.detect', () {
  const detector = BreakingNewsDetector();
  final now = DateTime.utc(2026, 9, 18, 12, 0);

  test('never alerted when nothing qualifies', () {
    final stories = detector.detect(
      articles: const [],
      readLinks: const {},
      wasAlerted: (_) => false,
      now: now,
    );
    expect(stories, isEmpty);
  });

  test('3 distinct outlets on one story alerts', () {
    final stories = detector.detect(
      articles: [
        _article(
            title: 'Central bank raises rates amid inflation',
            link: 'l1', sourceName: 'BBC', sourceUrl: 'https://bbc.co.uk/feed',
            publishedAt: _ago(now, const Duration(minutes: 30))),
        _article(
            title: 'Central bank raises rates amid inflation spike',
            link: 'l2', sourceName: 'NYT', sourceUrl: 'https://nytimes.com/feed',
            publishedAt: _ago(now, const Duration(minutes: 20))),
        _article(
            title: 'Central bank raises rates amid inflation fears',
            link: 'l3', sourceName: 'Guardian', sourceUrl: 'https://theguardian.com/feed',
            publishedAt: _ago(now, const Duration(minutes: 10))),
      ],
      readLinks: const {},
      wasAlerted: (_) => false,
      now: now,
    );
    expect(stories, hasLength(1));
    expect(stories.single.headline,
        'Central bank raises rates amid inflation fears'); // newest member
    expect(stories.single.link, 'l3');
    expect(stories.single.outletNames, ['Guardian', 'NYT', 'BBC']); // newest-first
  });

  test('same outlet via two feeds counts as ONE outlet', () {
    // BBC Technology + BBC Business = same host = 2 distinct outlets → no alert.
    final stories = detector.detect(
      articles: [
        _article(title: 'Magnitude 7 quake hits coast', link: 'a',
            sourceName: 'BBC Technology', sourceUrl: 'https://bbc.co.uk/tech',
            publishedAt: _ago(now, const Duration(minutes: 30))),
        _article(title: 'Magnitude 7 quake hits coast', link: 'b',
            sourceName: 'BBC Business', sourceUrl: 'https://bbc.co.uk/business',
            publishedAt: _ago(now, const Duration(minutes: 25))),
        _article(title: 'Entirely different headline about sports', link: 'c',
            sourceName: 'ESPN', sourceUrl: 'https://espn.com/feed',
            publishedAt: _ago(now, const Duration(minutes: 20))),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, isEmpty);
  });

  test('below threshold similarity stays unclustered (needs 3 outlets)', () {
    final stories = detector.detect(
      articles: [
        _article(title: 'Stocks rally on earnings', link: 'a',
            sourceName: 'A', sourceUrl: 'https://a.com/f',
            publishedAt: _ago(now, const Duration(minutes: 30))),
        _article(title: 'Completely unrelated weather news today', link: 'b',
            sourceName: 'B', sourceUrl: 'https://b.com/f',
            publishedAt: _ago(now, const Duration(minutes: 29))),
        _article(title: 'Another unrelated sports result last night', link: 'c',
            sourceName: 'C', sourceUrl: 'https://c.com/f',
            publishedAt: _ago(now, const Duration(minutes: 28))),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, isEmpty);
  });

  test('cluster whose newest member is too old does not alert', () {
    final stories = detector.detect(
      articles: [
        _article(title: 'Central bank raises rates amid inflation', link: 'a',
            sourceName: 'A', sourceUrl: 'https://a.com/f',
            publishedAt: _ago(now, const Duration(hours: 3))),
        _article(title: 'Central bank raises rates amid inflation', link: 'b',
            sourceName: 'B', sourceUrl: 'https://b.com/f',
            publishedAt: _ago(now, const Duration(hours: 4))),
        _article(title: 'Central bank raises rates amid inflation', link: 'c',
            sourceName: 'C', sourceUrl: 'https://c.com/f',
            publishedAt: _ago(now, const Duration(hours: 5))),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, isEmpty);
  });

  test('articles older than the lookback window are invisible', () {
    final stories = detector.detect(
      articles: List.generate(3, (i) => _article(
          title: 'Central bank raises rates amid inflation',
          link: 'l$i', sourceName: 'S$i', sourceUrl: 'https://s$i.com/f',
          publishedAt: _ago(now, const Duration(hours: 13)))),
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, isEmpty);
  });

  test('articles with no timestamp are excluded', () {
    final stories = detector.detect(
      articles: [
        _article(title: 'Central bank raises rates amid inflation', link: 'a',
            sourceName: 'A', sourceUrl: 'https://a.com/f', publishedAt: null),
        _article(title: 'Central bank raises rates amid inflation', link: 'b',
            sourceName: 'B', sourceUrl: 'https://b.com/f', publishedAt: null),
        _article(title: 'Central bank raises rates amid inflation', link: 'c',
            sourceName: 'C', sourceUrl: 'https://c.com/f', publishedAt: null),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, isEmpty);
  });

  test('alerted signature suppresses the cluster', () {
    final stories = detector.detect(
      articles: [/* the 3-outlet qualifying set from above */],
      readLinks: const {},
      wasAlerted: (sig) => sig == <the cluster's signature>, // compute once via a first detect() call
      now: now,
    );
    expect(stories, isEmpty);
  });

  test('all-read clusters suppress; partially-read alert with freshest unread link', () {
    final articles = [
      _article(title: 'Central bank raises rates amid inflation', link: 'a',
          sourceName: 'A', sourceUrl: 'https://a.com/f',
          publishedAt: _ago(now, const Duration(minutes: 30))),
      _article(title: 'Central bank raises rates amid inflation', link: 'b',
          sourceName: 'B', sourceUrl: 'https://b.com/f',
          publishedAt: _ago(now, const Duration(minutes: 20))),
      _article(title: 'Central bank raises rates amid inflation', link: 'c',
          sourceName: 'C', sourceUrl: 'https://c.com/f',
          publishedAt: _ago(now, const Duration(minutes: 10))),
    ];
    final allRead = detector.detect(
      articles: articles,
      readLinks: {'a', 'b', 'c'},
      wasAlerted: (_) => false, now: now,
    );
    expect(allRead, isEmpty);

    final partial = detector.detect(
      articles: articles,
      readLinks: {'a', 'b'},
      wasAlerted: (_) => false, now: now,
    );
    expect(partial, hasLength(1));
    expect(partial.single.link, 'c'); // freshest unread
  });

  test('maxAlertsPerRun caps, ranked freshest-first', () {
    // Two independent qualifying clusters (6 outlets, two stories),
    // first (older) cluster should lose.
    final stories = detector.detect(
      articles: [
        // older cluster (newest 40 min ago)
        _article(title: 'Football club fires head coach', link: 'f1',
            sourceName: 'A', sourceUrl: 'https://a.com/f',
            publishedAt: _ago(now, const Duration(minutes: 40))),
        _article(title: 'Football club fires head coach', link: 'f2',
            sourceName: 'B', sourceUrl: 'https://b.com/f',
            publishedAt: _ago(now, const Duration(minutes: 39))),
        _article(title: 'Football club fires head coach', link: 'f3',
            sourceName: 'C', sourceUrl: 'https://c.com/f',
            publishedAt: _ago(now, const Duration(minutes: 38))),
        // newer cluster (newest 5 min ago)
        _article(title: 'Central bank raises rates amid inflation', link: 'm1',
            sourceName: 'D', sourceUrl: 'https://d.com/f',
            publishedAt: _ago(now, const Duration(minutes: 6))),
        _article(title: 'Central bank raises rates amid inflation', link: 'm2',
            sourceName: 'E', sourceUrl: 'https://e.com/f',
            publishedAt: _ago(now, const Duration(minutes: 5))),
        _article(title: 'Central bank raises rates amid inflation', link: 'm3',
            sourceName: 'F', sourceUrl: 'https://f.com/f',
            publishedAt: _ago(now, const Duration(minutes: 4))),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, hasLength(2));
    expect(stories.first.headline, contains('Central bank'));
    expect(stories.last.headline, contains('Football'));
  });

  test('bodyText renders two names then +N more', () {
    final stories = detector.detect(
      articles: [
        _article(title: 'Central bank raises rates amid inflation', link: 'a',
            sourceName: 'Alpha', sourceUrl: 'https://a.com/f',
            publishedAt: _ago(now, const Duration(minutes: 30))),
        _article(title: 'Central bank raises rates amid inflation', link: 'b',
            sourceName: 'Beta', sourceUrl: 'https://b.com/f',
            publishedAt: _ago(now, const Duration(minutes: 20))),
        _article(title: 'Central bank raises rates amid inflation', link: 'c',
            sourceName: 'Gamma', sourceUrl: 'https://c.com/f',
            publishedAt: _ago(now, const Duration(minutes: 10))),
        _article(title: 'Central bank raises rates amid inflation', link: 'd',
            sourceName: 'Delta', sourceUrl: 'https://d.com/f',
            publishedAt: _ago(now, const Duration(minutes: 5))),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories.single.bodyText, 'Delta, Gamma +2 more reporting');
  });

  test('cross-language cluster on shared proper nouns alerts', () {
    // Hebrew + English headlines sharing the proper noun cluster.
    final stories = detector.detect(
      articles: [
        _article(title: 'SpaceX launch delayed by weather', link: 'a',
            sourceName: 'A', sourceUrl: 'https://a.com/f',
            publishedAt: _ago(now, const Duration(minutes: 30))),
        _article(title: 'SpaceX launch delayed again at Cape Canaveral', link: 'b',
            sourceName: 'B', sourceUrl: 'https://b.com/f',
            publishedAt: _ago(now, const Duration(minutes: 20))),
        _article(title: 'חללית SpaceX נדחתה בשל מזג האוויר', link: 'c',
            sourceName: 'C', sourceUrl: 'https://c.com/f',
            publishedAt: _ago(now, const Duration(minutes: 10))),
      ],
      readLinks: const {}, wasAlerted: (_) => false, now: now,
    );
    expect(stories, hasLength(1));
  });
});
```

Note on the "alerted signature suppresses" test: call `detect` once with `wasAlerted: (_) => false`, capture `stories.single.signature`, then call again with `wasAlerted: (s) => s == captured`. Write it that way — no hardcoded signature literal.

- [ ] **Step 2: Run to verify failure**

Run: `flutter test test/breaking_news_detector_test.dart`
Expected: FAIL — `breaking_news_story.dart` / `breaking_news_detector.dart` don't exist.

- [ ] **Step 3: Implement**

`lib/features/breaking_news/domain/breaking_news_story.dart`:

```dart
/// One qualifying breaking-news cluster, ready to render as a notification.
class BreakingNewsStory {
  const BreakingNewsStory({
    required this.headline,
    required this.link,
    required this.outletNames,
    required this.signature,
  });

  /// Newest member's headline — the notification title.
  final String headline;

  /// Freshest unread member's link — the payload/tap target.
  final String link;

  /// One display name per distinct outlet (RSS host), newest-first.
  final List<String> outletNames;

  /// Dedup identity: the cluster's top-5 tokens, sorted, joined.
  final String signature;

  /// Spec: no category label (clusters legitimately span targets).
  String get bodyText {
    final a = outletNames[0];
    final b = outletNames.length > 1 ? outletNames[1] : a;
    final more = outletNames.length - 2;
    return more > 0 ? '$a, $b +$more more reporting' : '$a, $b reporting';
  }
}
```

`lib/features/breaking_news/domain/breaking_news_detector.dart`:

```dart
import '../../../features/feed/domain/entities/article.dart';
import 'breaking_news_config.dart';
import 'breaking_news_story.dart';
import 'headline_text.dart';

/// Pure breaking-news detection over already-fetched articles (spec:
/// "Detection heuristic"). Deterministic given its inputs — `now` is
/// always injected. Never throws on malformed articles; they're filtered.
class BreakingNewsDetector {
  const BreakingNewsDetector();

  List<BreakingNewsStory> detect({
    required List<Article> articles,
    required Set<String> readLinks,
    required bool Function(String signature) wasAlerted,
    required DateTime now,
    BreakingNewsConfig config = const BreakingNewsConfig(),
  }) {
    final utcNow = now.toUtc();
    final fresh = <_Candidate>[];
    for (final a in articles) {
      final published = a.publishedAt?.toUtc();
      if (published == null) continue;
      final age = utcNow.difference(published);
      if (age.isNegative || age > config.lookbackWindow) continue;
      fresh.add(_Candidate(
        article: a,
        tokens: headlineTokens(a.title),
        published: published,
      ));
    }
    if (fresh.length < config.minDistinctOutlets) return const [];

    // Greedy union-find over pairwise headline similarity.
    final parent = List<int>.generate(fresh.length, (i) => i);
    int find(int i) => parent[i] == i ? i : (parent[i] = find(parent[i]));
    void union(int i, int j) => parent[find(i)] = find(parent[j]);
    for (var i = 0; i < fresh.length; i++) {
      for (var j = i + 1; j < fresh.length; j++) {
        if (jaccard(fresh[i].tokens, fresh[j].tokens) >=
            config.jaccardThreshold) {
          union(i, j);
        }
      }
    }

    final clusters = <int, List<_Candidate>>{};
    for (var i = 0; i < fresh.length; i++) {
      clusters.putIfAbsent(find(i), () => []).add(fresh[i]);
    }

    final stories = <(DateTime, BreakingNewsStory)>[];
    for (final members in clusters.values) {
      final story = _qualify(members, readLinks, wasAlerted, utcNow, config);
      if (story != null) stories.add((members.map((m) => m.published).reduce(_newer), story));
    }

    stories.sort((x, y) => y.$1.compareTo(x.$1)); // freshest cluster first
    return stories.take(config.maxAlertsPerRun).map((s) => s.$2).toList();
  }

  /// Applies every gate; null = not breaking / suppressed.
  BreakingNewsStory? _qualify(
    List<_Candidate> members,
    Set<String> readLinks,
    bool Function(String) wasAlerted,
    DateTime utcNow,
    BreakingNewsConfig config,
  ) {
    // Outlet = distinct host of the source feed URL (BBC Tech + BBC
    // Business = one outlet).
    final byHost = <String, List<_Candidate>>{};
    for (final m in members) {
      final host = _outletHost(m.article.sourceUrl);
      if (host == null) continue;
      byHost.putIfAbsent(host, () => []).add(m);
    }
    if (byHost.length < config.minDistinctOutlets) return null;

    members.sort((a, b) => b.published.compareTo(a.published)); // newest-first
    final newest = members.first;
    if (utcNow.difference(newest.published) > config.maxNewestAge) return null;

    final signature = _signature(members);
    if (wasAlerted(signature)) return null;

    // Link the freshest *unread* member; all-read → nothing new to open.
    final unread = members.where((m) => !readLinks.contains(m.article.link)).toList();
    if (unread.isEmpty) return null;

    final outletNames = [
      for (final candidates in byHost.values)
        (candidates.toList()
              ..sort((a, b) => b.published.compareTo(a.published)))
            .first
            .article
            .sourceName,
    ];
    // outletNames currently in byHost insertion order — reorder newest-first
    // by each host's newest member:
    final hostNewest = <String, DateTime>{};
    byHost.forEach((host, list) {
      hostNewest[host] = list.map((m) => m.published).reduce(_newer);
    });
    final orderedHosts = hostNewest.keys.toList()
      ..sort((a, b) => hostNewest[b]!.compareTo(hostNewest[a]!));

    return BreakingNewsStory(
      headline: newest.article.title,
      link: (unread.toList()
            ..sort((a, b) => b.published.compareTo(a.published)))
          .first
          .article
          .link,
      outletNames: [for (final h in orderedHosts) byHost[h]!.first.article.sourceName],
      signature: signature,
    );
  }

  /// Normalized host of the source feed URL, `www.` stripped; null when
  /// unparseable (such members don't count toward any outlet).
  String? _outletHost(String sourceUrl) {
    final host = Uri.tryParse(sourceUrl)?.host.toLowerCase();
    if (host == null || host.isEmpty) return null;
    return host.startsWith('www.') ? host.substring(4) : host;
  }

  /// Top-5 most frequent tokens across the cluster, sorted, joined.
  /// Stable across member churn (unlike hashing member links); a collision
  /// costs one suppressed duplicate alert, never a false one (spec).
  String _signature(List<_Candidate> members) {
    final freq = <String, int>{};
    for (final m in members) {
      for (final t in m.tokens) {
        freq[t] = (freq[t] ?? 0) + 1;
      }
    }
    final top = freq.keys.toList()
      ..sort((a, b) {
        final byCount = freq[b]!.compareTo(freq[a]!);
        return byCount != 0 ? byCount : a.compareTo(b);
      });
    return top.take(5).toList()..sort().join(' ');
  }
}

DateTime _newer(DateTime a, DateTime b) => a.isAfter(b) ? a : b;

class _Candidate {
  const _Candidate({
    required this.article,
    required this.tokens,
    required this.published,
  });
  final Article article;
  final Set<String> tokens;
  final DateTime published;
}
```

Wait — one redundancy above: `outletNames` is built twice; keep ONLY the `orderedHosts` version (delete the first `outletNames` list). The final code is:

```dart
    final hostNewest = <String, DateTime>{};
    final hostFirstSource = <String, String>{};
    for (final entry in byHost.entries) {
      final sorted = entry.value.toList()
        ..sort((a, b) => b.published.compareTo(a.published));
      hostNewest[entry.key] = sorted.first.published;
      hostFirstSource[entry.key] = sorted.first.article.sourceName;
    }
    final orderedHosts = hostNewest.keys.toList()
      ..sort((a, b) => hostNewest[b]!.compareTo(hostNewest[a]!));
```

and the story gets `outletNames: [for (final h in orderedHosts) hostFirstSource[h]!]`.

- [ ] **Step 4: Run to verify pass**

Run: `flutter test test/breaking_news_detector_test.dart`
Expected: PASS. (If the "3 distinct outlets" newest-first name order differs, fix the implementation, not the test — spec says newest-first.)

- [ ] **Step 5: Run the full suite**

Run: `flutter analyze && flutter test`
Expected: clean.

- [ ] **Step 6: Commit**

```bash
git add lib/features/breaking_news/domain/ test/breaking_news_detector_test.dart
git commit -m "feat(breaking-news): pure cluster detector with outlet/freshness gates

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 5: `AlertNotifier` interface + plugin shim + platform setup

**Files:**
- Create: `lib/features/breaking_news/domain/alert_notifier.dart`
- Create: `lib/features/breaking_news/data/flutter_local_alert_notifier.dart`
- Modify: `pubspec.yaml` (add `flutter_local_notifications`)
- Modify: `android/app/build.gradle.kts` (core-library desugaring — the plugin requires it)
- Modify: `android/app/src/main/AndroidManifest.xml` (POST_NOTIFICATIONS)

**Interfaces:**
- Produces:
  ```dart
  abstract interface class AlertNotifier {
    Future<bool> areNotificationsEnabled();   // true when nothing blocks posting
    Future<bool> requestPermission();         // true if granted (auto-true < Android 13)
    Future<void> show(BreakingNewsStory story);
  }
  // concrete class (used by app.dart for taps):
  class FlutterLocalAlertNotifier implements AlertNotifier {
    Future<void> ensureInitialized();
    Future<String?> coldStartPayload();       // null unless launched by a notification tap
    Stream<String> get tapPayloads;           // warm-tap payloads (article links)
  }
  ```

- [ ] **Step 1: Add the dependency**

Run: `flutter pub add flutter_local_notifications`
(Latest stable; the APIs used below exist across v14→19. Record the resolved version in the commit body.)

- [ ] **Step 2: Enable core-library desugaring** (`android/app/build.gradle.kts`)

In `android { compileOptions { ... } }` add `isCoreLibraryDesugaringEnabled = true`, and add a top-level dependencies block (after the `android {}` block):

```kotlin
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
        isCoreLibraryDesugaringEnabled = true
    }
    ...
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.5")
}
```

- [ ] **Step 3: Add the permission** (`android/app/src/main/AndroidManifest.xml`)

```xml
    <uses-permission android:name="android.permission.INTERNET"/>
    <!-- Breaking-news alerts (roadmap #7): Android 13+ runtime notification
         permission. Requested only from the Settings toggle tap — never at
         launch. See features/breaking_news/. -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
```

- [ ] **Step 4: Write the interface** (`lib/features/breaking_news/domain/alert_notifier.dart`)

```dart
import 'breaking_news_story.dart';

/// Platform-facing surface for posting alerts. Android-only v1; iOS
/// resumption later implements this behind the same seam (spec: parked,
/// not dropped). Faked in every test — see FlutterLocalAlertNotifier for
/// the real implementation (the only file allowed to import
/// flutter_local_notifications, same isolation as WorkmanagerBackgroundScheduler).
abstract interface class AlertNotifier {
  /// Whether notifications can currently be posted (OS permission granted
  /// or pre-13 auto-grant). The background job checks this before spending
  /// any network.
  Future<bool> areNotificationsEnabled();

  /// Requests the OS permission. Returns true if granted. Non-Android
  /// platforms return false (feature not shipped there in v1).
  Future<bool> requestPermission();

  /// Posts the heads-up notification for [story].
  Future<void> show(BreakingNewsStory story);
}
```

- [ ] **Step 5: Write the shim** (`lib/features/breaking_news/data/flutter_local_alert_notifier.dart`)

```dart
import 'dart:io' show Platform;

import 'package:flutter_local_notifications/flutter_local_notifications.dart';

import '../domain/alert_notifier.dart';
import '../domain/breaking_news_story.dart';

/// The real [AlertNotifier], and the owner of notification tap routing.
/// Untested by design — a thin plugin shim, faked at [AlertNotifier]
/// everywhere else (same policy as WorkmanagerBackgroundScheduler).
///
/// Safe to construct anywhere; [ensureInitialized] is idempotent and must
/// run once per isolate before [show]/[tapPayloads]/[coldStartPayload] —
/// the foreground calls it from `app.dart`, the background job from its
/// own isolate (posting needs only the application context, no Activity).
class FlutterLocalAlertNotifier implements AlertNotifier {
  final FlutterLocalNotificationsPlugin _plugin = FlutterLocalNotificationsPlugin();
  bool _initialized = false;

  static const _channelId = 'breaking_news';
  static const _channelName = 'Breaking news';
  static const _channelDescription =
      'Alerts when 3 or more outlets cover the same fresh story in your '
      'followed topics. Checked about every 15 minutes — not real-time.';

  Future<void> ensureInitialized() async {
    if (_initialized) return;
    await _plugin.initialize(
      const InitializationSettings(
        android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      ),
    );
    await _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(const AndroidNotificationChannel(
          _channelId,
          _channelName,
          description: _channelDescription,
          importance: Importance.high,
        ));
    _initialized = true;
  }

  @override
  Future<bool> areNotificationsEnabled() async {
    if (!Platform.isAndroid) return true; // nothing will be posted anyway
    await ensureInitialized();
    return await _plugin
            .resolvePlatformSpecificImplementation<
                AndroidFlutterLocalNotificationsPlugin>()
            ?.areNotificationsEnabled() ??
        false;
  }

  @override
  Future<bool> requestPermission() async {
    if (!Platform.isAndroid) return false; // iOS parked — Settings stays honest
    await ensureInitialized();
    return await _plugin
            .resolvePlatformSpecificImplementation<
                AndroidFlutterLocalNotificationsPlugin>()
            ?.requestNotificationsPermission() ??
        false;
  }

  @override
  Future<void> show(BreakingNewsStory story) async {
    await ensureInitialized();
    // Distinct id per story so two stories in one wake don't replace
    // each other; hashCode keeps it deterministic across isolates.
    final id = story.link.hashCode & 0x7fffffff;
    await _plugin.show(
      id,
      story.headline,
      story.bodyText,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          _channelId,
          _channelName,
          channelDescription: _channelDescription,
          importance: Importance.high,
          priority: Priority.high,
        ),
      ),
      payload: story.link,
    );
  }

  /// Payload of the notification that cold-launched the app, if any.
  Future<String?> coldStartPayload() async {
    final details = await _plugin.getNotificationAppLaunchDetails();
    if (details == null || !details.didNotificationLaunchApp) return null;
    return details.notificationResponse?.payload;
  }

  /// Warm taps while the app is running (after [ensureInitialized]).
  Stream<String> get tapPayloads =>
      _plugin.onDidReceiveNotificationResponse
          .map((r) => r.payload)
          .whereType<String>();
}
```

- [ ] **Step 6: Verify the build**

Run: `flutter pub get && flutter analyze && flutter build apk --debug`
Expected: clean analyze; debug APK builds (this is what proves the gradle/desugaring change).
Then `flutter test` (full suite — nothing should have regressed).

- [ ] **Step 7: Commit**

```bash
git add pubspec.yaml pubspec.lock android/app/build.gradle.kts \
  android/app/src/main/AndroidManifest.xml \
  lib/features/breaking_news/domain/alert_notifier.dart \
  lib/features/breaking_news/data/flutter_local_alert_notifier.dart
git commit -m "feat(breaking-news): AlertNotifier interface + flutter_local_notifications shim

Plugin isolated to one file; desugaring enabled (plugin requirement);
POST_NOTIFICATIONS added (runtime-prompted only from the Settings toggle).

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 6: `BreakingNewsJob` — the background job

**Files:**
- Create: `lib/features/breaking_news/data/breaking_news_job.dart`
- Test: `test/breaking_news_job_test.dart`

**Interfaces:**
- Consumes: everything above — `BackgroundJob`/`BackgroundJobContext`, `CatalogCache`, `feedTargetsFor`, `ArticleRepositoryImpl`, `AlertLedger`, `BreakingNewsDetector`, `AlertNotifier`, storage keys from Task 1, `deviceLocaleCountryCode` (from `core/utils/device_locale.dart`, as `PrefetchJob` uses it).
- Produces: `class BreakingNewsJob implements BackgroundJob` with `static const jobId = 'breaking_news_check'`; constructor `BreakingNewsJob({required AlertNotifier notifier, BreakingNewsDetector detector = const BreakingNewsDetector(), String? Function() localeCountryCodeFallback = deviceLocaleCountryCode})`.

- [ ] **Step 1: Write the failing tests**

Create `test/breaking_news_job_test.dart`. The RSS-serving fake adapts `prefetch_job_test.dart`'s `_RecordingAdapter` — key difference: per-URL **titles and pubDates** are controllable, since detection depends on both:

```dart
import 'dart:async';
import 'dart:typed_data';

import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/background/background_job_context.dart';
import 'package:newsfeed/core/storage/local_storage_service.dart';
import 'package:newsfeed/features/breaking_news/data/alert_ledger.dart';
import 'package:newsfeed/features/breaking_news/data/breaking_news_job.dart';
import 'package:newsfeed/features/breaking_news/domain/alert_notifier.dart';
import 'package:newsfeed/features/breaking_news/domain/breaking_news_config.dart';
import 'package:newsfeed/features/breaking_news/domain/breaking_news_story.dart';
import 'package:newsfeed/core/network/http_client.dart';
import 'package:shared_preferences/shared_preferences.dart';

const _nowIso = 'Wed, 18 Sep 2026 11:30:00 GMT'; // 30 min before test `now`

class _FakeAlertNotifier implements AlertNotifier {
  _FakeAlertNotifier({this.enabled = true, this.throwOnShow = false});
  bool enabled;
  bool throwOnShow;
  final List<BreakingNewsStory> shown = [];

  @override
  Future<bool> areNotificationsEnabled() async => enabled;

  @override
  Future<bool> requestPermission() async => enabled;

  @override
  Future<void> show(BreakingNewsStory story) async {
    if (throwOnShow) throw StateError('notification machinery down');
    shown.add(story);
  }
}

/// Serves one RSS item per feed URL — title/pubDate from [items].
class _RssAdapter implements HttpClientAdapter {
  _RssAdapter({required this.items});
  final Map<String, ({String title, String pubDate})> items;
  final List<String> requestedUrls = [];

  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    final url = options.uri.toString();
    requestedUrls.add(url);
    final item = items[url] ??
        (title: 'Filler headline about ordinary news', pubDate: _nowIso);
    return ResponseBody.fromString(
      '''
      <rss version="2.0"><channel>
        <item>
          <title>${item.title}</title>
          <link>$url/article</link>
          <pubDate>${item.pubDate}</pubDate>
        </item>
      </channel></rss>
      ''',
      200,
      headers: {
        Headers.contentTypeHeader: [Headers.textPlainContentType],
      },
    );
  }

  @override
  void close({bool force = false}) {}
}

const _catalogJson = '''
[{"id":"world","label":"World","icon":"public_rounded",
  "gradient":["#4F6BFF","#7B5CFA"],"sources":[
    {"name":"SrcA","rssUrl":"https://a.example.com/feed","language":"en","type":"international"},
    {"name":"SrcB","rssUrl":"https://b.example.com/feed","language":"en","type":"international"},
    {"name":"SrcC","rssUrl":"https://c.example.com/feed","language":"en","type":"international"}
  ]}]
''';
const _regionsJson = '''
[{"countryCode":"IL","countryName":"Israel","flagEmoji":"🇮🇱","sources":[
    {"name":"IL","rssUrl":"https://il.example.com/feed","language":"he","type":"national"}]}]
''';

void main() {
  const breakingTitle = 'Central bank raises rates amid inflation';
  final now = DateTime.utc(2026, 9, 18, 12, 0);

  late LocalStorageService storage;
  late _FakeAlertNotifier notifier;

  setUp(() async {
    SharedPreferences.setMockInitialValues({});
    storage = LocalStorageService(await SharedPreferences.getInstance());
    await storage.setCachedInterestCategoriesJson(_catalogJson);
    await storage.setCachedLocalNewsRegionsJson(_regionsJson);
    await storage.setSelectedInterests(['world']);
    await storage.setLastResolvedCountryCode('IL');
    await storage.setBreakingNewsEnabled(true);
    notifier = _FakeAlertNotifier();
  });

  BreakingNewsJob job() => BreakingNewsJob(
        notifier: notifier,
        localeCountryCodeFallback: () => 'IL',
      );

  /// Same-quake story on the 3 catalog sources (three distinct hosts).
  Map<String, ({String title, String pubDate})> quakeItems() => {
        'https://a.example.com/feed': (
          title: breakingTitle,
          pubDate: 'Wed, 18 Sep 2026 11:50:00 GMT'
        ),
        'https://b.example.com/feed': (
          title: breakingTitle,
          pubDate: 'Wed, 18 Sep 2026 11:55:00 GMT'
        ),
        'https://c.example.com/feed': (
          title: breakingTitle,
          pubDate: 'Wed, 18 Sep 2026 11:58:00 GMT'
        ),
      };

  Future<bool> runWith(Map<String, ({String title, String pubDate})> items) async {
    final prefs = await SharedPreferences.getInstance();
    final dio = Dio(BaseOptions())
      ..httpClientAdapter = _RssAdapter(items: items);
    return job().run(BackgroundJobContext(storage: LocalStorageService(prefs), dio: dio));
  }

  test('disabled short-circuits: no fetch, no notification', () async {
    await storage.setBreakingNewsEnabled(false);
    await runWith(quakeItems());
    expect(notifier.shown, isEmpty);
  });

  test('OS permission revoked short-circuits before fetching', () async {
    notifier.enabled = false;
    await runWith(quakeItems());
    expect(notifier.shown, isEmpty);
    expect(storage.breakingNewsLastCheckAt, isNull); // never spent the run
  });

  test('happy path: one qualifying story is shown and recorded', () async {
    final ok = await runWith(quakeItems());
    expect(ok, isTrue);
    expect(notifier.shown, hasLength(1));
    expect(notifier.shown.single.link, contains('b.example.com')); // newest member
    final ledger = AlertLedger(storage);
    // whatever signature the detector derived, it must now be recorded:
    final prefs = storage;
    expect(prefs.breakingNewsAlertTimes, hasLength(1));
    expect(ledger, isNotNull);
    expect(storage.breakingNewsLastCheckAt, isNotNull);
  });

  test('same story again is suppressed by the ledger', () async {
    await runWith(quakeItems());
    notifier.shown.clear();
    await runWith(quakeItems());
    expect(notifier.shown, isEmpty);
  });

  test('rate gap blocks a second distinct story within 60 minutes', () async {
    // First run alerts on the quake.
    await runWith(quakeItems());
    notifier.shown.clear();
    // Second run, same wake-window, a *different* 3-outlet story: the
    // ledger's 60-min gap must block it.
    final second = quakeItems()
        .map((url, item) => MapEntry(
            url,
            (
              title: 'Football club fires head coach overnight',
              pubDate: item.pubDate
            )));
    await runWith(Map.from(second)..remove('https://a.example.com/feed') // avoid re-clustering with quake? no — 3 sources needed
    );
    // NOTE: build `second` as full 3-source set with the football title on
    // all three URLs; the ledger gap blocks the notification either way.
    expect(notifier.shown, isEmpty);
  });

  test('show() failure returns false and records nothing', () async {
    notifier.throwOnShow = true;
    final ok = await runWith(quakeItems());
    expect(ok, isFalse);
    expect(storage.breakingNewsAlertTimes, isEmpty);
    final ledger = AlertLedger(storage);
    // nothing recorded → a retry can re-alert:
    expect(
      ledger.wasAlerted(
        'x', now: DateTime.utc(2026, 9, 18, 12, 1), ttl: const Duration(hours: 24),
      ),
      isFalse,
    );
  });

  test('all sources failing returns false', () async {
    // every feed URL missing from `items` still returns filler — so make a
    // job run where the adapter itself is unreachable instead:
    final prefs = await SharedPreferences.getInstance();
    final dio = Dio(BaseOptions())..httpClientAdapter = _FailingAdapter();
    final ok = await job().run(
        BackgroundJobContext(storage: LocalStorageService(prefs), dio: dio));
    expect(ok, isFalse);
  });

  test('writes only breaking_news_* keys', () async {
    final prefs = await SharedPreferences.getInstance();
    final before = prefs.getKeys().toSet();
    final dio = Dio(BaseOptions())
      ..httpClientAdapter = _RssAdapter(items: quakeItems());
    await job().run(
        BackgroundJobContext(storage: LocalStorageService(prefs), dio: dio));
    final newKeys = prefs.getKeys().difference(before);
    expect(newKeys, everyElement(startsWith('breaking_news_')));
  });
}

class _FailingAdapter implements HttpClientAdapter {
  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    throw DioException(
      requestOptions: options,
      type: DioExceptionType.connectionError,
    );
  }

  @override
  void close({bool force = false}) {}
}
```

Clean up the two sloppy bits flagged in comments above before committing: the rate-gap test should set the football title on **all three** feed URLs (a fresh 3-outlet cluster), and drop the stray `Map.from(second)..remove(...)` fragment — assert only `expect(notifier.shown, isEmpty)` after that run. Also delete the unused `final prefs = storage;` / `expect(ledger, isNotNull);` lines in the happy-path test (leftovers), keeping the real assertions. The intent of each test is unchanged.

Also note: the ledger/time writes use **real** `DateTime.now()`, not the test's `now` const — that's fine here: `recordNotification`/`record` use the job's wall clock, and the suppression test runs immediately after (gap still in force, TTL far away).

- [ ] **Step 2: Run to verify failure**

Run: `flutter test test/breaking_news_job_test.dart`
Expected: FAIL — `breaking_news_job.dart` doesn't exist.

- [ ] **Step 3: Implement**

Create `lib/features/breaking_news/data/breaking_news_job.dart`:

```dart
import '../../../core/background/background_job.dart';
import '../../../core/background/background_job_context.dart';
import '../../../core/catalog/catalog_cache.dart';
import '../../../core/utils/device_locale.dart';
import '../../feed/data/datasources/rss_remote_data_source.dart';
import '../../feed/data/repositories/article_repository_impl.dart';
import '../../feed/domain/feed_targets.dart';
import '../domain/alert_notifier.dart';
import '../domain/breaking_news_config.dart';
import '../domain/breaking_news_detector.dart';
import 'alert_ledger.dart';

/// Periodic breaking-news check (roadmap #7, spec: "Job run, step by
/// step"). Fetches all followed targets itself (Approach A — independent
/// of PrefetchJob's toggle and schedule), detects qualifying multi-source
/// stories, and posts at most [BreakingNewsConfig.maxAlertsPerRun]
/// notifications per wake, bounded by the ledger's gap/cap windows.
///
/// **Background-isolate write rule** (see PrefetchJob): writes ONLY
/// `breaking_news_*` keys; reads `read_article_links` and mute/custom
/// keys. Never touches read-events, bookmarks, or the article cache.
class BreakingNewsJob implements BackgroundJob {
  const BreakingNewsJob({
    required this.notifier,
    this.detector = const BreakingNewsDetector(),
    this.localeCountryCodeFallback = deviceLocaleCountryCode,
  });

  static const jobId = 'breaking_news_check';

  final AlertNotifier notifier;
  final BreakingNewsDetector detector;

  /// Same locale fallback PrefetchJob uses, so both jobs resolve the same
  /// effective region (see PrefetchJob.localeCountryCodeFallback).
  final String? Function() localeCountryCodeFallback;

  @override
  String get id => jobId;

  @override
  Future<bool> run(BackgroundJobContext ctx) async {
    // 1. Feature toggle off — nothing to do (success, not failure).
    if (!ctx.storage.breakingNewsEnabled) return true;

    // 2. Permission revoked since last run — skip the whole fetch (spec:
    //    saving the network/battery is the point). Alerts resume on
    //    re-grant with no re-scheduling needed (the periodic task was
    //    never cancelled).
    try {
      if (!await notifier.areNotificationsEnabled()) return true;
    } catch (_) {
      return true; // notifier misbehaving — don't retry-loop the job over it
    }

    // 3-4. Resolve + fetch targets exactly like PrefetchJob.
    final catalog = CatalogCache(ctx.storage);
    final targets = feedTargetsFor(
      categories: catalog.categories(),
      followedCategoryIds: ctx.storage.selectedInterests,
      regions: catalog.regions(),
      countryCode: ctx.storage.lastResolvedCountryCode ?? localeCountryCodeFallback(),
      customSources: ctx.storage.customSources
          .map((m) => NewsSource(
                name: m['name'] as String,
                rssUrl: m['rssUrl'] as String,
              ))
          .toList(),
    );
    if (targets.isEmpty) {
      await _stampLastCheck(ctx);
      return true;
    }

    final repository =
        ArticleRepositoryImpl(RssRemoteDataSource(ctx.dio), ctx.storage);
    final muted = ctx.storage.mutedSourceUrls;
    final results = await Future.wait(
      targets.map((target) async {
        try {
          return await repository.fetchArticlesForSources(
            target.cacheKey,
            target.sources,
            forceRefresh: true,
            excludedSourceUrls: muted,
          );
        } catch (_) {
          return const []; // one dead target can't sink the run
        }
      }),
    );
    final articles = repository.mergeAndSort(results.expand((a) => a).toList());

    // 5. Detect.
    final config = const BreakingNewsConfig();
    final ledger = AlertLedger(ctx.storage);
    final stories = detector.detect(
      articles: articles,
      readLinks: ctx.storage.readLinks,
      wasAlerted: (signature) =>
          ledger.wasAlerted(signature, now: DateTime.now().toUtc(), ttl: config.ledgerTtl),
      now: DateTime.now().toUtc(),
      config: config,
    );

    // 6. Notify, bounded by gap + rolling cap; record only after a
    //    successful show() so a failure returns false → retry re-alerts.
    for (final story in stories) {
      final canNotify = ledger.canNotifyNow(
        now: DateTime.now().toUtc(),
        minGap: config.minGapBetweenAlerts,
        window: config.rollingWindow,
        windowCap: config.rollingWindowCap,
      );
      if (!canNotify) break;
      try {
        await notifier.show(story);
      } catch (_) {
        return false; // WorkManager backoff; story stays unrecorded
      }
      final nowUtc = DateTime.now().toUtc();
      await ledger.recordNotification(now: nowUtc, window: config.rollingWindow);
      await ledger.record(
        story.signature,
        now: nowUtc,
        ttl: config.ledgerTtl,
        cap: config.ledgerCap,
      );
    }

    // 7. Stamp the status row; done.
    await _stampLastCheck(ctx);
    return true;
  }

  Future<void> _stampLastCheck(BackgroundJobContext ctx) =>
      ctx.storage.setBreakingNewsLastCheck(DateTime.now());
}
```

Fix before committing: `const BreakingNewsConfig()` — drop the stray `const`-in-`final` (`final config = const BreakingNewsConfig();` is fine as written; keep as is). Ensure `NewsSource` is imported via `feed_targets.dart`'s own imports (`interest_categories.dart`) — add `import '../../../core/constants/interest_categories.dart';` if the analyzer flags `NewsSource`.

- [ ] **Step 4: Run to verify pass**

Run: `flutter test test/breaking_news_job_test.dart`
Expected: PASS.

- [ ] **Step 5: Full suite + analyze, then commit**

Run: `flutter analyze && flutter test`

```bash
git add lib/features/breaking_news/data/breaking_news_job.dart test/breaking_news_job_test.dart
git commit -m "feat(breaking-news): background job — fetch, detect, notify, record

Own fetch via the shared feedTargetsFor pipeline; writes confined to
breaking_news_* keys; ledger recorded only after a successful show().

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 7: Registration — `background_main.dart` + cold-start re-assertion

**Files:**
- Modify: `lib/background_main.dart` (registry list)
- Modify: `lib/main.dart` (sibling re-assertion clause, after the prefetch one)

**Interfaces:**
- Consumes: `BreakingNewsJob.jobId` (Task 6), `FlutterLocalAlertNotifier` (Task 5).

- [ ] **Step 1: Register the job** (`lib/background_main.dart`)

```dart
import 'features/breaking_news/data/breaking_news_job.dart';
import 'features/breaking_news/data/flutter_local_alert_notifier.dart';

final _registry = BackgroundJobRegistry([
  PrefetchJob(imagePrefetcher: CacheManagerImagePrefetcher()),
  BreakingNewsJob(notifier: FlutterLocalAlertNotifier()),
]);
```

Update the `_registry` doc comment's "#7 adds its job here" sentence to past tense ("#7's BreakingNewsJob is registered here alongside PrefetchJob").

- [ ] **Step 2: Cold-start re-assertion** (`lib/main.dart`)

After the existing prefetch `if` block (same style — `unawaited`, `.catchError`, Android-guarded; note **no `hasOnboarded` gate is strictly needed** since the toggle can only be switched on post-onboarding, but keep `hasOnboarded` for symmetry with the prefetch clause):

```dart
  if (Platform.isAndroid && storage.hasOnboarded && storage.breakingNewsEnabled) {
    unawaited(
      WorkmanagerBackgroundScheduler(callbackDispatcher)
          .schedulePeriodic(
            jobId: BreakingNewsJob.jobId,
            interval: const Duration(minutes: 15),
            // Alerts are time-sensitive by definition — no Wi-Fi-only
            // constraint (spec: no Wi-Fi-only switch in v1).
            unmeteredOnly: false,
          )
          .catchError((_) {}),
    );
  }
```

with `import 'features/breaking_news/data/breaking_news_job.dart';` added.

- [ ] **Step 3: Verify**

Run: `flutter analyze && flutter test`
Expected: clean (no unit test targets `main.dart` — the e2e wake is Task 10's on-device verification, same as #5's close-out).

- [ ] **Step 4: Commit**

```bash
git add lib/background_main.dart lib/main.dart
git commit -m "feat(breaking-news): register job + cold-start schedule re-assertion

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 8: Settings — providers + tile

**Files:**
- Create: `lib/features/breaking_news/presentation/providers/breaking_news_providers.dart`
- Create: `lib/features/breaking_news/presentation/widgets/breaking_news_tile.dart`
- Modify: `lib/features/settings/presentation/screens/settings_screen.dart` (place after `const OfflinePrefetchTile(),` at `settings_screen.dart:100`)
- Test: `test/breaking_news_tile_test.dart`

**Interfaces:**
- Consumes: `AlertNotifier`, `BreakingNewsJob.jobId`, storage keys (Task 1), `backgroundSchedulerProvider` (existing, from offline_prefetch_providers.dart), `localStorageServiceProvider`.
- Produces:
  ```dart
  final alertNotifierProvider = Provider<AlertNotifier>((ref) => FlutterLocalAlertNotifier());
  final breakingNewsEnabledProvider = NotifierProvider<BreakingNewsEnabledNotifier, bool>(...);
  class BreakingNewsEnabledNotifier extends Notifier<bool> {
    Future<bool> setEnabled(bool value); // false = OS permission denied
  }
  final osNotificationsAllowedProvider = FutureProvider<bool>(...);
  ```

- [ ] **Step 1: Write the failing widget tests**

Create `test/breaking_news_tile_test.dart` (model on `test/offline_prefetch_tile_test.dart`'s `ProviderScope(overrides:)` style):

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/background/background_scheduler.dart';
import 'package:newsfeed/core/providers/core_providers.dart';
import 'package:newsfeed/core/storage/local_storage_service.dart';
import 'package:newsfeed/features/breaking_news/domain/alert_notifier.dart';
import 'package:newsfeed/features/breaking_news/domain/breaking_news_story.dart';
import 'package:newsfeed/features/breaking_news/presentation/providers/breaking_news_providers.dart';
import 'package:newsfeed/features/breaking_news/presentation/widgets/breaking_news_tile.dart';
import 'package:shared_preferences/shared_preferences.dart';

class _FakeAlertNotifier implements AlertNotifier {
  _FakeAlertNotifier({this.granted = true, this.osEnabled = true});
  bool granted;
  bool osEnabled;
  int permissionRequests = 0;
  final List<BreakingNewsStory> shown = [];

  @override
  Future<bool> areNotificationsEnabled() async => osEnabled;

  @override
  Future<bool> requestPermission() async {
    permissionRequests++;
    return granted;
  }

  @override
  Future<void> show(BreakingNewsStory story) async => shown.add(story);
}

class _RecordingScheduler implements BackgroundScheduler {
  final scheduled = <String>[];
  final cancelled = <String>[];

  @override
  Future<void> schedulePeriodic({
    required String jobId,
    required Duration interval,
    required bool unmeteredOnly,
  }) async =>
      scheduled.add(jobId);

  @override
  Future<void> cancel(String jobId) async => cancelled.add(jobId);
}

Future<LocalStorageService> _storage() async {
  SharedPreferences.setMockInitialValues({});
  return LocalStorageService(await SharedPreferences.getInstance());
}

Widget _wrap(List<Override> overrides) => ProviderScope(
      overrides: overrides,
      child: const MaterialApp(home: Scaffold(body: BreakingNewsTile())),
    );

void main() {
  testWidgets('toggle on + permission granted: schedules + persists',
      (tester) async {
    final storage = await _storage();
    final notifier = _FakeAlertNotifier();
    final scheduler = _RecordingScheduler();
    await tester.pumpWidget(_wrap([
      localStorageServiceProvider.overrideWithValue(storage),
      alertNotifierProvider.overrideWithValue(notifier),
      backgroundSchedulerProvider.overrideWithValue(scheduler),
      osNotificationsAllowedProvider.overrideWith((ref) async => true),
    ]));

    await tester.tap(find.byType(Switch).first);
    await tester.pumpAndSettle();

    expect(notifier.permissionRequests, 1);
    expect(scheduler.scheduled, ['breaking_news_check']);
    expect(storage.breakingNewsEnabled, isTrue);
    final sw = tester.widget<Switch>(find.byType(Switch).first);
    expect(sw.value, isTrue);
  });

  testWidgets('toggle on + permission denied: stays off with hint',
      (tester) async {
    final storage = await _storage();
    final notifier = _FakeAlertNotifier(granted: false);
    final scheduler = _RecordingScheduler();
    await tester.pumpWidget(_wrap([
      localStorageServiceProvider.overrideWithValue(storage),
      alertNotifierProvider.overrideWithValue(notifier),
      backgroundSchedulerProvider.overrideWithValue(scheduler),
      osNotificationsAllowedProvider.overrideWith((ref) async => false),
    ]));

    await tester.tap(find.byType(Switch).first);
    await tester.pumpAndSettle();

    expect(notifier.permissionRequests, 1);
    expect(scheduler.scheduled, isEmpty);
    expect(storage.breakingNewsEnabled, isFalse);
    expect(find.text('Notification permission was denied'), findsOneWidget);
    final sw = tester.widget<Switch>(find.byType(Switch).first);
    expect(sw.value, isFalse);
  });

  testWidgets('toggle off: cancels + persists', (tester) async {
    final storage = await _storage();
    await storage.setBreakingNewsEnabled(true);
    final notifier = _FakeAlertNotifier();
    final scheduler = _RecordingScheduler();
    await tester.pumpWidget(_wrap([
      localStorageServiceProvider.overrideWithValue(storage),
      alertNotifierProvider.overrideWithValue(notifier),
      backgroundSchedulerProvider.overrideWithValue(scheduler),
      osNotificationsAllowedProvider.overrideWith((ref) async => true),
    ]));

    await tester.tap(find.byType(Switch).first);
    await tester.pumpAndSettle();

    expect(scheduler.cancelled, ['breaking_news_check']);
    expect(storage.breakingNewsEnabled, isFalse);
  });

  testWidgets('enabled + OS permission revoked: system-settings hint shown',
      (tester) async {
    final storage = await _storage();
    await storage.setBreakingNewsEnabled(true);
    final notifier = _FakeAlertNotifier(osEnabled: false);
    final scheduler = _RecordingScheduler();
    await tester.pumpWidget(_wrap([
      localStorageServiceProvider.overrideWithValue(storage),
      alertNotifierProvider.overrideWithValue(notifier),
      backgroundSchedulerProvider.overrideWithValue(scheduler),
      osNotificationsAllowedProvider.overrideWith((ref) async => false),
    ]));
    await tester.pumpAndSettle();

    expect(find.text('Notifications are off in system settings'),
        findsOneWidget);
  });

  testWidgets('enabled + allowed + stamped: Last checked row renders',
      (tester) async {
    final storage = await _storage();
    await storage.setBreakingNewsEnabled(true);
    await storage.setBreakingNewsLastCheck(DateTime.utc(2026, 9, 18, 11, 0));
    final notifier = _FakeAlertNotifier();
    final scheduler = _RecordingScheduler();
    await tester.pumpWidget(_wrap([
      localStorageServiceProvider.overrideWithValue(storage),
      alertNotifierProvider.overrideWithValue(notifier),
      backgroundSchedulerProvider.overrideWithValue(scheduler),
      osNotificationsAllowedProvider.overrideWith((ref) async => true),
    ]));
    await tester.pumpAndSettle();

    expect(find.textContaining('Last checked'), findsOneWidget);
  });
}
```

Notes for whoever writes this file: seed persisted state through the `LocalStorageService` accessors after construction (as above), never by pre-populating `SharedPreferences.setMockInitialValues` with raw keys — that avoids the plugin's `flutter.` key-prefix question entirely and matches how `local_storage_service_test.dart` seeds. The literal `'breaking_news_check'` is intentional — tests should assert the real workmanager task name, not re-export it from the job class.

1. **toggle on + granted** → permission requested once, `scheduler.scheduled == ['breaking_news_check']`, `storage.breakingNewsEnabled == true`, switch renders on. (Use the literal `'breaking_news_check'` in tests, not a sentinel import — keeps the test honest about the workmanager task name.)
2. **toggle on + permission denied** → switch stays off, `storage.breakingNewsEnabled == false`, `scheduler.scheduled` empty, and the inline hint text ("Notifications were blocked...") is visible.
3. **toggle off** (starting enabled: seed `SharedPreferences.setMockInitialValues({'breaking_news_enabled': true})`) → `scheduler.cancelled == ['breaking_news_check']`, storage flag false.
4. **enabled + OS permission later revoked** (override `osNotificationsAllowedProvider` → false) → status row replaced by the "Notifications are off in system settings" hint.
5. **enabled + OS allowed + last check stamped** (seed `breaking_news_last_check_at`) → "Last checked" text renders.

(`backgroundSchedulerProvider` lives in `offline_prefetch_providers.dart` — import from there.)

- [ ] **Step 2: Run to verify failure**

Run: `flutter test test/breaking_news_tile_test.dart`
Expected: FAIL — providers/tile don't exist.

- [ ] **Step 3: Implement providers** (`breaking_news_providers.dart`)

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../../../core/providers/core_providers.dart';
import '../../data/breaking_news_job.dart';
import '../../data/flutter_local_alert_notifier.dart';
import '../../domain/alert_notifier.dart';
import '../../../../features/offline_prefetch/presentation/providers/offline_prefetch_providers.dart'
    show backgroundSchedulerProvider;

const _kBreakingNewsInterval = Duration(minutes: 15); // workmanager's floor

final alertNotifierProvider =
    Provider<AlertNotifier>((ref) => FlutterLocalAlertNotifier());

/// Live OS-permission state for the tile's revoked-permission hint.
/// Re-evaluated whenever the tile is (re)built after a toggle change.
final osNotificationsAllowedProvider = FutureProvider<bool>((ref) async {
  return ref.watch(alertNotifierProvider).areNotificationsEnabled();
});

/// Owns the "Breaking news alerts" toggle. Returns false when the OS
/// permission was denied — a user outcome, not an error, so the tile
/// keeps the switch off and shows the hint instead of a SnackBar.
///
/// Success-before-state ordering as OfflinePrefetchEnabledNotifier: a
/// failed schedule/cancel must never leave storage claiming the new state.
class BreakingNewsEnabledNotifier extends Notifier<bool> {
  @override
  bool build() => ref.watch(localStorageServiceProvider).breakingNewsEnabled;

  Future<bool> setEnabled(bool value) async {
    final notifier = ref.read(alertNotifierProvider);
    if (value && !await notifier.requestPermission()) return false;
    final scheduler = ref.read(backgroundSchedulerProvider);
    if (value) {
      await scheduler.schedulePeriodic(
        jobId: BreakingNewsJob.jobId,
        interval: _kBreakingNewsInterval,
        unmeteredOnly: false, // timeliness is the point (spec)
      );
    } else {
      await scheduler.cancel(BreakingNewsJob.jobId);
    }
    state = value;
    await ref
        .read(localStorageServiceProvider)
        .setBreakingNewsEnabled(value);
    // Refresh the revoked-permission hint after any toggle interaction.
    ref.invalidate(osNotificationsAllowedProvider);
    return true;
  }
}

final breakingNewsEnabledProvider =
    NotifierProvider<BreakingNewsEnabledNotifier, bool>(
  BreakingNewsEnabledNotifier.new,
);
```

- [ ] **Step 4: Implement the tile** (`breaking_news_tile.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../../../core/providers/core_providers.dart';
import '../../../../core/utils/date_format_utils.dart';
import '../providers/breaking_news_providers.dart';

/// Settings block for breaking-news alerts: the opt-in toggle (which is
/// the only place the OS POST_NOTIFICATIONS prompt is ever triggered),
/// the honest-cadence subtitle, and a status/hint row ("Last checked" —
/// or the system-settings hint when the OS permission was revoked since).
class BreakingNewsTile extends ConsumerStatefulWidget {
  const BreakingNewsTile({super.key});

  @override
  ConsumerState<BreakingNewsTile> createState() => _BreakingNewsTileState();
}

class _BreakingNewsTileState extends ConsumerState<BreakingNewsTile> {
  bool _permissionDeniedHint = false;

  Future<void> _setEnabled(bool value) async {
    try {
      final granted = await ref
          .read(breakingNewsEnabledProvider.notifier)
          .setEnabled(value);
      if (!mounted) return;
      setState(() => _permissionDeniedHint = value && !granted);
    } catch (_) {
      if (!mounted) return;
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
            content: Text("Couldn't update breaking-news alerts — try again")),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final enabled = ref.watch(breakingNewsEnabledProvider);
    final storage = ref.read(localStorageServiceProvider);
    final osAllowed = ref.watch(osNotificationsAllowedProvider);

    return Column(
      children: [
        SwitchListTile(
          contentPadding:
              const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          secondary: const Icon(Icons.notifications_active_outlined),
          title: const Text('Breaking news alerts'),
          subtitle: const Text(
            'Checks your followed topics every ~15 min and tells you when '
            '3+ outlets break the same story. Not real-time.',
          ),
          value: enabled,
          onChanged: _setEnabled,
        ),
        if (enabled)
          osAllowed.when(
            data: (allowed) {
              if (!allowed) {
                return const ListTile(
                  contentPadding:
                      EdgeInsets.symmetric(horizontal: 16, vertical: 4),
                  leading: Icon(Icons.warning_amber_rounded),
                  title: Text('Notifications are off in system settings'),
                  subtitle: Text(
                      'Enable them there to start receiving breaking-news alerts'),
                );
              }
              final lastCheck = storage.breakingNewsLastCheckAt;
              return ListTile(
                contentPadding:
                    const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
                leading: const Icon(Icons.schedule),
                title: Text(lastCheck == null
                    ? 'Waiting for the first check'
                    : 'Last checked ${formatRelativeTime(lastCheck)}'),
              );
            },
            loading: () => const SizedBox.shrink(),
            error: (_, _) => const SizedBox.shrink(),
          ),
        if (_permissionDeniedHint && !enabled)
          const ListTile(
            contentPadding: EdgeInsets.symmetric(horizontal: 16, vertical: 4),
            leading: Icon(Icons.block_outlined),
            title: Text('Notification permission was denied'),
            subtitle: Text(
                'Turn the switch on again and allow notifications to enable alerts'),
          ),
      ],
    );
  }
}
```

Check `date_format_utils.dart` for the actual relative-time helper name; if none exists, format with `intl` (`DateFormat.MMMEd().add_jm()` — "Fri, Sep 18 · 2:15 PM") and adjust the code above. Also `error: (_, _) =>` must match the repo's Dart SDK wildcard style (`(_, _)` needs Dart 3.7+; use `(_, __)` if the repo is older — match existing code).

- [ ] **Step 5: Place it in Settings** (`settings_screen.dart`)

After `const OfflinePrefetchTile(),` (line 100) add:

```dart
                  const BreakingNewsTile(),
```

with the import. (Same section — both are notification/data-surface settings.)

- [ ] **Step 6: Run to verify pass**

Run: `flutter test test/breaking_news_tile_test.dart test/screens_smoke_test.dart`
Expected: PASS (screens smoke test exercises Settings; the tile must render there without plugin channels — that's why the tile only touches `AlertNotifier` via the overridable provider).

If `screens_smoke_test.dart` fails on the plugin channel (production `alertNotifierProvider` constructs `FlutterLocalAlertNotifier`, and `osNotificationsAllowedProvider` hits the channel): add the missing-channel mocks to that test's `setUp` (`TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger.setMockMethodCallHandler` on channel `dexterous.com/flutter/local_notifications`, returning `true` for `initialize`/`areNotificationsEnabled`/`requestNotificationsPermission` and `{}`/null for `getNotificationAppLaunchDetails`/`createNotificationChannel`) — do NOT change the tile to avoid the provider.

- [ ] **Step 7: Full suite, then commit**

```bash
git add lib/features/breaking_news/presentation/ \
  lib/features/settings/presentation/screens/settings_screen.dart \
  test/breaking_news_tile_test.dart
git commit -m "feat(breaking-news): settings tile — opt-in toggle, honest copy, status row

OS permission requested only from the toggle tap; denied keeps the
switch off with an inline hint; revoked-permission state surfaced.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 9: Tap routing in `app.dart`

**Files:**
- Modify: `lib/app.dart` (extract `_openArticleLink`, add notification-tap consumption)
- Test: `test/app_test.dart` (extend mocks)

**Interfaces:**
- Consumes: `FlutterLocalAlertNotifier.ensureInitialized/coldStartPayload/tapPayloads` (Task 5); the article-open flow in `_openFromWidgetTap` (`app.dart`).

- [ ] **Step 1: Refactor + wire**

In `lib/app.dart`:

1. Extract the body of `_openFromWidgetTap` after link extraction into:

```dart
  Future<void> _openArticleLink(String link) async {
    final cached = ref
        .read(allArticlesDiskCacheProvider)
        .where((a) => a.link == link)
        .firstOrNull;
    if (cached != null) {
      widget.router.push('/article', extra: cached);
      return;
    }

    // No longer in the disk cache — fall back to opening the link
    // directly, same http(s)-only guard as everywhere else a feed-derived
    // link is opened.
    final externalUri = safeArticleUri(link);
    if (externalUri != null) {
      launchUrl(externalUri, mode: LaunchMode.externalApplication);
    }
  }
```

`_openFromWidgetTap` becomes: extract link via `homeWidgetServiceProvider`, then `await _openArticleLink(link)`.

2. Add a `FlutterLocalAlertNotifier` field + lifecycle wiring in `initState` (mirroring the widget-tap wiring — deferred past first frame; the warm subscription needs `ensureInitialized()` first, so chain it):

```dart
  final _notificationNotifier = FlutterLocalAlertNotifier();
  StreamSubscription<String>? _notificationTapSubscription;

  // in initState, inside the existing addPostFrameCallback:
  Future<void>(() async {
    await _notificationNotifier.ensureInitialized();
    final coldPayload = await _notificationNotifier.coldStartPayload();
    if (coldPayload != null) await _openArticleLink(coldPayload);
  })();
  _notificationTapSubscription ??= _notificationNotifier.tapPayloads.listen(
    (payload) => _openArticleLink(payload),
  );
```

cancel `_notificationTapSubscription` in `dispose` (the existing pattern).

- [ ] **Step 2: Extend `test/app_test.dart` mocks**

Add to the test's channel-mock `setUp` (channel `dexterous.com/flutter/local_notifications`):

```dart
TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
    .setMockMethodCallHandler(
  const MethodChannel('dexterous.com/flutter/local_notifications'),
  (call) async {
    switch (call.method) {
      case 'initialize':
        return true;
      case 'getNotificationAppLaunchDetails':
        return {
          'notificationResponse': null,
          'didNotificationLaunchApp': false,
        };
      case 'areNotificationsEnabled':
      case 'requestNotificationsPermission':
        return true;
      default:
        return null;
    }
  },
);
```

(If the existing `app_test.dart` has no widget-tap mock section, add this handler alongside whatever HomeWidget mocks exist.) Add one test if the harness makes it cheap: cold-start launch details returning `didNotificationLaunchApp: true` + `payload: <link of a cached article>` results in `/article` being pushed; otherwise note the path as covered by Task 10's on-device check.

- [ ] **Step 3: Verify**

Run: `flutter analyze && flutter test test/app_test.dart`
Expected: clean.

- [ ] **Step 4: Commit**

```bash
git add lib/app.dart test/app_test.dart
git commit -m "feat(breaking-news): notification tap opens the linked article

Same cached-article / safe-http(s)-fallback flow as widget taps; cold
and warm tap paths both handled.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 10: Docs + roadmap + full verification + on-device checklist

**Files:**
- Modify: `CLAUDE.md` (Main features: one bullet under Reading features/adjacent)
- Modify: `FEATURE_ROADMAP.md` (status section: #7 shipped-Android pending verification state → update honestly per what Task 10 verified)
- Modify: `README.md` (features list, one line)

- [ ] **Step 1: Full gate**

Run: `flutter analyze && flutter test && flutter build apk --release --obfuscate --split-debug-info=build/symbols`
(the release flags mirror `.github/workflows/release-android.yml`'s build — the build must survive the same obfuscation the CI ships).

- [ ] **Step 2: CLAUDE.md bullet** (concise, matching the file's voice):

```markdown
- **Breaking-news alerts (Android, opt-in)**: Settings toggle; a 15-min
  background job clusters fresh headlines across followed feeds and
  notifies when 3+ distinct outlets cover the same story (<2h old).
  No push backend — client-side detection only. See
  `docs/superpowers/specs/2026-09-18-breaking-news-alerts-design.md`.
```

- [ ] **Step 3: README one-liner** in the features list: "Opt-in breaking-news alerts — on-device clustering across 3+ outlets, no server involved."

- [ ] **Step 4: On-device verification checklist** (mirrors #5's close-out; run on a physical device or emulator, `--release` + obfuscated):

1. `flutter run --release --obfuscate --split-debug-info=build/symbols` (device connected).
2. Settings → toggle "Breaking news alerts" on → OS permission prompt appears (Android 13+); allow.
3. Confirm WorkManager registered: `adb shell dumpsys jobscheduler | grep -A3 dk.newsfeed` shows a 15-min periodic job. (Note from #5's verification: `cmd jobscheduler run -f` is rejected before the period elapses — don't rely on forcing a fire; wait out the natural schedule, or temporarily lower the interval in a local debug build if iteration speed matters, reverting before commit.)
4. On the next natural wake with 3+ same-story outlets in followed feeds: heads-up notification appears.
5. Tap the notification **warm** (app backgrounded): article screen opens.
6. `adb shell am force-stop com.dk.newsfeed`, wait for another wake/notification, tap **cold**: article screen opens via launch-details payload.
7. Toggle off → job gone from `dumpsys jobscheduler`; re-enable → back.
8. Revoke notifications in system settings → next wake does no fetch (Settings tile shows the "off in system settings" hint on next app open).

- [ ] **Step 5: Roadmap status update** (`FEATURE_ROADMAP.md`)

In the "Status" section and the "What's next" list, mark #7 per the actual outcome (shipped Android / verification done or pending), note the iOS-parked status, and strike item 2 in "What's next" following the file's existing strikethrough convention. Mention the `breaking_news_check` job sharing `core/background/` with the prefetch job.

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md README.md FEATURE_ROADMAP.md
git commit -m "docs: breaking-news alerts shipped (roadmap #7) — status + guides

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Self-Review Notes

- **Spec coverage:** every spec section maps to a task — Detection heuristic → Task 3/4; Dedup & rate limits/state ownership → Task 1/2/6; Job run steps 1-7 → Task 6; Notification & platform surface → Task 5/9; Settings & permission UX → Task 8; Error handling → Task 6 (tests) + Task 8 (snackbar); Testing → each task's TDD cycle; Registration/re-assertion → Task 7; iOS-parked → interface-only seams (Task 5); obfuscation → Task 10 gate. Spec's "Manual/device verification" → Task 10 Step 4.
- **Type consistency:** `AlertNotifier` methods identical across Tasks 5/6/8; `BreakingNewsJob.jobId = 'breaking_news_check'` used verbatim in Tasks 6/7/8; storage accessor names consistent Task 1 → 2 → 6 → 8; `detect()` signature consistent Task 3/4 → 6.
- **Known thin spots (accepted, recorded):** no unit test for the plugin shim (policy: interface-faked everywhere, like `WorkmanagerBackgroundScheduler`); `main.dart`/`background_main.dart` wiring has no unit test — covered by Task 10's on-device pass; detector's `wasAlerted` callback deviates from the spec's ledger-object signature (recorded above).
