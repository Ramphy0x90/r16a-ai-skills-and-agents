---
name: app-security-review
description: Use before calling any feature that handles user input, authentication, secrets, or external data done, or when asked to review code for security issues. Covers secrets handling, input validation, injection risks, auth/session handling, insecure deserialization, and dependency vulnerabilities — application/code level, complementing k8s-manifest-hardening (infrastructure level).
---

# Application security review

Run this as a self-review pass on code that touches user input, authentication, secrets, external data, or dependencies — not only when explicitly asked for a "security review." This is the code/application-level counterpart to `k8s-manifest-hardening` (infra-level) — use both together on a full-stack feature.

## Secrets

- **No hardcoded credentials, API keys, tokens, or private keys in source code, ever** — not even "temporarily," not even in a comment, not even in a test fixture that looks disposable. Use environment variables, a secrets manager, or the project's established secrets pattern (check `CLAUDE.md`/existing code before introducing a new one).
- Before committing, scan the diff for anything that looks like a credential (long random-looking strings, anything named `*_key`, `*_secret`, `*_token`, `*_password` assigned a literal value) — this is cheap to check and catastrophic to miss.
- Secrets must not end up in logs — review `log`/`println!`/`print`/`eprintln!`/`console.log` calls near auth/credential code for accidental leakage (e.g. logging a full request object that includes a password field, or logging a token "for debugging").
- Secrets must not end up in client-visible error messages — a stack trace or error response shown to an end user should never include a connection string, internal file path revealing secrets, or similar.

## Input validation

- Treat all external input as untrusted: HTTP request bodies/params/headers, file uploads, query strings, deep-linked data, data from another service, and (for a Matrix/federated or any multi-tenant system) data received from other servers/users.
- Validate shape and bounds (type, length, range, allowed character set) at the boundary where input enters the system — don't assume validation happened earlier in the call chain unless that's structurally guaranteed.
- For anything that builds a query, command, or path from input: use parameterized queries (never string-concatenate user input into SQL), avoid shelling out with unsanitized input (avoid `sh -c` string interpolation; prefer an API that takes arguments as a list), and canonicalize/validate file paths built from input to prevent path traversal (`../../etc/passwd`-style).

## Authentication & session handling

- Session/auth tokens must be stored securely on the client (e.g. platform secure storage — Keychain/Keystore equivalents — not plain `SharedPreferences`/`localStorage`/unencrypted files) and transmitted only over encrypted transport.
- **Checking that a session *exists* locally is not the same as checking that it's *valid*.** A client-side "is a session stored?" check is a UX convenience, not an auth boundary — the server must independently verify the token/session on every privileged request. If the project currently only does the local-existence check (common early in a build), flag this explicitly as a known gap rather than treating it as sufficient.
- Passwords (if the system ever handles raw passwords, e.g. at a login form before exchange for a token) should never be logged, cached, or persisted — only transmitted once, over an encrypted channel, to the authentication endpoint.
- Use well-reviewed libraries for anything cryptographic (hashing, encryption, token signing/verification) — never hand-roll crypto.

## Deserialization & parsing

- Deserializing untrusted input into a type that can trigger arbitrary behavior (e.g. deserializing to a dynamic/polymorphic type that invokes constructors or code based on input content) is dangerous — prefer deserializing into fixed, known schemas (e.g. `serde` into a concrete `struct`, not an open-ended dynamic map, when the shape is known).
- Validate deserialized data's invariants after parsing (e.g. an ID field that must be a specific format) rather than assuming a successful parse means the data is semantically valid.

## Dependency hygiene

- Before adding a new dependency, check it's actively maintained and has no widely-known unpatched vulnerabilities (a quick check of the ecosystem's advisory database — `cargo audit` for Rust, `npm audit`/`dart pub outdated`-equivalent for the relevant ecosystem — is cheap and worth doing).
- If the project has a lockfile, don't bypass it casually (e.g. don't resolve a Docker build failure by switching `npm ci` to `npm install` without first checking *why* the lockfile is out of sync — regenerate the lockfile deliberately instead, so the dependency tree stays reproducible and audited).
- Periodically (or when touching a dependency-heavy area) check for known-vulnerable versions of direct dependencies, especially for anything handling crypto, parsing, or network input.

## End-to-end encryption specifics (relevant for Matrix/E2EE work)

- Never log decrypted message content, room keys, or session keys, even at debug level.
- Session/device verification state should be checked before treating a peer as trusted (if building UI around device verification, don't silently treat unverified devices as verified for convenience).
- Key material must use secure storage, matching the session-storage guidance above — never a plain file or unencrypted database field.

## How to run this review

1. Identify what in the current task touches input, auth, secrets, or dependencies — most tasks will touch at least one.
2. Walk the relevant sections above against the actual code.
3. Fix straightforward findings inline (e.g. moving a hardcoded value to an env var).
4. For findings that are structural/architectural (e.g. "auth only checks local session existence, not server-side validity"), flag them explicitly rather than silently fixing or silently ignoring — these often need a product decision, not just a code change.
5. Don't claim a security review was done if none of the sections actually applied — say plainly that the code didn't touch any of these areas.
