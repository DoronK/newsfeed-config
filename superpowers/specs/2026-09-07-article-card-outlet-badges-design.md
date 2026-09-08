# Outlet badges on article cards

## Purpose

FEATURE_ROADMAP.md item #6. The outlet-type badge (wire/national/
international/regional/specialty — see the `SourceType` enum in
`lib/core/constants/interest_categories.dart`) already ships in Settings →
Manage Sources, but `Article` only carries a `sourceName` string, so the
badge never reaches the feed or the article-detail screen — the two places
people actually read. This closes that gap: thread `sourceType` through
`Article`, and surface it on `ArticleCard` and `ArticleDetailScreen`.

Deliberately unchanged from #3's shipped scope: outlet type only, never a
political-lean judgment (see the doc comment on `SourceType`).

## Requirements (from product-manager + architect + designer passes)

- `Article` gains a `sourceType` field, populated from the `NewsSource`
  already in hand at both RSS/Atom mapping call sites.
- The field round-trips through disk-cache JSON (`Article.toJson`/
  `fromJson`), with a safe "no badge" fallback — never a wrong or blank
  badge shown as if real — for articles cached before this field existed.
- The badge is visible on `ArticleCard` (feed grid — For You, Category,
  Search, Saved all reuse this widget) and `ArticleDetailScreen`.
- No badge for `SourceType.custom` (user-added feeds) or for any article
  whose resolved type is unresolvable — matching the existing
  "never shown with a badge" rule already on the enum.
- No change to `RssRemoteDataSource`'s network/fetch behavior,
  `LocalStorageService`'s cache key/schema shape, or any other feature
  area.

## Corrections from the design passes, recorded so they aren't re-litigated

- **No reusable badge widget exists yet.** `ManageSourcesScreen` renders a
  plain `Text(source.type.label)` as a `SwitchListTile` subtitle, not a
  pill. This work builds the first one, at
  `lib/core/widgets/source_type_badge.dart`. `ManageSourcesScreen` is left
  on its own `Text` for now — unifying it is out of scope (see below).
- **`Article` has 11 fields, not 14** (the product-manager pass's count was
  off; verified directly against `article.dart`).
- **The feed grid's metadata row can't take a third element.** Fixed
  `childAspectRatio` (`ArticleGridDelegate`) leaves as little as ~190px for
  `sourceName · timeAgo` on a 2-column layout — a third element there
  would ellipsize the publisher name, the more important string. The card
  badge goes on the image overlay instead (same idiom as the existing
  `_CategoryBadge`), costing zero row space.
- **Data reality check**: across the built-in catalog, type is
  `national` for 44/74 sources, and 5 of 12 category feeds
  (health/world/politics/gaming/finance) are 100% one type — in those
  feeds the badge is constant, not discriminating. Recorded as an honest
  limitation of this feature, not a reason to change scope: absence must
  still mean exactly one thing (custom/unknown), so type is never
  suppressed just because a feed happens to be uniform.

## Data flow

### `SourceType` (`lib/core/constants/interest_categories.dart`)

Add two members and one static helper, next to the existing `label`
getter:

- `String get longLabel` — full-word form ("National outlet" vs the
  existing `label`'s "National") for the detail screen and accessibility
  strings, where a bare short label risks reading as a second category
  pill next to the topic gradient chip.
- `String? get badgeLabel` — `null` for `custom`, else the same text as
  `label`. Centralizes the "never badge custom" rule in the enum (already
  documented there) instead of duplicating an `if (type == custom)` check
  in every widget that renders a badge.
- `static SourceType fromName(String? name)` — tolerant parser: an
  unrecognized or missing name (predates this field, or an
  unrecognized/future value from the remote catalog) falls back to
  `SourceType.custom`, never throws. Replaces the `firstWhere/orElse`
  block already written once inline in `NewsSource.fromJson` — refactor
  that call site onto this helper so the fallback logic exists in exactly
  one place before `Article.fromJson` needs the identical logic.

### `Article` (`lib/features/feed/domain/entities/article.dart`)

- New `required SourceType sourceType` field (required, not defaulted —
  matches the entity's existing all-required constructor and forces every
  construction site, present and future, to state intent rather than
  silently producing a badge-less article).
- `toJson`: `'sourceType': sourceType.name`.
- `fromJson`: `sourceType: SourceType.fromName(json['sourceType'] as String?)`
  — `null`/missing/unrecognized all resolve to `custom` (no badge), the
  same fallback shape already used for `sourceUrl`/`language` in this
  factory.
- New import: `'../../../../core/constants/interest_categories.dart' show SourceType;`
  — same `show`-clause convention already used by
  `article_repository.dart:1` to keep the domain layer's dependency on
  `core/constants` explicit and narrow.

### `RssRemoteDataSource` (`lib/features/feed/data/datasources/rss_remote_data_source.dart`)

Both `_mapItem` and `_mapAtomItem` already receive the full `NewsSource
source` parameter. Add one line to each `Article(...)` call:
`sourceType: source.type,`.

## UI

### Shared widget: `lib/core/widgets/source_type_badge.dart` (new)

One widget, two variants, so the "never badge custom" check and the
tooltip/accessibility wiring exist once:

```dart
enum SourceTypeBadgeVariant { overlay, inline }

class SourceTypeBadge extends StatelessWidget {
  const SourceTypeBadge({super.key, required this.type, required this.variant});
  final SourceType type;
  final SourceTypeBadgeVariant variant;
  // build(): type.badgeLabel == null -> SizedBox.shrink(); else render the
  // variant's pill, wrapped in Tooltip + Semantics (see Accessibility).
}
```

- **`overlay` variant** (`ArticleCard`): bottom-left `Positioned` pill
  inside the card's existing image `Stack`
  (`left: 10, right: 10, bottom: 10`, `Align(alignment: centerLeft)` so
  the pill shrink-wraps instead of stretching). Same scrim treatment as
  the existing `_CategoryBadge`/`_BookmarkButton` overlays
  (`Colors.black` scrim, white text) but visually subordinate to the
  category pill so two overlay elements on one image don't read as
  competing peers: **uppercase, 10px, w600, letter-spacing 0.5, tighter
  8/4 padding** vs. the category pill's title-case 11px/w700/10-5 padding.
  This mirrors the app's own existing idiom for quiet structural labels —
  `ManageSourcesScreen`'s section headers are already
  `title.toUpperCase()` at `letterSpacing: 0.6`,
  `onSurfaceVariant` — applied here in white-on-scrim form since this
  sits over a photo, not a plain surface.
- **`inline` variant** (`ArticleDetailScreen`): `theme.chipTheme.backgroundColor`
  fill (falls back to `colorScheme.surfaceContainerHighest` if that theme
  key is ever unset), `onSurfaceVariant` text at `labelSmall`/w600, 12/6
  padding, `BorderRadius.circular(20)` — matches the existing category
  pill's proportions and the app's real chip fill (measured 8.2:1 light /
  9.6:1 dark contrast). Uses `longLabel`, not `label` — "National" bare
  next to a "Technology" gradient pill reads as a second category;
  "National outlet" doesn't.

**Bundled accessibility fix (same file, same visual system, one line ×2):**
bump the pre-existing `_CategoryBadge` and `_BookmarkButton` overlay scrim
from `Colors.black.withValues(alpha: 0.35)` to `0.55` in `article_card.dart`.
Measured: 0.35 composites to 2.44:1 white-text contrast over a bright
image (fails WCAG AA's 4.5:1); 0.55 composites to 4.76:1 (passes). The new
overlay badge uses 0.55 from the start — leaving the two existing overlays
at 0.35 would put three different scrim weights on one image and look like
a bug, not a design choice. Two-line change, no behavior change beyond
color.

**Accessibility (both variants):**

```dart
Semantics(
  label: 'Outlet type: ${type.longLabel}',
  child: ExcludeSemantics(child: Tooltip(message: 'Outlet type: ${type.longLabel}', child: pill)),
)
```

`Tooltip` alone sets `Semantics(tooltip:)`, which screen readers announce
inconsistently, and would otherwise read the abbreviated overlay label
("NATIONAL") rather than the full one — hence the explicit `Semantics`
wrapper with `ExcludeSemantics` on the inner tooltip. `Tooltip`'s default
`triggerMode` is long-press on touch platforms (already used this way by
`_BookmarkButton`, `article_card.dart:189`); the card's `InkWell` declares
only `onTap`, so there's no gesture conflict. The badge is never wrapped
in its own `onTap` — it's a label, not a control.

### `ArticleCard` (`lib/features/feed/presentation/widgets/article_card.dart`)

Add a third `Positioned` inside the existing `Stack` (`build`, after the
category-badge `Positioned`):

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

Renders on all four screens that reuse `ArticleCard` (For You, Category
feed, Search, Saved) with no per-screen change needed. `RelatedArticleTile`
(a separate, more compact widget) is explicitly excluded — no room, and
it's a browse-more strip, not the primary trust surface.

### `ArticleDetailScreen` (`lib/features/feed/presentation/screens/article_detail_screen.dart`)

Replace the existing `if (category != null) Container(...)` pill +
unconditional `SizedBox(height: 16)` (currently lines 163–182) with a
`Wrap` holding both pills, only when at least one exists — this also
fixes a small pre-existing bug (a dangling 16px gap when `category` is
null and there was nothing above it):

```dart
if (category != null || article.sourceType.badgeLabel != null) ...[
  Wrap(
    spacing: 8,
    runSpacing: 8,
    children: [
      if (category != null)
        Container(/* existing gradient category pill, unchanged */),
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

`Wrap` (not `Row`) so a long category label plus the outlet pill degrades
to two lines at 2x OS text scaling instead of overflowing a ~332px
content width. Category pill first (loud/brand), outlet pill second
(quiet/neutral) — same reading order the card teaches.

Explicitly untouched: the pre-existing overflow-prone
`Row(children: [Text(article.sourceName), ...])` byline below the
headline (`article_detail_screen.dart:185-225`, no `Flexible`) — a
separate, already-existing issue this change doesn't touch or worsen.

## Loading / error / empty states

- **No loading state.** `sourceType` is resolved synchronously at
  RSS-map time from a `NewsSource` already in hand — nothing to wait on,
  no shimmer/skeleton pill, no reserved-but-empty layout slot (a reserved
  slot would be visually indistinguishable from "still loading" on feeds
  that will never have a badge).
- **Three "no badge" cases, all `SizedBox.shrink()`, all correct, not
  bugs:** a user-added custom feed (by policy); an article deserialized
  from a pre-update disk cache (missing key → `custom` via `fromName`);
  an unrecognized/future `type` string from the remote catalog (same
  fallback).
- **One transitional, accepted state:** the first cold start after this
  ships paints the cached feed with no badges until the next network
  refresh replaces those articles. Self-heals in one refresh; no special
  handling.
- **One permanent, accepted state, named so it isn't rediscovered as a
  bug:** bookmarks are persisted as serialized `Article` JSON and never
  refetched (`bookmarks_provider.dart`). Articles bookmarked before this
  ships will never show a badge. Acceptable per this spec's own
  "no badge beats a wrong badge" rule — not fixed here. (A same-session
  design note flagged a possible follow-up — resolving a `custom`
  bookmark's `sourceType` against `sourceUrl` in the live catalog at
  display time — explicitly deferred; see Out of scope.)

## Testing

- **`test/interest_categories_test.dart`** (extended): a new `SourceType`
  group covering `badgeLabel` (null for `custom`, non-null text for every
  other value), `longLabel` (non-empty, distinct from `label`, for every
  value), and `fromName` (round-trips every enum's own `.name`, and falls
  back to `custom` for `null` and an unrecognized string). Also confirm
  `NewsSource.fromJson`'s existing fallback tests (in this same file)
  still pass unchanged after it's refactored onto `fromName`.
- **New `test/article_entity_test.dart`**: `Article.toJson`/`fromJson`
  round-trips `sourceType` for a non-custom value; a JSON map missing the
  `sourceType` key deserializes to `SourceType.custom` (the pre-existing-
  cache case); an unrecognized string value also falls back to `custom`.
- **`test/rss_remote_data_source_test.dart`** (extended): extend the
  shared `_fetch` helper to accept an optional `SourceType type` (default
  matching its current implicit `custom`), and add a test asserting a
  fetched article's `sourceType` matches the `NewsSource.type` passed in,
  for both the RSS and the Atom path.
- **4 existing test fixtures updated** for the new required field (verify
  no other `Article(...)` construction sites exist beyond these before
  starting — `grep -rn "Article(" lib/ test/"`):
  `test/article_grid_layout_test.dart`, `test/feed_providers_test.dart`,
  `test/article_repository_test.dart`, `test/screens_smoke_test.dart`.
- **`test/article_grid_layout_test.dart`**: the existing worst-case
  fixture gets a worst-case `sourceType` too (`internationalOutlet` — the
  longest `label`), so the new overlay pill is included in the existing
  no-overflow check across all four tested widths, rather than silently
  never being exercised there.
- **New widget test** (extend `test/screens_smoke_test.dart` or a new
  small file) asserting: `SourceTypeBadge` renders its label text for a
  non-custom type and renders nothing (`SizedBox.shrink()`, no `Text`
  found) for `SourceType.custom`.
- `flutter analyze && flutter test` both pass at the end.

## Out of scope (named so it's deferred, not forgotten)

- Unifying `ManageSourcesScreen`'s plain `Text(source.type.label)` onto
  the new `SourceTypeBadge` widget.
- Adding the badge to `RelatedArticleTile` or the Android home-screen
  widget.
- Resolving a legacy `custom`-tagged bookmark's real type by matching
  `sourceUrl` against the live catalog at display time (would remove the
  "permanent no-badge" state above, but is a separate, optional
  enhancement with its own trade-offs — not needed to ship this item).
- Fixing the pre-existing unwrapped `Text(article.sourceName)` overflow
  risk in the detail-screen byline row.
- Per-outlet-type color coding — rejected outright: it would make color
  the sole differentiator and risks reading as a credibility rating,
  which is exactly the political-lean framing #3 already deliberately
  rejected.
