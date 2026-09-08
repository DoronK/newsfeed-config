# Outlet Badges on Article Cards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Surface the outlet-type badge (wire/national/international/
regional/specialty) that already exists in Manage Sources on the feed's
`ArticleCard` and on `ArticleDetailScreen`, by threading a new
`sourceType` field through `Article`.

**Architecture:** `SourceType` (in `interest_categories.dart`) gains
`badgeLabel`/`longLabel`/`fromName` helpers. `Article` gains a required
`sourceType` field, populated at RSS/Atom mapping time from the
`NewsSource` already in hand, and round-tripped through
`toJson`/`fromJson` with a `custom` (no-badge) fallback for pre-existing
cached data. A new shared `SourceTypeBadge` widget (overlay + inline
variants) renders it — bottom-left image overlay on `ArticleCard`, an
inline pill next to the category chip on `ArticleDetailScreen`.

**Tech Stack:** Flutter, no new dependencies. `flutter_test` +
`flutter_riverpod`'s `ProviderScope` overrides (existing test patterns).

**Spec:** `docs/superpowers/specs/2026-09-07-article-card-outlet-badges-design.md`

## Global Constraints

- Never show a badge for `SourceType.custom`, or for an article whose
  stored type can't be resolved (missing key, unrecognized value) —
  both fall back to `custom` via `SourceType.fromName`, never a crash and
  never a wrong/blank badge shown as if real.
- `Article.sourceType` is `required` — every construction site (2
  production, 4 test fixtures) must be updated in the task that touches
  it; don't leave one defaulted as a shortcut.
- No change to `RssRemoteDataSource`'s network/fetch behavior,
  `LocalStorageService`'s cache key/schema shape, or any screen besides
  `ArticleCard`/`ArticleDetailScreen`.
- The badge is never interactive (no `onTap`) and never conveys type by
  color alone — text label only, per the spec's accessibility section.

---

### Task 1: `SourceType` helpers + `Article.sourceType` (domain layer, no UI)

**Files:**
- Modify: `lib/core/constants/interest_categories.dart`
- Modify: `lib/features/feed/domain/entities/article.dart`
- Modify: `lib/features/feed/data/datasources/rss_remote_data_source.dart`
- Modify (test fixtures for the new required field): `test/article_grid_layout_test.dart`, `test/feed_providers_test.dart`, `test/article_repository_test.dart`, `test/screens_smoke_test.dart`
- Modify: `test/interest_categories_test.dart`
- Modify: `test/rss_remote_data_source_test.dart`
- Create: `test/article_entity_test.dart`

**Interfaces:**
- Produces: `String get SourceType.longLabel`; `String? get SourceType.badgeLabel` (null iff `custom`); `static SourceType SourceType.fromName(String? name)` (tolerant parser, falls back to `custom`).
- Produces: `Article.sourceType` (required `SourceType` field), serialized as `sourceType` (the enum's `.name`) in `toJson`/`fromJson`.
- Consumed by: Task 2 (`SourceTypeBadge` widget reads `type.badgeLabel`/`type.longLabel`).

**Surface:** app

- [x] **Step 1: Write the failing `SourceType` tests**

Add to `test/interest_categories_test.dart` (new top-level `group`,
alongside the existing ones):

```dart
  group('SourceType', () {
    test('badgeLabel is null for custom, non-null for every other value', () {
      for (final type in SourceType.values) {
        if (type == SourceType.custom) {
          expect(type.badgeLabel, isNull, reason: type.name);
        } else {
          expect(type.badgeLabel, isNotEmpty, reason: type.name);
          expect(type.badgeLabel, type.label);
        }
      }
    });

    test('longLabel is non-empty for every value', () {
      for (final type in SourceType.values) {
        expect(type.longLabel, isNotEmpty, reason: type.name);
      }
    });

    test('fromName round-trips every value\'s own name', () {
      for (final type in SourceType.values) {
        expect(SourceType.fromName(type.name), type);
      }
    });

    test('fromName falls back to custom for null or an unrecognized name', () {
      expect(SourceType.fromName(null), SourceType.custom);
      expect(SourceType.fromName('not-a-real-type'), SourceType.custom);
    });
  });
```

- [x] **Step 2: Run the tests to verify they fail**

Run: `flutter test test/interest_categories_test.dart`
Expected: FAIL — `badgeLabel`/`longLabel`/`fromName` don't exist yet.

- [x] **Step 3: Add the `SourceType` members**

In `lib/core/constants/interest_categories.dart`, in the `SourceType`
enum, after the existing `label` getter:

```dart
  /// Full-word form of [label], for contexts with more room (the article
  /// detail screen, accessibility strings) where the abbreviated [label]
  /// risks reading as a second category — "National" next to a
  /// "Technology" pill looks like a category; "National outlet" doesn't.
  String get longLabel => switch (this) {
        SourceType.wireService => 'Wire service',
        SourceType.nationalOutlet => 'National outlet',
        SourceType.internationalOutlet => 'International outlet',
        SourceType.regionalOutlet => 'Regional outlet',
        SourceType.specialtyPublication => 'Specialty publication',
        SourceType.custom => 'Custom',
      };

  /// [label], or `null` for [custom] — user-added feeds are never shown
  /// with an outlet badge (see the doc comment on this enum). Centralizes
  /// that rule here so every badge-rendering widget can render `null` as
  /// "nothing" without its own `== SourceType.custom` check.
  String? get badgeLabel => this == SourceType.custom ? null : label;

  /// Tolerant parser for a serialized [SourceType] name (from [NewsSource]
  /// or [Article] JSON). Falls back to [custom] — no badge — for `null`,
  /// a missing key, or any unrecognized/future value, rather than
  /// throwing. Single source of truth for this fallback; both
  /// [NewsSource.fromJson] and `Article.fromJson` use it.
  static SourceType fromName(String? name) => SourceType.values.firstWhere(
        (t) => t.name == name,
        orElse: () => SourceType.custom,
      );
```

- [x] **Step 4: Refactor `NewsSource.fromJson` onto `fromName`**

In the same file, `NewsSource.fromJson`, replace:

```dart
        type: SourceType.values.firstWhere(
          (t) => t.name == json['type'],
          orElse: () => SourceType.custom,
        ),
```

with:

```dart
        type: SourceType.fromName(json['type'] as String?),
```

- [x] **Step 5: Run the tests to verify they pass**

Run: `flutter test test/interest_categories_test.dart`
Expected: PASS (existing tests + 4 new ones). This confirms the refactor
in Step 4 didn't change `NewsSource.fromJson`'s observable behavior.

- [x] **Step 6: Write the failing `Article` entity tests**

Create `test/article_entity_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/constants/interest_categories.dart';
import 'package:newsfeed/features/feed/domain/entities/article.dart';

Article _article({required SourceType sourceType}) => Article(
      id: '1',
      title: 'Title',
      summary: 'Summary',
      content: 'Content',
      link: 'https://example.com/1',
      imageUrl: null,
      sourceName: 'Source',
      sourceUrl: 'https://example.com/feed',
      categoryId: 'technology',
      publishedAt: DateTime.utc(2026, 1, 1),
      language: 'en',
      sourceType: sourceType,
    );

void main() {
  group('Article sourceType round-trip', () {
    test('toJson/fromJson round-trips a non-custom sourceType', () {
      final article = _article(sourceType: SourceType.nationalOutlet);
      final decoded = Article.fromJson(article.toJson());
      expect(decoded.sourceType, SourceType.nationalOutlet);
    });

    test('a JSON map missing sourceType falls back to custom (pre-existing cache)', () {
      final json = _article(sourceType: SourceType.custom).toJson()
        ..remove('sourceType');
      final decoded = Article.fromJson(json);
      expect(decoded.sourceType, SourceType.custom);
    });

    test('an unrecognized sourceType string falls back to custom', () {
      final json = _article(sourceType: SourceType.custom).toJson();
      json['sourceType'] = 'not-a-real-type';
      final decoded = Article.fromJson(json);
      expect(decoded.sourceType, SourceType.custom);
    });
  });
}
```

- [x] **Step 7: Run the tests to verify they fail**

Run: `flutter test test/article_entity_test.dart`
Expected: FAIL — `Article` doesn't have a `sourceType` parameter yet.

- [x] **Step 8: Add `sourceType` to `Article`**

In `lib/features/feed/domain/entities/article.dart`:

Add the import at the top:

```dart
import '../../../../core/constants/interest_categories.dart' show SourceType;
```

Add the field (after `language`):

```dart
  /// What kind of outlet [sourceName] is (see [SourceType]) — shown as a
  /// badge on the article card/detail screen. `SourceType.custom` (no
  /// badge) for user-added feeds, and for any article cached before this
  /// field existed (see [fromJson]).
  final SourceType sourceType;
```

Add it to the constructor (after `required this.language,`):

```dart
    required this.sourceType,
```

Add it to `toJson` (after `'language': language,`):

```dart
        'sourceType': sourceType.name,
```

Add it to `fromJson` (after the `language` line), with a comment matching
this factory's existing style:

```dart
        // Falls back to `custom` (no badge) for articles cached before
        // this field existed, or an unrecognized/future value.
        sourceType: SourceType.fromName(json['sourceType'] as String?),
```

- [x] **Step 9: Update the 4 existing test fixtures for the new required field**

Add `sourceType: SourceType.custom,` (or a more specific value where it
makes the test's intent clearer — see Step 10 for the one exception) to
each existing `Article(...)` literal, and add
`import 'package:newsfeed/core/constants/interest_categories.dart';` to
each file that doesn't already import it:

- `test/screens_smoke_test.dart:162` — add `sourceType: SourceType.custom,`.
- `test/article_repository_test.dart:15` — add `sourceType: SourceType.custom,`.
- `test/feed_providers_test.dart:33` — add `sourceType: SourceType.custom,`.
- `test/article_grid_layout_test.dart:14` (`_worstCaseArticle`) — add
  `sourceType: SourceType.internationalOutlet,` instead (the longest
  `label`/`longLabel` value), so the worst-case layout fixture also
  exercises the new overlay badge's longest string — see Task 2, Step 6.

- [x] **Step 10: Run the tests to verify they pass**

Run: `flutter test test/article_entity_test.dart test/screens_smoke_test.dart test/article_repository_test.dart test/feed_providers_test.dart test/article_grid_layout_test.dart`
Expected: PASS. (`article_grid_layout_test.dart` still passes at this
point since it doesn't yet render a source-type badge — that's Task 2.)

- [x] **Step 11: Write the failing `RssRemoteDataSource` mapping tests**

In `test/rss_remote_data_source_test.dart`, change `_fetch` to accept an
optional `NewsSource` type override, and use it in a new group:

```dart
Future<List<dynamic>> _fetch(String xml, {SourceType type = SourceType.custom}) async {
  final dio = Dio()..httpClientAdapter = _FakeAdapter(xml);
  final source = RssRemoteDataSource(dio);
  return source.fetchSource(
    NewsSource(name: 'Test', rssUrl: 'https://example.com/feed', type: type),
    'technology',
  );
}
```

(`NewsSource(...)` above drops the `const` the current line 33 uses,
since `type` is now a runtime parameter — `NewsSource`'s constructor
itself is unaffected.)

Add a new group at the end of `main()`:

```dart
  group('RssRemoteDataSource source type', () {
    test('stamps each RSS article with the source\'s type', () async {
      const xml = '''
      <rss version="2.0"><channel>
        <item>
          <title>Article</title>
          <link>https://example.com/article</link>
        </item>
      </channel></rss>
      ''';

      final articles = await _fetch(xml, type: SourceType.nationalOutlet);

      expect(articles, hasLength(1));
      expect(articles.single.sourceType, SourceType.nationalOutlet);
    });

    test('stamps each Atom entry with the source\'s type', () async {
      const xml = '''
      <feed xmlns="http://www.w3.org/2005/Atom">
        <entry>
          <title>Entry</title>
          <link href="https://example.com/entry"/>
        </entry>
      </feed>
      ''';

      final articles = await _fetch(xml, type: SourceType.specialtyPublication);

      expect(articles, hasLength(1));
      expect(articles.single.sourceType, SourceType.specialtyPublication);
    });
  });
```

- [x] **Step 12: Run the tests to verify they fail**

Run: `flutter test test/rss_remote_data_source_test.dart`
Expected: FAIL — `Article` mapping doesn't set `sourceType` yet, so it's
still `custom` regardless of what's passed.

- [x] **Step 13: Wire `source.type` into both mappers**

In `lib/features/feed/data/datasources/rss_remote_data_source.dart`, add
one line to each `Article(...)` call (after `language: source.language,`):

`_mapItem` (currently ends at line 78):

```dart
      language: source.language,
      sourceType: source.type,
    );
```

`_mapAtomItem` (currently ends at line 102):

```dart
      language: source.language,
      sourceType: source.type,
    );
```

- [x] **Step 14: Run the tests to verify they pass**

Run: `flutter test test/rss_remote_data_source_test.dart`
Expected: PASS (existing tests + 2 new ones).

- [x] **Step 15: Run the full suite so far**

Run: `flutter test`
Expected: PASS — everything through Task 1.

- [x] **Step 16: Commit**

```bash
git add lib/core/constants/interest_categories.dart lib/features/feed/domain/entities/article.dart lib/features/feed/data/datasources/rss_remote_data_source.dart test/interest_categories_test.dart test/article_entity_test.dart test/rss_remote_data_source_test.dart test/screens_smoke_test.dart test/article_repository_test.dart test/feed_providers_test.dart test/article_grid_layout_test.dart
git commit -m "Thread sourceType through Article and the RSS/Atom mappers"
```

---

### Task 2: `SourceTypeBadge` widget + `ArticleCard` overlay

**Files:**
- Create: `lib/core/widgets/source_type_badge.dart`
- Modify: `lib/features/feed/presentation/widgets/article_card.dart`
- Create: `test/source_type_badge_test.dart`
- Modify: `test/article_grid_layout_test.dart`

**Interfaces:**
- Consumes: `SourceType.badgeLabel`/`longLabel` from Task 1.
- Produces: `class SourceTypeBadge extends StatelessWidget` (`{required SourceType type, required SourceTypeBadgeVariant variant}`); `enum SourceTypeBadgeVariant { overlay, inline }`. Consumed here by `ArticleCard` (`overlay`) and by Task 3's `ArticleDetailScreen` (`inline`).

**Surface:** ui

- [x] **Step 1: Write the failing widget tests**

Create `test/source_type_badge_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:newsfeed/core/constants/interest_categories.dart';
import 'package:newsfeed/core/widgets/source_type_badge.dart';

Widget _wrap(Widget child) => MaterialApp(home: Scaffold(body: child));

void main() {
  group('SourceTypeBadge', () {
    testWidgets('renders the short label for a non-custom overlay badge', (tester) async {
      await tester.pumpWidget(_wrap(const SourceTypeBadge(
        type: SourceType.nationalOutlet,
        variant: SourceTypeBadgeVariant.overlay,
      )));

      expect(find.text(SourceType.nationalOutlet.label.toUpperCase()), findsOneWidget);
    });

    testWidgets('renders the long label for a non-custom inline badge', (tester) async {
      await tester.pumpWidget(_wrap(const SourceTypeBadge(
        type: SourceType.specialtyPublication,
        variant: SourceTypeBadgeVariant.inline,
      )));

      expect(find.text(SourceType.specialtyPublication.longLabel), findsOneWidget);
    });

    testWidgets('renders nothing for a custom source, either variant', (tester) async {
      await tester.pumpWidget(_wrap(const Column(children: [
        SourceTypeBadge(type: SourceType.custom, variant: SourceTypeBadgeVariant.overlay),
        SourceTypeBadge(type: SourceType.custom, variant: SourceTypeBadgeVariant.inline),
      ])));

      expect(find.byType(Text), findsNothing);
      expect(tester.takeException(), isNull);
    });
  });
}
```

- [x] **Step 2: Run the tests to verify they fail**

Run: `flutter test test/source_type_badge_test.dart`
Expected: FAIL — `lib/core/widgets/source_type_badge.dart` doesn't exist yet.

- [x] **Step 3: Implement `SourceTypeBadge`**

Create `lib/core/widgets/source_type_badge.dart`:

```dart
import 'package:flutter/material.dart';

import '../constants/interest_categories.dart';

/// Which surface a [SourceTypeBadge] renders on — the two styles are
/// deliberately different weights, not just different colors, so an
/// [overlay] badge (photo, competing with the existing category pill)
/// reads as visually subordinate, while an [inline] badge (plain
/// background, more room) can afford to be the full-word [SourceType.longLabel].
enum SourceTypeBadgeVariant { overlay, inline }

/// Small label naming a source's outlet type (wire/national/international/
/// regional/specialty — see [SourceType]). Renders nothing for
/// [SourceType.custom] (or any value whose [SourceType.badgeLabel] is
/// null), so callers never need their own custom-type check.
class SourceTypeBadge extends StatelessWidget {
  const SourceTypeBadge({super.key, required this.type, required this.variant});

  final SourceType type;
  final SourceTypeBadgeVariant variant;

  @override
  Widget build(BuildContext context) {
    if (type.badgeLabel == null) return const SizedBox.shrink();

    final pill = switch (variant) {
      SourceTypeBadgeVariant.overlay => _overlayPill(),
      SourceTypeBadgeVariant.inline => _inlinePill(context),
    };

    // Tooltip alone only sets Semantics(tooltip:), which screen readers
    // announce inconsistently and would otherwise read the abbreviated
    // overlay label rather than the full one — so the explicit outer
    // Semantics carries the real accessibility label, and the inner
    // Tooltip's own semantics are excluded to avoid a duplicate/garbled
    // announcement.
    return Semantics(
      label: 'Outlet type: ${type.longLabel}',
      child: ExcludeSemantics(
        child: Tooltip(
          message: 'Outlet type: ${type.longLabel}',
          child: pill,
        ),
      ),
    );
  }

  Widget _overlayPill() => Container(
        padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
        decoration: BoxDecoration(
          // Alpha 0.55, not the 0.35 the card's other overlays used to use
          // — see the matching bump in ArticleCard's _CategoryBadge/
          // _BookmarkButton, needed to clear WCAG AA contrast (measured
          // 4.76:1 at 0.55 vs. 2.44:1 at 0.35 over a bright image).
          color: Colors.black.withValues(alpha: 0.55),
          borderRadius: BorderRadius.circular(20),
        ),
        child: Text(
          type.badgeLabel!.toUpperCase(),
          maxLines: 1,
          overflow: TextOverflow.ellipsis,
          softWrap: false,
          style: const TextStyle(
            color: Colors.white,
            fontSize: 10,
            fontWeight: FontWeight.w600,
            letterSpacing: 0.5,
          ),
        ),
      );

  Widget _inlinePill(BuildContext context) {
    final theme = Theme.of(context);
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      decoration: BoxDecoration(
        color: theme.chipTheme.backgroundColor ??
            theme.colorScheme.surfaceContainerHighest,
        borderRadius: BorderRadius.circular(20),
      ),
      child: Text(
        type.longLabel,
        style: theme.textTheme.labelSmall?.copyWith(
          color: theme.colorScheme.onSurfaceVariant,
          fontWeight: FontWeight.w600,
        ),
      ),
    );
  }
}
```

- [x] **Step 4: Run the tests to verify they pass**

Run: `flutter test test/source_type_badge_test.dart`
Expected: PASS (3 tests).

- [x] **Step 5: Wire the overlay variant into `ArticleCard`, and bump the existing overlay scrims**

In `lib/features/feed/presentation/widgets/article_card.dart`:

Add the import:

```dart
import '../../../../core/widgets/source_type_badge.dart';
```

In `build`'s `Stack` (after the `if (showCategory) Positioned(...)` block,
i.e. after what's currently line 71):

```dart
                  Positioned(
                    left: 10,
                    right: 10,
                    bottom: 10,
                    child: Align(
                      alignment: Alignment.centerLeft,
                      child: SourceTypeBadge(
                        type: article.sourceType,
                        variant: SourceTypeBadgeVariant.overlay,
                      ),
                    ),
                  ),
```

Bump the two existing overlay scrims from `0.35` to `0.55` (accessibility
fix bundled with this change — see spec):

- `_BookmarkButton` (currently `article_card.dart:196`):
  `color: Colors.black.withValues(alpha: 0.35),` → `alpha: 0.55`.
- `_CategoryBadge` (currently `article_card.dart:222`):
  `color: Colors.black.withValues(alpha: 0.35),` → `alpha: 0.55`.

- [x] **Step 6: Update the grid layout test's worst-case fixture**

`test/article_grid_layout_test.dart`'s `_worstCaseArticle` already got
`sourceType: SourceType.internationalOutlet` in Task 1, Step 9 — this
means the existing 4 width-based no-overflow tests in this file now
exercise the new overlay badge with its longest string. No further edit
needed here beyond confirming the import from Task 1 covers
`SourceType` (it does — `article_grid_layout_test.dart` already imports
`Article`, and Task 1 added the `interest_categories.dart` import
alongside it).

- [x] **Step 7: Run the tests to verify they pass**

Run: `flutter test test/article_grid_layout_test.dart`
Expected: PASS — no overflow at any of the 4 tested widths with the
overlay badge present.

- [x] **Step 8: Run the full suite so far**

Run: `flutter test`
Expected: PASS — everything through Task 2.

- [x] **Step 9: Commit**

```bash
git add lib/core/widgets/source_type_badge.dart lib/features/feed/presentation/widgets/article_card.dart test/source_type_badge_test.dart
git commit -m "Add SourceTypeBadge and render it on ArticleCard"
```

---

### Task 3: Inline badge on `ArticleDetailScreen`

**Files:**
- Modify: `lib/features/feed/presentation/screens/article_detail_screen.dart`
- Modify (or extend): `test/screens_smoke_test.dart`

**Interfaces:**
- Consumes: `SourceTypeBadge`/`SourceTypeBadgeVariant.inline` from Task 2.

**Surface:** ui

- [x] **Step 1: Write the failing widget test**

`test/screens_smoke_test.dart` already has the `article` fixture (updated
in Task 1, Step 9) pumping `ArticleDetailScreen` somewhere in its existing
smoke tests — read that section first to match its existing pump/wrap
helper, then add a case (or extend an existing "renders article detail"
test) asserting the inline badge's long label appears when the fixture's
`sourceType` is non-custom, and does not appear when it's `custom`:

```dart
    testWidgets('shows the outlet-type badge on the detail screen', (tester) async {
      // Reuse this file's existing pump helper for ArticleDetailScreen,
      // with `article` (or a copy of it) using sourceType: SourceType.nationalOutlet.
      // ... pump ...
      expect(find.text(SourceType.nationalOutlet.longLabel), findsOneWidget);
    });

    testWidgets('shows no outlet-type badge for a custom source', (tester) async {
      // Same pump, sourceType: SourceType.custom (the fixture's existing default).
      // ... pump ...
      expect(find.text(SourceType.custom.longLabel), findsNothing);
    });
```

(Exact wrapping/pump code should match whatever helper this file already
uses to render `ArticleDetailScreen` in its existing tests — do not
duplicate a second ad hoc harness.)

- [x] **Step 2: Run the tests to verify they fail**

Run: `flutter test test/screens_smoke_test.dart`
Expected: FAIL — the detail screen doesn't render an outlet badge yet.

- [x] **Step 3: Replace the category-pill block with a `Wrap`**

In `lib/features/feed/presentation/screens/article_detail_screen.dart`,
add the import:

```dart
import '../../../../core/widgets/source_type_badge.dart';
```

Replace the existing block (currently lines 163–182 — the
`if (category != null) Container(...)` pill immediately followed by
`const SizedBox(height: 16),`) with:

```dart
                if (category != null || article.sourceType.badgeLabel != null) ...[
                  Wrap(
                    spacing: 8,
                    runSpacing: 8,
                    children: [
                      if (category != null)
                        Container(
                          padding: const EdgeInsets.symmetric(
                            horizontal: 12,
                            vertical: 6,
                          ),
                          decoration: BoxDecoration(
                            gradient: LinearGradient(colors: category.gradient),
                            borderRadius: BorderRadius.circular(20),
                          ),
                          child: Text(
                            category.label,
                            style: const TextStyle(
                              color: Colors.white,
                              fontWeight: FontWeight.w700,
                              fontSize: 12,
                            ),
                          ),
                        ),
                      if (article.sourceType.badgeLabel != null)
                        SourceTypeBadge(
                          type: article.sourceType,
                          variant: SourceTypeBadgeVariant.inline,
                        ),
                    ],
                  ),
                  const SizedBox(height: 16),
                ],
```

This also fixes a small pre-existing bug: the old code emitted the
`SizedBox(height: 16)` unconditionally, leaving a dangling gap when
`category` was null and nothing else preceded it; it's now conditional on
there being at least one pill to separate from the headline below.

- [x] **Step 4: Run the tests to verify they pass**

Run: `flutter test test/screens_smoke_test.dart`
Expected: PASS.

- [x] **Step 5: Run the full suite**

Run: `flutter test`
Expected: PASS — every existing test plus all new ones from Tasks 1–3.

- [x] **Step 6: Run static analysis**

Run: `flutter analyze`
Expected: no new issues.

- [x] **Step 7: Commit**

```bash
git add lib/features/feed/presentation/screens/article_detail_screen.dart test/screens_smoke_test.dart
git commit -m "Show the outlet-type badge on ArticleDetailScreen"
```

---

## Self-Review Notes

- **Spec coverage:** `SourceType` helpers + `Article.sourceType` +
  RSS/Atom mapping (Task 1), the shared badge widget + card overlay +
  bundled contrast fix (Task 2), detail-screen inline badge + the
  dangling-`SizedBox` fix (Task 3) — every spec section (Data flow, UI,
  Loading/error/empty states, Testing) maps to a task. Out-of-scope items
  from the spec (Manage Sources unification, `RelatedArticleTile`,
  legacy-bookmark catalog lookup, byline overflow fix, per-type color)
  are intentionally not tasked here.
- **Placeholder scan:** no TBD/TODO markers; every step carries literal
  code or an exact command, except Task 3 Step 1's test, which
  deliberately defers to this file's own existing pump helper rather than
  inventing a parallel one — flagged explicitly rather than left implicit.
- **Type consistency:** `Article.sourceType` is `SourceType` (Task 1),
  consumed as `SourceType` by `SourceTypeBadge.type` (Task 2) and read
  via `.badgeLabel`/`.longLabel` (both `String?`/`String`, Task 1) in
  both `ArticleCard` (Task 2) and `ArticleDetailScreen` (Task 3) — no
  nullable/non-nullable mismatch at any call site.
- **New-required-field blast radius:** verified via
  `grep -rn "Article(" lib/ test/` before writing this plan — exactly 6
  construction sites (2 production in `rss_remote_data_source.dart`, the
  entity's own `fromJson`, and 4 test fixtures), all accounted for in
  Task 1.
