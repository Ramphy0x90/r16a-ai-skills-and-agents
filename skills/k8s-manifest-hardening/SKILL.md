---
name: k8s-manifest-hardening
description: Use when writing or reviewing any Kubernetes manifest (Deployment, Job, StatefulSet, Pod, CronJob, ConfigMap, Secret). Applies the securityContext, secrets, resources and image baseline that Trivy/kube-linter flag by default.
---

# Kubernetes manifest hardening

Apply this checklist to every workload manifest you write or review — not just when a scanner has already flagged something. These are the categories that misconfiguration scanners (Trivy config, kube-bench, kube-linter, etc.) check by default, and fixing them upfront avoids the iterative "write → scan → patch → rescan" loop.

## securityContext (the most commonly missed category)

Every container in every Deployment/Job/StatefulSet/Pod should set an explicit `securityContext`, both at the container level and, where applicable, the pod level. Relying on the runtime default is itself a finding (e.g. Trivy's KSV-0118), independent of what that default happens to be.

Minimum baseline for a container that doesn't need special privileges (the common case):

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: <a non-zero UID the image's user owns the relevant paths as>
  capabilities:
    drop:
      - ALL
```

- **`runAsNonRoot: true` + `runAsUser: <uid>`**: never run as root (UID 0) unless the image genuinely requires it (rare, and should be a deliberate documented exception, not a default). Check the image's Dockerfile/docs for what non-root UID it supports, if any — some images document a specific UID to use (don't guess blindly; if unknown, check the image's upstream docs or run `docker run --rm <image> id` style inspection before picking a number).
- **`readOnlyRootFilesystem: true`**: pair this with explicit `emptyDir` (or other) volume mounts for any path the container actually needs to write to (temp files, logs, sockets). A container that writes logs to its own filesystem (e.g. to `/app/server.log` instead of stdout) will break under this setting — the fix is usually to mount a writable `emptyDir` at that specific path, or better, reconfigure the app to log to stdout/stderr (the k8s-native pattern) instead of a file.
- **`allowPrivilegeEscalation: false`**: blocks a process from gaining more privileges than its parent (e.g. via setuid binaries) — safe to set `false` for essentially all application containers.
- **`capabilities.drop: [ALL]`**: drop all Linux capabilities by default; only add back a specific capability (`capabilities.add: [...]`) if the container has a proven, specific need (e.g. `NET_BIND_SERVICE` for binding to a port <1024) — don't add capabilities preemptively.

## Secrets

- **Never put a secret value (password, API key, token, private key) in a `ConfigMap`.** ConfigMaps are not encrypted at rest by default and are readable by anyone with ConfigMap read access in the namespace. Use a `Secret` instead, even for values that feel "low stakes" — e.g. a SQL init script that embeds a password inline should instead template the password in via an env var sourced from a `Secret` (e.g. `psql -v app_password="$APP_DB_PASSWORD"` with `APP_DB_PASSWORD` from a `secretKeyRef`), keeping only the non-secret SQL logic in the ConfigMap.
- Reference secret values via `valueFrom.secretKeyRef` in `env`, or as mounted secret volumes — never inline a literal secret value in a manifest committed to version control.
- If secrets are managed via SOPS, Sealed Secrets, or an external secrets operator, follow the project's existing pattern rather than introducing a different one.

## Resource limits

- Every container should set `resources.requests` and `resources.limits` for both `cpu` and `memory`. Missing resource limits isn't just a scanner finding — it allows one workload to starve others on a shared node, and makes scheduling/autoscaling decisions worse cluster-wide.

## Image and supply chain

- Avoid `:latest` (or no tag at all, which defaults to `:latest`) for anything beyond local experimentation — pin to a specific version tag (and ideally a digest, `image: name@sha256:...`, for production) so deployments are reproducible and rollbacks are possible.
- Set `imagePullPolicy: IfNotPresent` (or `Always` only when actually needed, e.g. a mutable `:latest`-style tag in a dev environment) deliberately rather than leaving it to defaults that vary by tag format.

## Other baseline checks

- `restartPolicy: Never` (or `OnFailure` with a sane `backoffLimit`) on one-shot Jobs — an unbounded-retry Job against a misconfigured dependency (e.g. wrong password, missing database) will otherwise spin forever, generating a pod per retry.
- Liveness/readiness probes on long-running workloads (Deployments/StatefulSets) so Kubernetes can detect and recover from an unhealthy-but-still-running container.
- Avoid `hostNetwork`, `hostPID`, `hostIPC`, and hostPath volume mounts unless there's a specific, justified infrastructure need (e.g. a node-level monitoring agent) — these all weaken container isolation.
- Namespace-scope RBAC: a workload's ServiceAccount should have only the permissions it actually needs — avoid binding to `cluster-admin` or using the `default` ServiceAccount for anything with real permissions attached.

## Common operational pitfall this baseline interacts with: app logging

A frequent failure mode when adding `readOnlyRootFilesystem: true` to an app not designed for it: the app tries to write a log file to its container filesystem (e.g. Python's `logging.FileHandler` configured to write to `/app.log`) and crashes with `OSError: Read-only file system` at startup. Two fixes, in order of preference:
1. Reconfigure the app to log to stdout/stderr instead (the standard container logging pattern — let the container runtime/log aggregator handle persistence).
2. If the app can't be reconfigured, mount a writable `emptyDir` specifically at that log path rather than disabling `readOnlyRootFilesystem` for the whole container.

## How to run this review

1. For every manifest written or touched, check each category above.
2. Apply the `securityContext` baseline by default to every container — treat "no securityContext" as always wrong, never as "fine for now."
3. If a scanner (Trivy, kube-linter, etc.) is available in the project's CI or locally, run it after making changes to confirm the fixes actually resolved the findings rather than assuming.
4. If a specific check genuinely can't be satisfied (e.g. an image that truly requires root), document why in a manifest comment rather than silently leaving the gap.
