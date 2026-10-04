# r16a AI skills and agents

Claude Code skills and review subagents for Flutter/Rust, Angular, Spring Boot and Kubernetes projects, plus a `/review` command that runs the relevant reviewers in parallel.

## Skills

| Skill | Triggers when | Notes |
|---|---|---|
| `review` | You type `/review [base-branch] [release]` | User-only (`disable-model-invocation`). Spawns the agents below in parallel. |
| `angular-best-practices` | Writing/reviewing Angular code | Version table (v14 → v22); signals/zoneless guidance gated by version |
| `spring-boot-best-practices` | Writing/reviewing Spring Boot / Java backend code | |
| `rust-idiomatic-review` | Finishing or reviewing Rust code | |
| `flutter-feature-scaffold` | Adding a feature/screen/widget to a feature-oriented Flutter app | Adapts to setState / Riverpod / Bloc / Provider |
| `flutter-performance-review` | Finishing or reviewing Flutter UI code | |
| `flutter-rust-env-setup` | Setting up or debugging flutter_rust_bridge + Cargokit | |
| `app-security-review` | Code touching input, auth, secrets, external data, dependencies | |
| `k8s-manifest-hardening` | Writing/reviewing Kubernetes manifests | |
| `privacy-legal-compliance` | Before launching an app that handles personal data | Not legal advice |
| `frontend-design` | Building new UI with a distinctive visual direction | Third-party, Apache-2.0 (see `skills/frontend-design/LICENSE.txt`) |

## Agents

All agents are read-only reviewers. Each returns findings as `[critical|high|medium|low] path:line — what's wrong — failure scenario`, most severe first, or exactly `No findings.`

| Agent | Triggers when | Preloaded skills (`skills:`) | Loads on demand (`Skill` tool) | Model |
|---|---|---|---|---|
| `code-quality-reviewer` | Rust, Dart, Java or Angular code changed | — | `rust-idiomatic-review`, `flutter-feature-scaffold`, `spring-boot-best-practices`, `angular-best-practices` (by file type) | default |
| `security-reviewer` | Input, auth, secrets, deps, crypto or k8s manifests changed | `app-security-review` | `k8s-manifest-hardening` (if manifests changed) | default |
| `performance-reviewer` | Flutter UI or performance-sensitive Rust changed | — | `flutter-performance-review` (for Dart) | sonnet |
| `design-reviewer` | UI/styling changed | — | — | sonnet |
| `legal-compliance-reviewer` | Before publishing / store submission | `privacy-legal-compliance` | — | default |

Preloaded skills must exist under the same names in your skills directory. Claude Code skips a missing one silently (debug-log warning only), so install skills and agents together.

## Install

**Globally (all projects)**: symlink, so `git pull` updates them:

```bash
REPO=~/Dev/r16a-ai-skills-and-agents   # adjust
mkdir -p ~/.claude/skills ~/.claude/agents
for d in "$REPO"/skills/*/;  do ln -sfn "${d%/}" ~/.claude/skills/; done
for f in "$REPO"/agents/*.md; do ln -sf "$f" ~/.claude/agents/; done
```

**Per project**: copy into the project's `.claude/` (committed with the project, shared with the team):

```bash
mkdir -p .claude/skills .claude/agents
cp -r "$REPO"/skills/* .claude/skills/
cp "$REPO"/agents/*.md .claude/agents/
```

Restart Claude Code (or start a new session) to pick up new agents. Check with `/agents` and by typing `/` for skills.

## Using `/review`

```
/review                 # changes vs main (committed on branch + uncommitted + untracked)
/review develop         # changes vs develop
/review main release    # also run legal-compliance-reviewer (pre-release check)
```

If the base branch doesn't exist (or you're on it), `/review` reviews staged + unstaged changes only. It runs only the reviewers whose area the change touched — e.g. a pure Spring change won't spawn the design reviewer — and merges their output into one deduplicated report sorted by severity. It never edits files.

If another `/review` command (built-in or from a plugin) takes precedence in your setup, rename the skill folder and its `name:` field (e.g. to `multi-review`).
