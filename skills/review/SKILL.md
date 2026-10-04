---
name: review
description: Use when the user runs /review to review the current branch or uncommitted changes. Picks the relevant reviewer agents from the changed files, runs them in parallel, and merges their findings into one severity-sorted report.
argument-hint: "[base-branch] [release]"
disable-model-invocation: true
allowed-tools: Bash(git diff *) Bash(git rev-parse *) Bash(git merge-base *) Bash(git status *)
---

# /review — parallel multi-reviewer pass

Arguments: `$ARGUMENTS`
- First argument that isn't `release`: the base branch (default `main`).
- `release` (or the user saying this is a pre-release / pre-launch check): also run legal-compliance.

## 1. Collect the changed files

1. If `git rev-parse --verify --quiet <base>` succeeds and `HEAD` isn't the base itself, the change set is the union of:
   - `git diff --name-only <base>...HEAD` (committed on this branch)
   - `git diff --name-only HEAD` (staged + unstaged)
2. Otherwise (no such base, or on the base branch): staged + unstaged only — `git diff --name-only HEAD` (or `git diff --name-only --cached` plus `git diff --name-only` in a repo with no commits).
3. Add untracked files from `git status --porcelain` (`??` lines) — new files are often the ones that most need review.
4. Drop deleted files. If the set is empty, say "Nothing to review." and stop.

Note the exact diff command for the reviewers (e.g. `git diff main...HEAD` + `git diff HEAD`) so they review the same change set.

## 2. Decide which reviewers apply

Run a reviewer only if its area was touched. Use paths first; for borderline cases, look at the diff hunks (`git diff <range> -- <file>`), not just the extension.

| Reviewer (agent) | Run when the change set contains |
|---|---|
| `code-quality-reviewer` | any `.dart`, `.ts`, `.java`, `.rs` source (tests count; generated files like `*.g.dart`, `*.freezed.dart`, `frb_generated.*` don't) |
| `design-reviewer` | UI: Flutter screens/widgets/theme files, Angular component `.ts`/`.html`, `.css`/`.scss`, other templates or design tokens |
| `performance-reviewer` | lists/collections rendered in UI, images, async/stream/state code, animations, or Rust/Java hot paths (loops over large data, serialization, per-request/per-frame code, queries) |
| `security-reviewer` | auth/session/login code, input handling (controllers, forms, parsers, deep links), secrets/config (`.env*`, `application*.yml`, `*secret*`), crypto/E2EE, dependency manifests or lockfiles (`pubspec.*`, `package*.json`, `Cargo.*`, `pom.xml`, `build.gradle*`), Kubernetes manifests |
| `legal-compliance-reviewer` | only when `release` was passed or the user said this is pre-release — never by default |

Docs-only, CI-only or formatting-only changes may need no reviewer at all — say so instead of spawning one.

## 3. Run them in parallel

Spawn every applicable reviewer **in a single message** (one Agent call per reviewer, all in the same turn) so they run concurrently. Don't run them one after another, and don't spawn any reviewer whose area wasn't touched.

Give each one a self-contained prompt — they don't see this conversation:
- the repo root and the exact diff command(s) from step 1,
- the subset of changed files relevant to that reviewer (it may look at adjacent files),
- for legal-compliance, that this is a pre-release review of the whole app, not just the diff.

## 4. Merge into one report

Each reviewer returns lines shaped `[severity] path:line — what's wrong — failure scenario`, or `No findings.`

1. Collect every finding line; keep the reviewer name with each.
2. Dedupe: the same `path:line` (or lines a few apart) describing the same underlying problem is one finding — keep the clearer wording and the higher severity, and list every reviewer that raised it.
3. Sort critical → high → medium → low; within a severity, group by file.
4. If a reviewer returned something that isn't in the format, keep its substance and normalize it — don't drop it.

Output:

```
## Review: <base or "uncommitted changes"> — <N> files

Ran: code-quality, security · Skipped: design (no UI files), performance (no hot paths), legal (not a release)

[critical] path:line — … — …  (security)
[high] path:line — … — …  (code-quality, performance)
…
```

If every reviewer returned `No findings.`, say so in one line after the Ran/Skipped line. Carry over legal-compliance's not-legal-advice line if it ran and reported anything.

Don't fix anything as part of /review — present the report and let the user decide what to fix.
