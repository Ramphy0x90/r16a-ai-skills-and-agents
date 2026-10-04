---
name: flutter-feature-scaffold
description: Use when adding a new feature to a feature-oriented Flutter app (a folder under lib/features/<name>/ with screens/ and widgets/), or when asked to create a new screen or widget in such an app. Scaffolds the folder structure and skeleton files enforcing a strict smart screen / dumb widget separation, so business logic never ends up in a presentational widget.
---

# Flutter feature scaffold

## When this applies

The project uses a feature-oriented folder structure:

```
lib/
  core/              # shared, cross-feature: router, routes, app shell, etc.
  features/
    <feature_name>/
      screens/
      widgets/
```

If the project's `CLAUDE.md` (or an equivalent project doc) describes a different structure, defer to that instead — this skill assumes the structure above unless told otherwise.

## The hard rule: smart screens, dumb widgets

This is not a style preference — treat it as a correctness requirement, carried over from Angular smart/presentational component conventions:

- **Screens** (`features/<name>/screens/*.dart`) are "smart":
  - Own all mutable state (`StatefulWidget` + `State`, or the project's state-management solution if one is in use — check before assuming `setState`).
  - Own all async calls: API clients, Rust FFI bridge calls, local services (storage, auth, etc.).
  - Own navigation (`context.go(...)`, `Navigator.push`, etc.).
  - Own loading/error state and pass it down as plain data.
  - Compose one or more widgets from `widgets/`, wiring callbacks and data into them.

- **Widgets** (`features/<name>/widgets/*.dart`) are "dumb":
  - Take everything they need as constructor parameters: data to display, and callbacks (`onSubmit`, `onTap`, `onChanged`, etc.) for anything that happens.
  - Never call a service, repository, API client, or FFI bridge function directly.
  - Never own state beyond trivial, purely-visual, non-business state (e.g. an `AnimationController`, a `TextEditingController` whose *value* is read by the parent on submit, a hover/focus flag). If you're unsure whether a piece of state is "business" or "visual," ask: does any other part of the app need to know about it? If yes, it belongs in the screen.
  - Prefer `StatelessWidget` by default. Only use `StatefulWidget` for the narrow visual-state case above.

**Before writing any widget file, ask: does this need to call a service/bridge/API, or just render data and report user actions upward?** If it needs to call something, that logic goes in the screen, and the widget gets a callback.

## Procedure

1. **Confirm the feature name** (snake_case, e.g. `chat`, `settings`, `user_profile`) and what screen(s)/widget(s) are actually needed — don't scaffold placeholder screens nobody asked for.

2. **Create the folders** (if they don't already exist):
   ```
   lib/features/<feature_name>/screens/
   lib/features/<feature_name>/widgets/
   ```

3. **Create the screen** at `lib/features/<feature_name>/screens/<screen_name>_screen.dart`:
   - `StatefulWidget` named `<ScreenName>Screen` (PascalCase) unless the project's state management makes it a `ConsumerWidget`/equivalent — check for Riverpod/Bloc/etc. in `pubspec.yaml` first and match the existing pattern if other screens already use one.
   - Owns `_isLoading` / `_errorMessage` (or equivalent) if the screen does any async work.
   - If it does async work, use a `try { ... } catch (e) { ... } finally { ... }` block, and guard any `setState` after an `await` with `if (mounted)`.
   - Dispose any controllers it owns.

4. **Create the widget(s)** at `lib/features/<feature_name>/widgets/<widget_name>.dart`:
   - `StatelessWidget` (default) named `<WidgetName>` (PascalCase, no "Widget" suffix needed unless it clarifies).
   - Constructor takes `required` data fields and `VoidCallback`/`ValueChanged<T>` callbacks for anything interactive.
   - No imports of services, repositories, API clients, or the generated FFI bridge bindings (e.g. anything under `lib/src/rust/`) in a widget file — that's the smell that logic leaked into the wrong layer.

5. **Wire them together** in the screen's `build()` — pass data down, pass callbacks down, handle what the callbacks do (including the actual async call) in the screen.

6. **Self-check before finishing** — re-read every new widget file and confirm:
   - It imports no service/repository/bridge code.
   - It has no `async` method that does real work (beyond e.g. a local `Future.delayed` for animation).
   - Every piece of interactive behavior is expressed as a constructor callback, not a method that reaches outside the widget.

   If any of these fail, move the offending code to the screen and re-wire via a callback — don't leave it "just this once."

7. **Routing**: if the project uses `go_router` with a `Routes` class of path constants (check `lib/core/routes.dart` or similar before assuming), add the new screen's route there and to the router config — don't hardcode path strings inline.

## Don'ts

- Don't create a `models/`, `services/`, `state/`, or `providers/` subfolder under the feature "for later" — only add structure once there's a second concrete file that needs it (see the project's own working-style conventions if a `CLAUDE.md` exists).
- Don't promote a widget to a shared/`core/` location just because it seems reusable in theory — only promote once it's actually used by a second feature.
- Don't introduce a state-management library as part of a scaffold task unless explicitly asked — scaffold with whatever the project already uses.
