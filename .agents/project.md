# Project reference

## Purpose

`sites_kubernetes` is the GitOps repository for small websites hosted in a k3s
cluster. It contains application Helm charts, infrastructure manifests, and
Argo CD Applications.

## Production topology

- Kubernetes distribution: k3s.
- Ingress controller: Traefik.
- GitOps controller: Argo CD in namespace `argocd`.
- TLS certificates: cert-manager.
- Container registry: GitHub Container Registry (`ghcr.io`).
- Application image automation: Argo CD Image Updater.

## Repository layout

- `apps/semka-informatics/` — Helm chart for the Flask website.
- `apps/semka-informatics/values.yaml` — image, service, ingress, resources,
  environment, security context, and persistence settings.
- `argocd/semka-informatics.yaml` — Argo CD Application for the website.
- `argocd/image-updater.yaml` — bootstrap Application for Image Updater.
- `argocd/semka-informatics-image-updater.yaml` — digest tracking configuration.
- `infra/namespaces.yaml` — application namespaces.
- `infra/cert-manager/` — ACME ClusterIssuers.
- `docs/argocd.md` — Argo CD operations.
- `docs/ci-image-flow.md` — application CI, GHCR, and deployment flow.

## semka-informatics deployment

- Namespace: `semka-informatics`.
- Image: `ghcr.io/isemene4kai/semka-informatics:latest`.
- Image Updater strategy: `digest`.
- Service: `ClusterIP`, port `80` to container port `8000`.
- Ingress: Traefik for `isemene4kai.ru` with TLS.
- Liveness: `/health` on port `8000`.
- Readiness: `/ready` on port `8000`.
- Persistent view data: PVC mounted at `/data`.
- `VIEWS_FILE`: `/data/views.json`.
- Workload user/group and volume `fsGroup`: `1000`.
- ServiceAccount token automount is disabled.

## Ownership boundaries

Application source code, Dockerfile, tests, and GHCR publishing workflow live in
the separate `ISemene4kaI/semka_informatics` repository. This repository defines
only the desired Kubernetes deployment state and shared cluster infrastructure.
