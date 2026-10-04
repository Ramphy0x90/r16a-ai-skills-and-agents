---
name: angular-best-practices
description: Use when writing or reviewing Angular code (components, services, directives, pipes, routing, forms). Covers component architecture, change detection and performance, RxJS/subscription hygiene, dependency injection, forms, and routing conventions, with attention to Angular version-gated features.
---

# Angular best practices

## Check the Angular version first

Before applying any version-gated recommendation below, check `package.json` for the installed `@angular/core` version. Don't suggest or write code using a feature the project's version doesn't support:
- Standalone components (stable): v14+
- Typed Reactive Forms: v14+
- Functional route guards/resolvers (`CanActivateFn`, etc.): v14.1+
- Signals: v16+ (stable primitives), with `input()`/`output()`/`model()` signal-based APIs from v17+
- New built-in control flow (`@if`/`@for`/`@switch`): v17+
- `inject()` function: v14+, usable outside DI context broadly from there on

If the project is on an older version (e.g. Angular 12-15, NgModule-based), use NgModules, constructor injection, and `*ngIf`/`*ngFor` idiomatically rather than writing standalone/signal-based code that won't compile.

## Component architecture: smart/container vs presentational

Same discipline as container/presentational component separation elsewhere:

- **Container (smart) components**: fetch data (via services), hold state, handle routing, pass data down to presentational components via `@Input()`, handle events bubbled up via `@Output()`.
- **Presentational (dumb) components**: receive data via `@Input()`, emit events via `@Output()`, contain no service injection beyond what's needed for pure display logic (e.g. a pipe-like formatting helper), and are easy to unit test in isolation.
- Don't inject `HttpClient` or a data-fetching service directly into a component meant to be purely presentational — that's a smart/dumb violation.

## Change detection & performance

- **Default new components to `ChangeDetectionStrategy.OnPush`** unless there's a specific reason not to (e.g. a component that genuinely needs default dirty-checking for a legacy reason). OnPush plus immutable data flow (new object/array references on change, not in-place mutation) is the standard high-performance pattern.
- **Always use `trackBy` with `*ngFor`** (or the equivalent tracking expression in `@for` on v17+) over any list that can reorder, filter, or have items added/removed — without it, Angular destroys and recreates DOM nodes unnecessarily on every change.
- Don't call a function directly in a template expression (`{{ someFunction() }}` or `[prop]="someFunction()"`) if that function does real work — it re-runs on every change detection cycle. Use a pipe (ideally a pure pipe) or precompute the value and bind to a property instead.
- Avoid deeply nested `*ngIf`/structural directive chains where a single computed boolean/observable would do — both for readability and to avoid redundant re-evaluation.

## RxJS & subscription hygiene

- **Prefer the `async` pipe in templates over manual `.subscribe()` in the component class** wherever the value just needs to be rendered — it handles subscription and unsubscription automatically and integrates with change detection.
- **Every manual `.subscribe()` must be unsubscribed** — use `takeUntilDestroyed()` (v16+, via `DestroyRef`) or a traditional `takeUntil(this.destroy$)` pattern with `ngOnDestroy` on older versions. An un-unsubscribed subscription in a component that gets created/destroyed repeatedly (e.g. per route navigation) is a classic Angular memory leak.
- Avoid nested `.subscribe()` calls (subscribing inside a subscribe callback) — use RxJS operators (`switchMap`, `mergeMap`, `concatMap`, `combineLatest`, etc.) to compose streams instead. Nested subscribes are a strong readability and bug-risk smell (easy to leak the inner subscription).
- Pick the right flattening operator deliberately: `switchMap` when a new value should cancel the previous in-flight request (e.g. search-as-you-type), `concatMap` when order and completion matter (e.g. sequential writes), `mergeMap` when concurrent and unordered is fine, `exhaustMap` when in-flight requests should block new ones (e.g. a submit button).
- Don't use `any` to silence an RxJS typing issue — fix the actual type flow; RxJS typing issues are almost always a sign the operator chain's shape doesn't match what's intended.

## Dependency injection

- Use constructor injection (or `inject()` on v14+, which is also fine and often cleaner for optional/conditional injection) — don't mix styles inconsistently within the same codebase; follow whatever the project already does predominantly.
- Default to `providedIn: 'root'` for singleton services. Only provide a service at the component level when it's deliberately meant to be scoped per-component-instance (e.g. a service that holds state for one instance of a reusable widget).
- Avoid injecting `Injector` and manually resolving dependencies as a workaround for a DI design problem — restructure the dependency graph instead.

## Forms

- Prefer Reactive Forms (`FormGroup`/`FormControl`/`FormBuilder`) over template-driven forms for anything beyond a trivial single-field form — reactive forms give better type safety, testability, and validation composition.
- On v14+, use typed Reactive Forms (`FormGroup<{...}>`) rather than the untyped legacy API, for actual compile-time safety on form values.
- Keep validation logic in validators (built-in or custom `ValidatorFn`/`AsyncValidatorFn`), not scattered as ad-hoc checks in the component class.

## Routing

- Lazy-load feature areas (`loadChildren` for NgModules, or `loadComponent`/standalone route lazy loading on v14+) rather than putting everything in the eagerly-loaded main bundle — this is a real, measurable startup-performance factor, not a micro-optimization.
- Use route guards (functional guards on v14.1+, class-based on older versions) for auth/permission checks rather than checking auth state ad hoc inside components.
- Keep route-resolved data (via a resolver) for data a route genuinely can't render without, but don't over-use resolvers for data that could just as well be fetched inside the component with a loading state — resolvers block navigation until they complete.

## Testing

- Unit-test presentational components by asserting on rendered output / emitted events given inputs, not by reaching into private implementation details.
- Mock `HttpClient` via `HttpClientTestingModule`/`HttpTestingController` rather than mocking the service that wraps it, when the goal is to verify the actual HTTP call shape.
- Keep `TestBed` configuration minimal — importing the whole app module/unrelated providers into a unit test slows the suite and hides what's actually being tested.

## How to run this review

1. Check the Angular version before applying anything version-gated.
2. Walk the sections above against the actual component/service files touched.
3. Prioritize: subscription leaks and missing `OnPush`/`trackBy` on large lists are the highest-impact, most common findings — check those first.
4. Fix inline where straightforward; flag anything that would require a broader refactor (e.g. migrating an entire module to standalone components) rather than doing it unprompted.
