---
name: flutter-feature-scaffold
description: Use when adding a new feature, screen or widget to a feature-oriented Flutter app (lib/features/<name>/). Scaffolds files matching the project's state management with a strict smart/dumb split, so business logic never lands in a widget.
---

# Flutter feature scaffold

## Step 1: defer to the project's own structure

Read the project's `CLAUDE.md` (or equivalent project doc) and look at one or two existing feature folders before creating anything. If they establish a structure, naming scheme, or state-management pattern, follow it — everything below is the default only when the project hasn't decided.

Default structure when nothing else is established:

```
lib/
  core/              # shared, cross-feature: router, routes, app shell, etc.
  features/
    <feature_name>/
      screens/
      widgets/
```

## Step 2: detect state management

Check `pubspec.yaml` and existing screens:

- **Riverpod** (`flutter_riverpod`/`hooks_riverpod`), **Bloc** (`flutter_bloc`), **Provider** (`provider`) → the smart layer is the screen **plus its providers/notifiers/blocs**. Put those where existing features put them (e.g. `features/<name>/providers/` or `bloc/`).
- **None** → the smart layer is a `StatefulWidget` screen using `setState`.

Don't introduce a state-management library as part of a scaffold task unless explicitly asked.

## The hard rule: smart vs. dumb

This is a correctness requirement, not a style preference (same discipline as Angular container/presentational components).

**Smart** = the screen, together with its providers/notifiers/blocs if the project uses them:
- Owns all mutable business state and loading/error state.
- Owns all async work and service calls. "Service" means anything outside the widget tree: API clients, repositories, local storage, auth, platform channels, and — in apps with a native core — FFI bridge calls (e.g. flutter_rust_bridge bindings under `lib/src/rust/`). The FFI bridge is one kind of service, not a special case.
  - With Riverpod/Bloc: async work and service calls live in the notifier/bloc, not in the screen's `build()` or callbacks. The screen `watch`es state and dispatches intents (`ref.read(xProvider.notifier).submit(...)`, `context.read<XBloc>().add(...)`).
  - With `setState`: the screen's `State` does it directly.
- Owns navigation (`context.go(...)`, `Navigator.push`, etc.) — navigation is triggered from the screen, e.g. via `ref.listen`/`BlocListener` reacting to a state change, not from inside a notifier that holds a `BuildContext`.
- Composes widgets from `widgets/`, passing data and callbacks down.

**Dumb** = everything in `widgets/`:
- Takes everything it needs as constructor parameters: data to display, and callbacks (`onSubmit`, `onTap`, `onChanged`, …) for anything that happens.
- Never calls a service, repository, API client, or FFI bridge, and never reads providers/blocs (`ref.watch`, `context.read<…>()`, `BlocBuilder`) — it gets the data as a parameter instead.
- Never owns state beyond trivial visual state (an `AnimationController`, a `TextEditingController` whose value the parent reads on submit, a hover/focus flag). Test: does any other part of the app need to know about it? If yes, it belongs in the smart layer.
- `StatelessWidget` by default; `StatefulWidget` only for that narrow visual-state case.

**Before writing any widget file, ask: does this need to call a service or read app state, or just render data and report user actions upward?** If the former, that logic goes in the smart layer and the widget gets a parameter or callback.

## Procedure

1. **Confirm the feature name** (snake_case, e.g. `chat`, `settings`, `user_profile`) and which screen(s)/widget(s) are actually needed — don't scaffold placeholder screens nobody asked for.

2. **Create the folders** that are needed now: `screens/`, `widgets/`, and — only if the project's state management requires a separate file for the first real provider/notifier/bloc — the provider/bloc folder existing features use.

3. **Create the smart layer**:
   - Screen at `lib/features/<feature_name>/screens/<screen_name>_screen.dart`, named `<ScreenName>Screen`, matching what other screens are (`StatefulWidget`, `ConsumerWidget`, a widget wrapping `BlocProvider`, …).
   - **With Riverpod/Bloc**: the notifier/bloc owns the async call and exposes loading/error/data as state (e.g. `AsyncValue<T>`, or a sealed state class). Catch errors in the notifier/bloc and surface them as state; the screen maps state to widgets.
   - **With `setState`**: the screen owns `_isLoading` / `_errorMessage` (or equivalent), wraps async work in `try { … } catch (e) { … } finally { … }`, and guards every `setState` after an `await` with `if (mounted)`.
   - Dispose any controllers/subscriptions the smart layer owns (`dispose()`, `ref.onDispose`, `Bloc.close`).

4. **Create the widget(s)** at `lib/features/<feature_name>/widgets/<widget_name>.dart`:
   - `StatelessWidget` named `<WidgetName>` (no "Widget" suffix unless it clarifies).
   - `required` data fields plus `VoidCallback`/`ValueChanged<T>` callbacks for anything interactive.
   - No imports of services, repositories, API clients, providers/blocs, state-management packages, or generated bridge bindings — any of those in a widget file means logic leaked into the wrong layer.

5. **Wire them together** in the screen's `build()` — pass data and callbacks down; callbacks dispatch to the notifier/bloc (or run the async call in `State`).

6. **Self-check before finishing** — re-read every new widget file and confirm:
   - It imports no service/repository/bridge/provider/bloc code.
   - It has no `async` method that does real work (beyond e.g. a local `Future.delayed` for animation).
   - Every interactive behavior is a constructor callback, not a method that reaches outside the widget.

   If any check fails, move the offending code to the smart layer and re-wire via a callback — don't leave it "just this once."

7. **Routing**: if the project uses `go_router` with a `Routes` class of path constants (check `lib/core/routes.dart` or similar first), add the new screen's route there and to the router config — don't hardcode path strings inline.

## Don'ts

- Don't create `models/`, `services/`, `state/`, or `providers/` subfolders "for later" — only add structure when a concrete file needs it now (the first real provider counts; an empty folder doesn't).
- Don't promote a widget to a shared/`core/` location because it seems reusable in theory — only once a second feature actually uses it.
