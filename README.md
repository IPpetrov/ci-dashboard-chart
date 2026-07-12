# ci-dashboard-chart

Helm chart for the [CI Dashboard](https://github.com/IPpetrov/ci-dashboard-k8s)
app. Deployed to a local `kind` cluster via ArgoCD using GitOps — this repo
is the source of truth for what's running; nobody should run
`helm install`/`upgrade` against the cluster by hand except for local
testing (see below).

## How a deploy actually happens

1. Code is pushed to `ci-dashboard-k8s`.
2. Its GitHub Actions workflow builds an image, pushes to private ECR, and
   commits an updated `image.tag` (the commit SHA) into this repo's
   `values.yaml`.
3. ArgoCD (watching this repo) picks up the change and runs Helm to
   reconcile the cluster.
4. Prometheus (via the `ServiceMonitor` in this chart) scrapes the new
   pods' `/metrics`; Grafana visualizes it.

**Don't hand-edit `image.repository`/`image.tag`/`image.pullPolicy`** —
they're owned by the CI pipeline in `ci-dashboard-k8s`. Run `git pull`
before editing anything else in `values.yaml` to avoid conflicting with the
bot's commits (`git config pull.rebase false` once, to avoid the
"diverged branches" prompt on every pull).

## values.yaml reference (non-obvious fields only)

| Field | Purpose |
|---|---|
| `containerPort` | Must match what gunicorn binds to in the app (`8080`). Top-level, not nested under `service:`. |
| `service.nodePort` | `30080` — must match the `extraPortMappings` in `ci-dashboard-k8s`'s `kind-config.yml`. |
| `imagePullSecrets` | References `ecr-pull-secret`, a Kubernetes secret created/refreshed on the cluster (not in Git — it contains a live AWS token). Auto-refreshed every 6h by the CronJob in this repo (see below). |
| `livenessProbe` / `readinessProbe` | Point at `/health`, not `/`. `initialDelaySeconds: 5` gives gunicorn time to boot before the first check; readiness polls more often (`periodSeconds: 5`) than liveness (`10`) since it controls traffic routing, not pod restarts. |
| `resources` | Explicit requests/limits (50m/64Mi requests, 200m/128Mi limits) — needed for Grafana's utilization-percentage panels to show data at all; without these they show "No data" since there's nothing to divide usage by. |

## Templates beyond the Helm scaffold defaults

- `servicemonitor.yaml` — tells the Prometheus Operator to scrape this
  app's `/metrics`. Carries the `release: monitoring` label, which is
  required — the Operator only watches ServiceMonitors with a label
  matching the Prometheus Helm release name (`monitoring`); without it,
  Prometheus silently ignores the resource, no error anywhere.
- `networkpolicy.yaml` — restricts ingress to port 8080 only, and egress
  to 443 (HTTPS, for the GitHub API calls the app makes) and 53 (DNS,
  both TCP/UDP — needed to resolve `api.github.com` at all). Verified the
  app still works end-to-end with this applied (health check + real
  GitHub data on the homepage). Note: kind's default CNI has limited
  NetworkPolicy enforcement — this is correct and ready for a real
  cluster, but may not be actively enforced locally.

## ArgoCD Application

Defined in `argocd-app.yaml`, applied manually (ArgoCD doesn't manage its
own Application resource recursively):

```bash
kubectl apply -f argocd-app.yaml
```

`syncPolicy.automated` with `selfHeal: true` and `prune: true` — the
cluster auto-corrects manual `kubectl` changes to match this repo, and
deletes resources removed from Git. Verified live: `kubectl scale`'d the
deployment manually, ArgoCD reverted it within seconds; a `git push`
changing `replicaCount` took effect within ArgoCD's ~3 min default poll
interval.

## ECR pull secret auto-refresh

`ecr-refresher-rbac.yaml` + `ecr-refresher-cronjob.yaml`. Runs every 6
hours inside the cluster (well within the ~12h ECR token lifetime), using
a dedicated minimal-permission IAM user (`ECRPullRefresher`,
`AmazonEC2ContainerRegistryReadOnly` only — separate from the
GitHub Actions IAM user). Survives cluster restarts/reboots without any
manual intervention — verified after a full WSL power-off/restart.

Check it's running:
```bash
kubectl get cronjob ecr-pull-secret-refresh
kubectl get jobs   # look for recent "Completed" runs
```

## Local Helm testing (bypasses ArgoCD — for debugging only)

```bash
helm template ci-dashboard-chart .   # dry-run, check rendered YAML
helm install ci-dashboard .          # only if ArgoCD Application doesn't exist yet
helm upgrade ci-dashboard .          # manual upgrade — normally ArgoCD does this
helm history ci-dashboard
helm rollback ci-dashboard <REVISION>
```

If ArgoCD is already managing the `ci-dashboard` release, avoid running
these directly — Helm-managed and ArgoCD-managed changes to the same
release name can conflict (`helm uninstall` was needed once to give
ArgoCD a clean slate).

## Recreating the ArgoCD + monitoring stack from scratch

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
kubectl port-forward svc/argocd-server -n argocd 8081:443   # or rely on the NodePort patch, see ci-dashboard-k8s README
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
```
Login at `https://localhost:8081` (ArgoCD, user `admin`) and
`http://localhost:30300` (Grafana, `admin`/`prom-operator`).
