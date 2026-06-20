# Argo CD and GitOps skill

Use this file for changes under `argocd/`, image automation, or synchronization
behavior.

## Application behavior

- `targetRevision` should track the intended Git branch.
- `source.path` must point to a valid Helm chart or Kustomize application.
- `destination.namespace` must match chart resources and namespace bootstrap.
- Automated sync currently uses `prune: true` and `selfHeal: true`.
- Do not confuse auto-sync with image discovery: Argo CD reconciles Git state but
  does not detect a changed registry digest under an unchanged mutable tag.

## Image Updater

- The controller is bootstrapped by `argocd/image-updater.yaml`.
- The `ImageUpdater` CR must only be applied after its CRD is established.
- `semka-informatics` tracks `latest` with the `digest` strategy.
- Helm target fields are `image.repository` and `image.tag`.
- The configured write-back method is `argocd`; changing it to Git requires
  repository write credentials and a deliberate persistence strategy.

## Bootstrap order

These commands mutate the cluster and require explicit authorization and Argo
CD namespace permissions:

```bash
sudo k3s kubectl apply -f argocd/image-updater.yaml
sudo k3s kubectl wait --for=condition=Established \
  crd/imageupdaters.argocd-image-updater.argoproj.io --timeout=180s
sudo k3s kubectl apply -f argocd/semka-informatics.yaml
sudo k3s kubectl apply -f argocd/semka-informatics-image-updater.yaml
```

After bootstrap, application updates should flow through Git and controller
reconciliation rather than repeated manual applies.

## Verification

```bash
sudo k3s kubectl get applications -n argocd
sudo k3s kubectl get imageupdaters -n argocd
sudo k3s kubectl describe imageupdater semka-informatics -n argocd
```

Check sync status, health, Image Updater conditions, last checked time, recent
updates, and the image reported by the Application.
