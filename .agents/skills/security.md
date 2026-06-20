# Kubernetes security skill

Use this file for workload identity, RBAC, Secrets, ingress, TLS, container, or
storage changes.

## Workload security

- Use fixed non-root UID/GID values compatible with the image.
- Set `runAsNonRoot: true`, disable privilege escalation, drop all capabilities,
  and use `RuntimeDefault` seccomp.
- Add capabilities back only when a documented workload requirement exists.
- Keep root filesystems read-only when the application and mounted writable
  paths support it.
- Disable ServiceAccount token automount for workloads that do not call the
  Kubernetes API.

## RBAC and controllers

- Grant controllers only the verbs, resources, and namespaces they require.
- Prefer namespace-scoped Roles and RoleBindings over cluster-wide permissions.
- Before installing a controller, inspect its CRDs, ServiceAccount, RBAC, image,
  watch scope, and upgrade strategy.
- Do not install remote `stable` manifests blindly in a sensitive cluster;
  review changes before controller upgrades.

## Secrets and TLS

- Store secret values outside Git or use an approved encrypted-secret system.
- Manifest files may reference Secret names but must not contain plaintext
  credentials.
- Keep production and staging certificate issuers clearly separated.
- Public Ingress resources require TLS and an explicit host.
- Verify certificate readiness and renewal without exposing private key data.

## Storage

- Treat PVC deletion or recreation as destructive even when reclaim policies
  appear safe.
- Confirm access mode, storage class, reclaim policy, capacity, and backup needs
  before changing persistence.
- Maintain UID/GID and `fsGroup` compatibility for non-root writers.
