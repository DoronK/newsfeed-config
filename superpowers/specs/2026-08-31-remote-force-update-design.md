# Remote force-update mechanism

## Purpose

Let a device be blocked from using the app when its installed version
falls below a minimum the developer can raise remotely, with no app
release required — reusing the existing "static JSON on GitHub Pages"
remote-config pattern already used for the interest-category and
local-news-source catalogs (see `lib/core/providers/catalog_providers.dart`).

## Requirements (from brainstorming)

- **Block strength:** hard block. Below the minimum version, a
  full-screen, non-dismissible "update required" screen replaces the
  entire app UI — no back button, no way to reach any content.
- **Update target:** a hardcoded Google Play Store URL for
  `com.dk.newsfeed` (not itself remote-configurable — YAGNI; add an
  App Store URL and platform-branch the two once iOS ships).
- **Fetch-failure behavior:** fail open. No confirmed minimum version
  (never fetched yet, this session's fetch failed, JSON malformed)
  means the app runs normally. The block appears only once a fetch has
  actually confirmed the device is below the minimum. A minimum
  confirmed by a *previous* successful fetch stays enforced even if
  today's refresh fails — that's a real, already-confirmed
  requirement, not an unknown.
- **Threshold shape:** single threshold, `minSupportedVersion`. No
  second "recommended update" nudge tier — below the threshold blocks,
  at or above it shows nothing at all.
- **No periodic re-check** while the app is already running — checked
  once per app process, same as the existing catalog providers.

## Data flow

### Config file

New `docs/config/app_version.json`:

```json
{
  "minSupportedVersion": "1.0.0"
}
```

Generated the same way as the two existing catalog files: a new Dart
constant (`kMinSupportedVersion`, a plain dotted-version string) is
added near the other bundled defaults, and
`tool/export_remote_config_test.dart` is extended to also write this
file from that constant. Bumping the minimum is: edit the constant →
`flutter test tool/export_remote_config_test.dart` → `git subtree push
--prefix=docs config-pages main` (existing workflow, per
`CLAUDE.md`/`README.md`).

Hosted at
`https://doronk.github.io/newsfeed-config/config/app_version.json`,
matching the existing two URLs' shape.

### Notifier

New `MinSupportedVersionNotifier` in `catalog_providers.dart`,
structurally identical to `InterestCategoriesNotifier` /
`LocalNewsRegionsNotifier`:

- `build()`: read the cached string from `SharedPreferences`
  (`cachedAppVersionJson`, a new key in `LocalStorageService`, same
  shape as the two existing cache keys), decode
  `minSupportedVersion` out of it if present, kick off one
  fire-and-forget `_refreshOnce()`, return the cached value (or
  `null` if there's never been one) as the initial state.
- `_refreshOnce()`: `dio.get` the JSON, decode it, and if the raw body
  is unchanged from what's cached, bail out before touching `state`
  (same "avoid a spurious rebuild of every dependent provider" guard
  the two existing notifiers use). Any failure (network, non-200,
  malformed JSON, missing key) is swallowed — state is left exactly as
  it was (`null`, or whatever was last confirmed).
- State type: `String?` — the minimum-version string that's been
  confirmed applicable to this device, or `null` when there is none.

`minSupportedVersionProvider` exposes this via
`NotifierProvider<MinSupportedVersionNotifier, String?>`.

### Version comparison

- New dependency: `package_info_plus`, used once via
  `PackageInfo.fromPlatform()` to read the installed version string
  (matches the `version:` field in `pubspec.yaml`, e.g. `1.0.0`).
  Exposed as a `FutureProvider<String>` (`installedAppVersionProvider`).
- New `lib/core/utils/semver.dart`: a small dotted-integer version
  comparator, no new dependency needed for this part. Parses each
  dot-separated segment as an int; compares segment-by-segment,
  treating a missing trailing segment as `0` (so `1.2` == `1.2.0`).
  Returns `null` (rather than throwing) if either string fails to
  parse as dotted integers — callers treat `null` as "can't tell,
  don't block."
- `updateRequiredProvider` (`Provider<bool>`): watches
  `minSupportedVersionProvider` and `installedAppVersionProvider`.
  `true` only when both are available *and* the comparator says
  installed < minimum. Any missing/unresolved/incomparable case is
  `false` — consistent fail-open behavior at every layer, not just the
  network layer.

## UI

New `UpdateRequiredScreen` widget
(`lib/features/update_required/presentation/update_required_screen.dart`
or similar, following the existing `features/<name>/presentation`
layout):

- Wrapped in its own `MaterialApp` (not `.router`) — there is no route
  stack to navigate away through, by construction.
- `PopScope(canPop: false)` so the Android system back gesture/button
  is also a no-op.
- Content: app icon, "Update required" headline, short explanatory
  copy, and an "Update now" button.
- The button opens
  `https://play.google.com/store/apps/details?id=com.dk.newsfeed`
  via `url_launcher` (`LaunchMode.externalApplication`), guarded by
  the same http(s)-only validation helper style already used for
  feed-derived links (`lib/core/utils/safe_link.dart`) — belt-and-braces
  even though this URL is a compile-time constant, not feed-derived.

### Wiring into `app.dart`

In `_NewsfeedAppState.build()`, watch `updateRequiredProvider` first:

```dart
if (ref.watch(updateRequiredProvider)) {
  return const UpdateRequiredScreen();
}
```

before building/returning the normal `MaterialApp.router`. Everything
else in that build method (theme, router) is simply not reached in the
blocked case. `initState`'s home-widget-tap listener still runs (it's
lifecycle-bound, not build-bound) but its handler navigates via
`widget.router`, which is never pushed to a visible `Navigator` when
blocked, so it's inert — no special-casing needed there.

## Testing

- **`test/semver_test.dart`** (new): equal versions, differing segment
  counts (`1.2` vs `1.2.0`), clearly-below and clearly-above cases,
  and malformed input on either side (returns `null`, not a throw).
- **`test/catalog_providers_test.dart`** (extended): add cases for
  `MinSupportedVersionNotifier` mirroring the existing fake-adapter
  tests — cache-hit instant value, a successful fetch updating the
  cache, and a failed fetch leaving prior state (`null`, or a
  previously-confirmed version) untouched.
- **Widget test** (new or extended `app_test.dart`): with
  `updateRequiredProvider` overridden `true`, pumping `NewsfeedApp`
  renders `UpdateRequiredScreen` and a simulated back-button pop is a
  no-op; overridden `false`, it renders the normal routed app.
- **`tool/export_remote_config_test.dart`** (extended): also asserts
  `docs/config/app_version.json` is written/regenerated correctly.

## Out of scope (YAGNI, can follow later without redesign)

- A second "recommended update" soft-nudge tier.
- Remote-configuring the store URL itself.
- An iOS App Store URL / platform branch (added once iOS ships, per
  the existing "iOS pending" note on the home-screen widget feature).
- Periodic re-check while the app is already running.
