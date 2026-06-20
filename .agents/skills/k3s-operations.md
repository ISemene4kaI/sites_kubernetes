# k3s operations skill

Use this file when a decision depends on the actual cluster or when validating a
deployment after reconciliation.

## Requesting cluster state

Ask the user to run only the minimum relevant read-only commands. A useful
baseline is:

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get namespaces
sudo k3s kubectl get applications -n argocd
sudo k3s kubectl get pods -A -o wide
sudo k3s kubectl get storageclass
sudo k3s kubectl get pvc -A
sudo k3s kubectl get ingress -A
```

For `semka-informatics` troubleshooting:

```bash
sudo k3s kubectl get all,configmap,ingress,pvc -n semka-informatics
sudo k3s kubectl describe deployment semka-informatics -n semka-informatics
sudo k3s kubectl describe pod -n semka-informatics -l app=semka-informatics
sudo k3s kubectl logs -n semka-informatics deployment/semka-informatics --tail=200
sudo k3s kubectl get events -n semka-informatics --sort-by=.lastTimestamp
```

Do not request Secrets or complete kubeconfigs. Redact tokens, credentials,
certificate keys, and private registry data from shared output.

## Safe diagnosis order

1. Check Argo CD sync and health status.
2. Check Deployment rollout and ReplicaSet state.
3. Check Pod phase, conditions, probes, restarts, and events.
4. Check application logs.
5. Check Service selectors and endpoints.
6. Check Ingress, certificate, and Traefik routing.
7. Check PVC binding, mount permissions, and writable paths.

Do not restart or delete resources until the cause is understood. Pod deletion
is not a fix for a persistent declarative or storage problem.
