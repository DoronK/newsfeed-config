# Remote Force-Update Mechanism Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a device be blocked with a full-screen "update required" wall when its installed version falls below a minimum that can be raised remotely (via the same GitHub-Pages-hosted static JSON already used for the interest-category/local-news catalogs), with no app release required to raise the bar.

**Architecture:** A new remote-config value (`minSupportedVersion`) is fetched/cached the same instant-fallback + background-refresh way as the existing two catalogs. A pure comparator checks it against the installed version (`package_info_plus`). A single derived boolean provider (`updateRequiredProvider`) gates `app.dart`'s top-level build: when true, it renders a standalone, non-dismissible `UpdateRequiredScreen` instead of the normal routed app.

**Tech Stack:** Flutter, Riverpod (`Notifier`/`NotifierProvider`, `FutureProvider`, plain `Provider`), `dio` (already used), `package_info_plus` (new), `url_launcher` (already used), `flutter_test` + `shared_preferences` mock-values (existing test patterns).

**Spec:** `docs/superpowers/specs/2026-08-31-remote-force-update-design.md`

## Global Constraints

- **Fail open everywhere:** no confirmed minimum version (never fetched, fetch failed this session, malformed JSON) → never block. A minimum confirmed by a *previous* successful fetch stays enforced even if today's refresh fails. Any unparseable version string on either side of the comparison also fails open (no block), never throws.
- **Single threshold:** `minSupportedVersion` only. No second "recommended update" nudge tier.
- **Hard block, not dismissible:** below the threshold, a full-screen screen replaces the entire app UI — no back button, no way to reach any content.
- **No periodic re-check** while the app is already running — checked once per app process, same as the existing two catalog providers.
- **Update target is a hardcoded constant:** `https://play.google.com/store/apps/details?id=com.dk.newsfeed` — not itself remote-configurable.
- Config file lives at `docs/config/app_version.json`, generated from a Dart constant by `tool/export_remote_config_test.dart`, served from `https://doronk.github.io/newsfeed-config/config/app_version.json` — same generate → subtree-push workflow as the other two catalog files.

---

### Task 1: Dotted-version comparator

**Files:**
- Create: `lib/core/utils/semver.dart`
- Test: `test/semver_test.dart`

**Interfaces:**
- Produces: `int? compareDottedVersions(String a, String b)` — same contract as `Comparable.compareTo` (negative if `a < b`, zero if equal, positive if `a > b`), but returns `null` (never throws) if either string isn't dot-separated non-negative integers. A missing trailing segment counts as `0`, so `"1.2"` and `"1.2.0"` compare equal.

- [ ] **Step 1: Write the failing tests**

```dart
// test/semver_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/utils/semver.dart';

void main() {
  group('compareDottedVersions', () {
    test('equal versions compare to 0', () {
      expect(compareDottedVersions('1.2.0', '1.2.0'), 0);
    });

    test('a shorter version is padded with trailing zeros', () {
      expect(compareDottedVersions('1.2', '1.2.0'), 0);
      expect(compareDottedVersions('1.2.0', '1.2'), 0);
    });

    test('numeric (not lexicographic) segment comparison', () {
      expect(compareDottedVersions('1.2.0', '1.10.0'), lessThan(0));
      expect(compareDottedVersions('1.10.0', '1.2.0'), greaterThan(0));
    });

    test('a lower version compares less than a higher one', () {
      expect(compareDottedVersions('1.9.9', '2.0.0'), lessThan(0));
      expect(compareDottedVersions('2.0.0', '1.9.9'), greaterThan(0));
    });

    test('malformed input on either side returns null, never throws', () {
      expect(compareDottedVersions('1.2.x', '1.2.0'), isNull);
      expect(compareDottedVersions('1.2.0', 'not-a-version'), isNull);
      expect(compareDottedVersions('', '1.0.0'), isNull);
      expect(compareDottedVersions('1.0.0', ''), isNull);
    });
  });
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `flutter test test/semver_test.dart`
Expected: FAIL — `package:newsfeed/core/utils/semver.dart` doesn't exist yet.

- [ ] **Step 3: Implement the comparator**

```dart
// lib/core/utils/semver.dart

/// Compares two dotted-integer version strings (e.g. "1.2.0" vs "1.10.0")
/// numerically, segment by segment. A missing trailing segment counts as
/// `0`, so `"1.2"` and `"1.2.0"` compare equal.
///
/// Returns the same sign contract as [Comparable.compareTo] — negative if
/// [a] < [b], zero if equal, positive if [a] > [b] — or `null` if either
/// string isn't purely dot-separated non-negative integers. Callers treat
/// `null` as "can't tell, don't block" (fail open) rather than risking a
/// wrong compare off a malformed remote or platform value.
int? compareDottedVersions(String a, String b) {
  final segmentsA = _parseSegments(a);
  final segmentsB = _parseSegments(b);
  if (segmentsA == null || segmentsB == null) return null;

  final length = segmentsA.length > segmentsB.length
      ? segmentsA.length
      : segmentsB.length;
  for (var i = 0; i < length; i++) {
    final valueA = i < segmentsA.length ? segmentsA[i] : 0;
    final valueB = i < segmentsB.length ? segmentsB[i] : 0;
    final comparison = valueA.compareTo(valueB);
    if (comparison != 0) return comparison;
  }
  return 0;
}

List<int>? _parseSegments(String version) {
  if (version.isEmpty) return null;
  final parts = version.split('.');
  final segments = <int>[];
  for (final part in parts) {
    final value = int.tryParse(part);
    if (value == null || value < 0) return null;
    segments.add(value);
  }
  return segments;
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `flutter test test/semver_test.dart`
Expected: PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git add lib/core/utils/semver.dart test/semver_test.dart
git commit -m "Add dotted-version comparator for the update-gate check"
```

---

### Task 2: `minSupportedVersion` constant + remote config export

**Files:**
- Create: `lib/core/constants/app_version_config.dart`
- Modify: `tool/export_remote_config_test.dart`

**Interfaces:**
- Produces: `const String kMinSupportedVersion` — the bundled default minimum version, consumed by Task 3's `MinSupportedVersionNotifier` indirectly (it's not read at runtime by the app itself, only exported to JSON — the app only ever reads the *remote* value, never this constant, so a locally-installed app is never gated by its own build).

- [ ] **Step 1: Add the constant**

```dart
// lib/core/constants/app_version_config.dart

/// The lowest app version allowed to run. Consumed only by
/// `tool/export_remote_config_test.dart`, which bakes it into
/// `docs/config/app_version.json` — the app itself never reads this
/// constant directly, only the remote JSON (see
/// `minSupportedVersionProvider` in `lib/core/providers/catalog_providers.dart`).
///
/// To force devices below some version to update: bump this, regenerate
/// (`flutter test tool/export_remote_config_test.dart`), then publish
/// (`git subtree push --prefix=docs config-pages main`) — no app release
/// needed. Left at the app's own current version by default, so nothing
/// is blocked until this is deliberately raised above a prior release.
const kMinSupportedVersion = '1.0.0';
```

- [ ] **Step 2: Extend the export tool to also generate `app_version.json`**

Read `tool/export_remote_config_test.dart` first (it currently exports
`kDefaultInterestCategories`/`kDefaultLocalNewsRegions` to
`docs/config/*.json` inside one `test('export remote config json', ...)`
block). Add an import for `app_version_config.dart` and, inside that same
test body, after the two existing `File(...).writeAsStringSync(...)`
calls:

```dart
    final appVersionJson = encoder.convert({
      'minSupportedVersion': kMinSupportedVersion,
    });
    File('docs/config/app_version.json').writeAsStringSync('$appVersionJson\n');
```

- [ ] **Step 3: Run the export tool**

Run: `flutter test tool/export_remote_config_test.dart`
Expected: PASS, and `docs/config/app_version.json` now exists.

- [ ] **Step 4: Verify the generated file**

Read `docs/config/app_version.json` and confirm it is exactly:

```json
{
  "minSupportedVersion": "1.0.0"
}
```

- [ ] **Step 5: Commit**

```bash
git add lib/core/constants/app_version_config.dart tool/export_remote_config_test.dart docs/config/app_version.json
git commit -m "Generate remote app_version.json from a bundled minSupportedVersion constant"
```

---

### Task 3: `minSupportedVersionProvider` (fetch/cache/fallback)

**Files:**
- Modify: `lib/core/storage/local_storage_service.dart` (around `local_storage_service.dart:21` for the key constant, `local_storage_service.dart:154-171` for the cache getters/setters section)
- Modify: `lib/core/providers/catalog_providers.dart`
- Test: `test/catalog_providers_test.dart`

**Interfaces:**
- Consumes: `localStorageServiceProvider` and `dioProvider` from `lib/core/providers/core_providers.dart` (both already exist and are already used by the two sibling notifiers in `catalog_providers.dart`).
- Produces: `final minSupportedVersionProvider = NotifierProvider<MinSupportedVersionNotifier, String?>(...)` — state is the last-confirmed minimum-version string, or `null` if none has ever been confirmed. Consumed by Task 4's `updateRequiredProvider`.
- Produces on `LocalStorageService`: `String? get cachedAppVersionJson` and `Future<void> setCachedAppVersionJson(String json)`.

- [ ] **Step 1: Add the cache key and accessors to `LocalStorageService`**

In `lib/core/storage/local_storage_service.dart`, next to the existing
`_kInterestCategoriesConfigKey`/`_kLocalNewsRegionsConfigKey` constants,
add:

```dart
  static const _kAppVersionConfigKey = 'remote_app_version';
```

And in the "Remote catalog cache" section at the bottom of the class,
next to the two existing get/set pairs, add:

```dart
  String? get cachedAppVersionJson => _prefs.getString(_kAppVersionConfigKey);

  Future<void> setCachedAppVersionJson(String json) =>
      _prefs.setString(_kAppVersionConfigKey, json);
```

- [ ] **Step 2: Write the failing notifier tests**

Append to `test/catalog_providers_test.dart` (same file already has the
`_FakeAdapter` helper and the `dart:typed_data`/`dio`/`shared_preferences`
imports — add `import 'package:newsfeed/core/providers/catalog_providers.dart';`
is already there too):

```dart
const _sampleAppVersionJson = '{"minSupportedVersion":"2.0.0"}';

// ... inside the existing `void main() { ... }`, alongside the existing
// `test(...)` call:

test(
  'a cached minimum version is available instantly, before any fetch resolves',
  () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    await storage.setCachedAppVersionJson(_sampleAppVersionJson);

    final container = ProviderContainer(overrides: [
      localStorageServiceProvider.overrideWithValue(storage),
      dioProvider.overrideWithValue(
        Dio()..httpClientAdapter = _FakeAdapter(_sampleAppVersionJson),
      ),
    ]);
    addTearDown(container.dispose);

    expect(container.read(minSupportedVersionProvider), '2.0.0');
  },
);

test('a successful fetch updates state and the on-disk cache', () async {
  SharedPreferences.setMockInitialValues({});
  final storage = LocalStorageService(await SharedPreferences.getInstance());

  final container = ProviderContainer(overrides: [
    localStorageServiceProvider.overrideWithValue(storage),
    dioProvider.overrideWithValue(
      Dio()..httpClientAdapter = _FakeAdapter(_sampleAppVersionJson),
    ),
  ]);
  addTearDown(container.dispose);

  // Nothing cached yet, so the initial state is null.
  expect(container.read(minSupportedVersionProvider), isNull);

  // Let the notifier's fire-and-forget background refresh run.
  await Future<void>.delayed(Duration.zero);

  expect(container.read(minSupportedVersionProvider), '2.0.0');
  expect(storage.cachedAppVersionJson, _sampleAppVersionJson);
});

test('a failed fetch leaves a previously-confirmed minimum enforced', () async {
  SharedPreferences.setMockInitialValues({});
  final storage = LocalStorageService(await SharedPreferences.getInstance());
  await storage.setCachedAppVersionJson(_sampleAppVersionJson);

  final container = ProviderContainer(overrides: [
    localStorageServiceProvider.overrideWithValue(storage),
    dioProvider.overrideWithValue(
      Dio()..httpClientAdapter = _ThrowingAdapter(),
    ),
  ]);
  addTearDown(container.dispose);

  expect(container.read(minSupportedVersionProvider), '2.0.0');
  await Future<void>.delayed(Duration.zero);
  // Still '2.0.0' — the failed fetch didn't clear the prior confirmation.
  expect(container.read(minSupportedVersionProvider), '2.0.0');
});
```

Add the `_ThrowingAdapter` helper next to `_FakeAdapter` at the top of the
file:

```dart
/// Simulates a network failure for every request.
class _ThrowingAdapter implements HttpClientAdapter {
  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) {
    throw DioException(requestOptions: options, error: 'simulated failure');
  }

  @override
  void close({bool force = false}) {}
}
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `flutter test test/catalog_providers_test.dart`
Expected: FAIL — `minSupportedVersionProvider` doesn't exist yet.

- [ ] **Step 4: Implement `MinSupportedVersionNotifier`**

Append to `lib/core/providers/catalog_providers.dart` (after the existing
`localNewsRegionsProvider` declaration), matching the shape of
`LocalNewsRegionsNotifier` immediately above it:

```dart
const _kAppVersionUrl =
    'https://doronk.github.io/newsfeed-config/config/app_version.json';

/// The minimum app version this device is allowed to run, as last
/// confirmed by a successful remote fetch — or `null` if none has ever
/// been confirmed (never fetched, fetch failed, malformed JSON). `null`
/// means "no minimum enforced": see `updateRequiredProvider` in
/// `lib/core/providers/update_gate_providers.dart`, which fails open on
/// `null` rather than blocking on an unknown.
///
/// Same instant-fallback + one-shot background-refresh shape as
/// [InterestCategoriesNotifier]/[LocalNewsRegionsNotifier], except there's
/// no bundled default to fall back to — an unconfirmed minimum means "no
/// minimum", not "use some hardcoded value".
class MinSupportedVersionNotifier extends Notifier<String?> {
  @override
  String? build() {
    final cached = _readCache();
    _refreshOnce();
    return cached;
  }

  String? _readCache() {
    final raw = ref.read(localStorageServiceProvider).cachedAppVersionJson;
    if (raw == null) return null;
    try {
      final decoded = jsonDecode(raw) as Map<String, dynamic>;
      return decoded['minSupportedVersion'] as String?;
    } catch (_) {
      return null;
    }
  }

  Future<void> _refreshOnce() async {
    try {
      final dio = ref.read(dioProvider);
      final response = await dio.get<String>(_kAppVersionUrl);
      final body = response.data;
      if (body == null || body.isEmpty) return;
      final storage = ref.read(localStorageServiceProvider);
      // See the matching comment in LocalNewsRegionsNotifier._refreshOnce
      // — same fix, same reason.
      if (body == storage.cachedAppVersionJson) return;
      final decoded = jsonDecode(body) as Map<String, dynamic>;
      final minVersion = decoded['minSupportedVersion'] as String?;
      if (minVersion == null || minVersion.isEmpty || !ref.mounted) return;
      state = minVersion;
      await storage.setCachedAppVersionJson(body);
    } catch (_) {
      // Keep whatever's already in state (null, or last-confirmed).
    }
  }
}

final minSupportedVersionProvider =
    NotifierProvider<MinSupportedVersionNotifier, String?>(
  MinSupportedVersionNotifier.new,
);
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `flutter test test/catalog_providers_test.dart`
Expected: PASS (existing tests plus the 3 new ones)

- [ ] **Step 6: Commit**

```bash
git add lib/core/storage/local_storage_service.dart lib/core/providers/catalog_providers.dart test/catalog_providers_test.dart
git commit -m "Add minSupportedVersionProvider: fetch/cache/fallback for the update gate"
```

---

### Task 4: `updateRequiredProvider` (installed-version check)

**Files:**
- Modify: `pubspec.yaml` (add `package_info_plus: ^10.2.1` under `dependencies:`)
- Create: `lib/core/providers/update_gate_providers.dart`
- Test: `test/update_gate_providers_test.dart`

**Interfaces:**
- Consumes: `compareDottedVersions` from `lib/core/utils/semver.dart` (Task 1); `minSupportedVersionProvider` from `lib/core/providers/catalog_providers.dart` (Task 3).
- Produces: `final installedAppVersionProvider = FutureProvider<String>(...)`; `final updateRequiredProvider = Provider<bool>(...)` — consumed by Task 5's `app.dart` wiring.

- [ ] **Step 1: Add the dependency**

In `pubspec.yaml`, under `dependencies:` next to `home_widget: ^0.9.3`:

```yaml
  package_info_plus: ^10.2.1
```

Run: `flutter pub get`

- [ ] **Step 2: Write the failing tests**

```dart
// test/update_gate_providers_test.dart
import 'dart:typed_data';

import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/providers/core_providers.dart';
import 'package:newsfeed/core/providers/update_gate_providers.dart';
import 'package:newsfeed/core/storage/local_storage_service.dart';
import 'package:shared_preferences/shared_preferences.dart';

/// Returns [body] for every request, matching the sibling helper in
/// `test/catalog_providers_test.dart`.
class _FakeAdapter implements HttpClientAdapter {
  _FakeAdapter(this.body);
  final String body;

  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    return ResponseBody.fromString(body, 200, headers: {
      Headers.contentTypeHeader: [Headers.textPlainContentType],
    });
  }

  @override
  void close({bool force = false}) {}
}

void main() {
  Future<ProviderContainer> buildContainer({
    required String cachedMinVersionJson,
    required String installedVersion,
  }) async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());
    await storage.setCachedAppVersionJson(cachedMinVersionJson);

    final container = ProviderContainer(overrides: [
      localStorageServiceProvider.overrideWithValue(storage),
      dioProvider.overrideWithValue(
        Dio()..httpClientAdapter = _FakeAdapter(cachedMinVersionJson),
      ),
      installedAppVersionProvider.overrideWith((ref) async => installedVersion),
    ]);
    return container;
  }

  test('installed version below the minimum requires an update', () async {
    final container = await buildContainer(
      cachedMinVersionJson: '{"minSupportedVersion":"2.0.0"}',
      installedVersion: '1.0.0',
    );
    addTearDown(container.dispose);

    await container.read(installedAppVersionProvider.future);
    expect(container.read(updateRequiredProvider), isTrue);
  });

  test('installed version at or above the minimum does not require an update', () async {
    final container = await buildContainer(
      cachedMinVersionJson: '{"minSupportedVersion":"2.0.0"}',
      installedVersion: '2.0.0',
    );
    addTearDown(container.dispose);

    await container.read(installedAppVersionProvider.future);
    expect(container.read(updateRequiredProvider), isFalse);
  });

  test('no confirmed minimum version fails open (never requires an update)', () async {
    SharedPreferences.setMockInitialValues({});
    final storage = LocalStorageService(await SharedPreferences.getInstance());

    final container = ProviderContainer(overrides: [
      localStorageServiceProvider.overrideWithValue(storage),
      dioProvider.overrideWithValue(
        Dio()..httpClientAdapter = _FakeAdapter('not json'),
      ),
      installedAppVersionProvider.overrideWith((ref) async => '0.0.1'),
    ]);
    addTearDown(container.dispose);

    await container.read(installedAppVersionProvider.future);
    expect(container.read(updateRequiredProvider), isFalse);
  });

  test('an unparseable minimum version fails open (never requires an update)', () async {
    final container = await buildContainer(
      cachedMinVersionJson: '{"minSupportedVersion":"not-a-version"}',
      installedVersion: '1.0.0',
    );
    addTearDown(container.dispose);

    await container.read(installedAppVersionProvider.future);
    expect(container.read(updateRequiredProvider), isFalse);
  });
}
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `flutter test test/update_gate_providers_test.dart`
Expected: FAIL — `lib/core/providers/update_gate_providers.dart` doesn't exist yet.

- [ ] **Step 4: Implement the providers**

```dart
// lib/core/providers/update_gate_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:package_info_plus/package_info_plus.dart';

import '../utils/semver.dart';
import 'catalog_providers.dart';

/// The installed app's version string (e.g. "1.0.0"), read once from the
/// platform.
final installedAppVersionProvider = FutureProvider<String>((ref) async {
  final info = await PackageInfo.fromPlatform();
  return info.version;
});

/// True only once a remote fetch has confirmed the installed version is
/// below [minSupportedVersionProvider]. Every other case — no confirmed
/// minimum yet, the installed version still loading, either version
/// string unparseable — fails open to `false`. See
/// `docs/superpowers/specs/2026-08-31-remote-force-update-design.md` for
/// why: a network hiccup or malformed config must never lock out a
/// legitimate user.
final updateRequiredProvider = Provider<bool>((ref) {
  final minVersion = ref.watch(minSupportedVersionProvider);
  final installedVersion = ref.watch(installedAppVersionProvider).valueOrNull;
  if (minVersion == null || installedVersion == null) return false;
  final comparison = compareDottedVersions(installedVersion, minVersion);
  return comparison != null && comparison < 0;
});
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `flutter test test/update_gate_providers_test.dart`
Expected: PASS (4 tests)

- [ ] **Step 6: Commit**

```bash
git add pubspec.yaml pubspec.lock lib/core/providers/update_gate_providers.dart test/update_gate_providers_test.dart
git commit -m "Add updateRequiredProvider: gate on installed vs. minimum version"
```

---

### Task 5: `UpdateRequiredScreen` + wiring into `app.dart`

**Files:**
- Create: `lib/features/update_required/presentation/screens/update_required_screen.dart`
- Modify: `lib/app.dart`
- Test: `test/app_test.dart`

**Interfaces:**
- Consumes: `updateRequiredProvider` from `lib/core/providers/update_gate_providers.dart` (Task 4); `safeArticleUri` from `lib/core/utils/safe_link.dart` (existing).
- Produces: `class UpdateRequiredScreen extends StatelessWidget` with a public no-arg `const` constructor — this is the full public surface; nothing else depends on it beyond `app.dart`.

- [ ] **Step 1: Write the failing widget tests**

```dart
// test/app_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/app.dart';
import 'package:newsfeed/core/providers/core_providers.dart';
import 'package:newsfeed/core/providers/update_gate_providers.dart';
import 'package:newsfeed/core/router/app_router.dart';
import 'package:newsfeed/core/storage/local_storage_service.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  Future<LocalStorageService> buildStorage() async {
    SharedPreferences.setMockInitialValues({});
    return LocalStorageService(await SharedPreferences.getInstance());
  }

  testWidgets('shows the non-dismissible update screen when required', (tester) async {
    final storage = await buildStorage();
    final router = buildAppRouter(initialLocation: '/onboarding');

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          localStorageServiceProvider.overrideWithValue(storage),
          updateRequiredProvider.overrideWithValue(true),
        ],
        child: NewsfeedApp(router: router),
      ),
    );
    await tester.pumpAndSettle();

    expect(find.text('Update required'), findsOneWidget);
    expect(find.text('Update now'), findsOneWidget);

    // The system back gesture must be a no-op.
    final navigator = tester.state<NavigatorState>(find.byType(Navigator));
    final popped = await navigator.maybePop();
    expect(popped, isFalse);
    await tester.pumpAndSettle();
    expect(find.text('Update required'), findsOneWidget);
    expect(tester.takeException(), isNull);
  });

  testWidgets('renders the normal routed app when no update is required', (tester) async {
    final storage = await buildStorage();
    final router = buildAppRouter(initialLocation: '/onboarding');

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          localStorageServiceProvider.overrideWithValue(storage),
          updateRequiredProvider.overrideWithValue(false),
        ],
        child: NewsfeedApp(router: router),
      ),
    );
    await tester.pumpAndSettle();

    expect(find.text('Update required'), findsNothing);
    expect(tester.takeException(), isNull);
  });
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `flutter test test/app_test.dart`
Expected: FAIL — `updateRequiredProvider` isn't watched in `app.dart` yet, so `find.text('Update required')` finds nothing in the first test.

- [ ] **Step 3: Implement `UpdateRequiredScreen`**

```dart
// lib/features/update_required/presentation/screens/update_required_screen.dart
import 'package:flutter/material.dart';
import 'package:url_launcher/url_launcher.dart';

import '../../../../core/utils/safe_link.dart';

/// Full-screen, non-dismissible block shown by [NewsfeedApp] in place of
/// the normal routed app when `updateRequiredProvider` is true — a
/// remote fetch has confirmed the installed version is below the
/// enforced minimum. `NewsfeedApp` wraps this in its own `MaterialApp`
/// (not `go_router`), so there is no route stack to escape through; the
/// `PopScope` here additionally swallows the system back gesture.
class UpdateRequiredScreen extends StatelessWidget {
  const UpdateRequiredScreen({super.key});

  static const _playStoreUrl =
      'https://play.google.com/store/apps/details?id=com.dk.newsfeed';

  @override
  Widget build(BuildContext context) {
    return PopScope(
      canPop: false,
      child: Scaffold(
        body: SafeArea(
          child: Center(
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 32),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Image.asset('assets/icon/icon.png', width: 96, height: 96),
                  const SizedBox(height: 24),
                  Text(
                    'Update required',
                    style: Theme.of(context).textTheme.headlineSmall,
                    textAlign: TextAlign.center,
                  ),
                  const SizedBox(height: 12),
                  const Text(
                    'A new version of News Feed is required to keep using '
                    'the app. Please update from the Play Store.',
                    textAlign: TextAlign.center,
                  ),
                  const SizedBox(height: 24),
                  FilledButton(
                    onPressed: _openStore,
                    child: const Text('Update now'),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }

  Future<void> _openStore() async {
    final uri = safeArticleUri(_playStoreUrl);
    if (uri == null) return;
    await launchUrl(uri, mode: LaunchMode.externalApplication);
  }
}
```

- [ ] **Step 4: Wire the gate into `app.dart`**

In `lib/app.dart`, add imports for `update_gate_providers.dart` and
`update_required_screen.dart`:

```dart
import 'core/providers/update_gate_providers.dart';
import 'features/update_required/presentation/screens/update_required_screen.dart';
```

Then change the start of `_NewsfeedAppState.build` (currently
`lib/app.dart:70-82`) from:

```dart
  @override
  Widget build(BuildContext context) {
    final themeMode = ref.watch(themeModeProvider);

    return MaterialApp.router(
```

to:

```dart
  @override
  Widget build(BuildContext context) {
    if (ref.watch(updateRequiredProvider)) {
      return const MaterialApp(
        debugShowCheckedModeBanner: false,
        home: UpdateRequiredScreen(),
      );
    }

    final themeMode = ref.watch(themeModeProvider);

    return MaterialApp.router(
```

(leave the rest of the method — `title`, `theme`, `darkTheme`,
`routerConfig` — unchanged).

- [ ] **Step 5: Run the tests to verify they pass**

Run: `flutter test test/app_test.dart`
Expected: PASS (2 tests)

- [ ] **Step 6: Run the full test suite**

Run: `flutter test`
Expected: PASS — every existing test plus all new ones from Tasks 1-5.

- [ ] **Step 7: Commit**

```bash
git add lib/features/update_required/presentation/screens/update_required_screen.dart lib/app.dart test/app_test.dart
git commit -m "Add UpdateRequiredScreen and wire the remote force-update gate into app.dart"
```

---

## Self-Review Notes

- **Spec coverage:** config file + generation (Task 2), fetch/cache/fallback notifier with fail-open on fetch failure (Task 3), installed-version read + comparator-driven derived boolean with fail-open on missing/unparseable data (Tasks 1 & 4), hard-block UI wrapped in its own `MaterialApp` with `PopScope` and a Play-Store button (Task 5), full test coverage per the spec's Testing section (Tasks 1, 3, 4, 5) — every spec section maps to a task.
- **Placeholder scan:** no TBD/TODO markers; every step carries the literal code or command to run.
- **Type consistency:** `String? minSupportedVersionProvider` (Task 3) is consumed as `String?` by `updateRequiredProvider` (Task 4); `FutureProvider<String> installedAppVersionProvider` is read via `.valueOrNull` (nullable `String?`) in the same task; `int? compareDottedVersions(String, String)` (Task 1) signature matches its one call site in Task 4 exactly; `UpdateRequiredScreen` (Task 5) is a bare `const`-constructible `StatelessWidget`, matching how `app.dart` instantiates it.
