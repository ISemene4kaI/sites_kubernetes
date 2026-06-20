# Validation skill

Use this file before finishing any repository change.

## Local checks

```bash
helm lint apps/semka-informatics
helm template semka-informatics apps/semka-informatics \
  --namespace semka-informatics >/tmp/semka-informatics-rendered.yaml
git diff --check
git status --short
```

Review `/tmp/semka-informatics-rendered.yaml` when the chart changed. Validate
standalone manifests with a client-side dry run when the corresponding CRDs are
available:

```bash
kubectl create --dry-run=client --validate=false -f <manifest> -o yaml
```

Absence of a CRD is an expected blocker for validating its custom resources;
report it rather than pretending validation succeeded.

## Cluster checks after an authorized deployment

```bash
sudo k3s kubectl get application semka-informatics -n argocd
sudo k3s kubectl rollout status deployment/semka-informatics \
  -n semka-informatics --timeout=180s
sudo k3s kubectl get pods -n semka-informatics
sudo k3s kubectl get endpoints semka-informatics -n semka-informatics
```

If rollout fails, collect `describe`, logs, and events before changing anything.
Do not hide a failed rollout by increasing timeouts without evidence.
