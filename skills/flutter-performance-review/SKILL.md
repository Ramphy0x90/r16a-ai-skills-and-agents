---
name: flutter-performance-review
description: Use before calling any Flutter screen/widget work done, or when asked to review or optimize Flutter UI performance. Checks for unnecessary rebuilds, missing const, inefficient list rendering, unbounded image/asset costs, and other common Flutter performance mistakes.
---

# Flutter performance review

Run this as a self-review pass on any Flutter code you just wrote or are asked to review — not just when something is visibly slow. Catching these at write-time is cheaper than profiling later.

## Rebuild minimization

- **`const` everywhere it's valid.** Any widget constructor whose arguments are all compile-time constants should be `const`. This is the single highest-value, lowest-effort check — scan every new widget instantiation for a missing `const`.
- **Don't put a `StatefulWidget`'s whole subtree in one `build()` if only a small part changes.** If a `setState` call only affects one piece of UI, extract that piece into its own (ideally `const`-constructible, or separately-keyed) widget so the rest of the subtree isn't rebuilt. A classic smell: a `TextField`'s `onChanged` triggering `setState` that rebuilds an entire large `Column` including static header/footer content that never changes.
- **Don't create closures/callbacks inline inside `build()` when they could be hoisted**, if doing so would let a child widget be `const` or avoid rebuilding — but don't over-engineer this for trivial cases; weigh against readability.
- **Avoid rebuilding on every frame from an unthrottled stream/animation listener** without `AnimatedBuilder`/`ValueListenableBuilder` scoping the rebuild to just the subtree that needs it.

## Lists

- **Any list of unknown/large/unbounded length must use `ListView.builder` (or `SliverList`/`CustomScrollView` with builders), never `ListView(children: list.map(...).toList())`.** The `.map().toList()` pattern eagerly builds every item up front regardless of what's visible — this is the most common Flutter performance bug in list-heavy screens.
- Give list items a stable `key` when the list can reorder, insert, or delete, so Flutter can diff correctly instead of rebuilding/relaying out everything.
- For long/infinite lists backed by a paginated API, implement pagination (load-more on scroll-near-end) rather than fetching and rendering everything at once.

## Images and assets

- Specify explicit `width`/`height`/`cacheWidth`/`cacheHeight` on `Image` widgets displaying remote or large images, so Flutter decodes at the needed resolution instead of full source resolution — this matters a lot for memory, not just load time.
- Use `Image.asset`/`Image.network` with appropriate caching; for remote images, prefer a caching package (e.g. `cached_network_image`) over raw `Image.network` if the project already depends on one — don't add a new dependency for this alone without checking what's already available.
- SVGs via `flutter_svg`: avoid re-parsing the same SVG string repeatedly in a rebuilt widget — if a `colorFilter`/theme-tinted icon rebuilds often, verify the SVG isn't being reloaded from asset each time unnecessarily (it usually is cached by the package, but check if a custom loader is involved).

## Async & state

- Any `Future`/`Stream` subscription created in a widget must be properly disposed (`StreamSubscription.cancel()`, `AnimationController.dispose()`, `TextEditingController.dispose()`, etc.) in `dispose()` — a leaked subscription/controller is both a memory leak and a source of "setState called after dispose" crashes.
- Guard every `setState` that follows an `await` with `if (mounted)`.
- Don't run heavy synchronous computation (parsing, sorting large lists, image processing) directly on the UI isolate inside `build()` or a frame callback — move it to `compute()`/an isolate, or precompute outside the build phase.

## Layout

- Avoid deeply nested `Container`/`Padding`/`Align` chains where a single `Container` with multiple properties, or a `Padding` + a layout widget, would do the same job with fewer widget-tree layers — excess nesting costs layout/paint time at scale, though this matters far less than the rebuild/list issues above; don't over-index on it for small trees.
- Avoid `Opacity`/`ClipRRect` wrapping large, frequently-rebuilt subtrees when a cheaper equivalent exists (e.g. a pre-clipped asset, or `RepaintBoundary` isolating the expensive part) — these force an offscreen render pass.
- Use `RepaintBoundary` around a frequently-animating subtree (e.g. a custom animation, a chart) so it doesn't force repaints of unrelated siblings.

## How to run this review

1. Re-read every widget file touched in the current task.
2. Check each item above against that file — don't just skim for "does it look fine."
3. For any finding, fix it inline rather than noting it as a TODO, unless the fix would meaningfully change the scope of the task (e.g. introducing a new caching dependency) — in that case, flag it explicitly rather than silently skipping it.
4. If nothing applies (e.g. a tiny static widget with no lists, images, or async), say so briefly rather than forcing a finding.
