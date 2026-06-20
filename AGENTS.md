# Instructions for AI agents

## Core principles

- Treat this repository as the declarative source of truth for production
  Kubernetes resources.
- Inspect the relevant Helm chart, Argo CD Application, documentation, and
  current diff before making changes.
- Prefer the smallest complete and reversible change that achieves the requested
  operational outcome.
- Follow existing chart conventions and avoid introducing a second deployment
  mechanism for the same application.
- Keep desired state in Git. Do not rely on undocumented manual cluster changes.
- Preserve unrelated user changes and never overwrite a dirty worktree.

## Correctness

- Validate assumptions about API versions, chart values, controllers, CRDs,
  namespaces, storage classes, ingress classes, and cluster capabilities.
- Do not assume that a resource exists in the cluster merely because a manifest
  exists in this repository.
- Trace every value from `values.yaml` through templates into rendered
  Kubernetes resources.
- Keep selectors, labels, service ports, container ports, probe ports,
  namespaces, PVC names, and environment variables consistent.
- Account for rollout behavior, readiness, persistence, permissions, controller
  reconciliation, and failure recovery.
- Use immutable or digest-resolved image references for reproducibility. If a
  mutable tag is used, configure a controller that detects digest changes and
  triggers a rollout.

## Security

- Apply least privilege to ServiceAccounts, RBAC, container capabilities,
  filesystem access, and network exposure.
- Keep workloads non-root unless there is a documented technical requirement.
- Preserve `allowPrivilegeEscalation: false`, dropped capabilities, seccomp,
  non-root IDs, and compatible volume `fsGroup` settings.
- Disable automatic ServiceAccount token mounting unless the workload uses the
  Kubernetes API.
- Never commit kubeconfigs, tokens, passwords, private keys, registry
  credentials, certificate private keys, or unencrypted Secrets.
- Do not expose dashboards, metrics, or administrative endpoints through
  Ingress without explicit authentication and TLS requirements.
- Use supported Kubernetes APIs and authoritative controller documentation.

## Validation

- Every Helm change must pass `helm lint` and `helm template`.
- Inspect rendered manifests, not only source templates.
- Run `git diff --check` and review the complete final diff.
- Use client-side dry runs where useful, but remember that CRDs and admission
  policies may require validation against the real cluster.
- For cluster-dependent work, request the minimum read-only command output
  needed before deciding.
- Never claim a resource was deployed, healthy, or synchronized unless that was
  verified against the cluster.

## GitOps and operational safety

- Editing files authorizes local repository changes only.
- Committing, pushing, opening pull requests, applying manifests, syncing Argo
  CD, restarting workloads, and modifying the cluster require explicit user
  intent.
- Prefer Git and Argo CD reconciliation over direct `kubectl apply` for managed
  application resources.
- Bootstrap resources such as Argo CD Applications or controller CRDs may need
  a documented one-time manual apply.
- Never use destructive operations such as deleting PVCs, namespaces, CRDs, or
  Argo CD Applications without explicit confirmation and a recovery plan.
- Preserve persistent data and explain rollout or downtime implications before
  applying risky changes.

## Documentation quality

- Update operational documentation when bootstrap steps, image flow, required
  controllers, permissions, or rollback procedures change.
- Commands must state where they run, which privileges they require, and whether
  they mutate the cluster.
- Keep examples aligned with the actual manifests and current deployment model.

## Project knowledge and skills

Repository-specific information is stored under `.agents/`. Read
`.agents/project.md` before substantial work, then load the skill files relevant
to the task:

- `.agents/skills/helm-chart.md` — chart structure, values, templates, and render
  validation.
- `.agents/skills/argocd-gitops.md` — Applications, auto-sync, Image Updater, and
  bootstrap ordering.
- `.agents/skills/k3s-operations.md` — safe cluster inspection, server commands,
  deployment checks, and troubleshooting evidence.
- `.agents/skills/security.md` — workload, RBAC, Secret, TLS, ingress, and PVC
  security requirements.
- `.agents/skills/validation.md` — required local and cluster-aware checks.

Combine all applicable skills when a task crosses domains. If a skill conflicts
with this file, this file takes precedence.
