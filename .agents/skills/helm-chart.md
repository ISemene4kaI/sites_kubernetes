# Helm chart skill

Use this file for changes under `apps/`.

## Chart conventions

- Put deploy-time configuration in `values.yaml` and reference it from
  templates; avoid unexplained literals in templates.
- Quote string environment values in ConfigMaps.
- Keep resource names and selectors derived consistently from `.Chart.Name`.
- Guard optional resources and mounts with their corresponding `enabled` value.
- Preserve namespace consistency across Deployment, Service, PVC, Ingress,
  ConfigMap, and ServiceAccount.

## Workload requirements

- Keep service and container ports aligned.
- Readiness and liveness probes must target real application endpoints.
- Configure CPU and memory requests and limits.
- Keep the container non-root with dropped capabilities, runtime-default
  seccomp, and privilege escalation disabled.
- Keep `automountServiceAccountToken: false` unless Kubernetes API access is
  explicitly required.
- When mounting a writable PVC, align `runAsUser`, `runAsGroup`, and `fsGroup`
  with the image user.

## Required validation

```bash
helm lint apps/semka-informatics
helm template semka-informatics apps/semka-informatics \
  --namespace semka-informatics
git diff --check
```

Inspect the rendered Deployment for image, security context, environment,
probes, resources, volume mount, and Service port. Inspect the rendered Ingress
for host, class, issuer annotations, backend, and TLS secret.
