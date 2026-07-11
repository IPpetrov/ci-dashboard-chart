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

**Don't hand-edit `image.repository`/`image.tag`/`image.pullPolicy`** —
they're owned by the CI pipeline in `ci-dashboard-k8s`. Pull before editing
anything else in `values.yaml` to avoid conflicting with the bot's commits.

## values.yaml reference (non-obvious fields only)

| Field | Purpose |
|---|---|
| `containerPort` | Must match what gunicorn binds to in the app (`8080`). Top-level, not nested under `service:` — easy mistake, cost us a debugging round. |
| `service.nodePort` | `30080` — must match the `extraPortMappings` in the kind cluster config, or `localhost:30080` won't resolve to anything. |
| `imagePullSecrets` | References `ecr-pull-secret`, a Kubernetes secret created manually on the cluster (not in Git — it contains a live AWS token). See `ci-dashboard-k8s`'s README for the create command. Expires ~12h, needs periodic refresh. |
| `livenessProbe` / `readinessProbe` | Point at `/health`, not `/` — the scaffold defaults to `/`, which doesn't exist on this app. |

## ArgoCD Application

Defined in `argocd-app.yaml` (in this repo, applied manually — ArgoCD
doesn't manage its own Application resource recursively):

```bash
kubectl apply -f argocd-app.yaml
```

Key settings: `syncPolicy.automated` with `selfHeal: true` and
`prune: true` — the cluster will auto-correct any manual `kubectl` changes
to match this repo, and will delete resources removed from Git. This was
verified live: scaling the deployment manually via `kubectl scale` was
reverted by ArgoCD within seconds; a `git push` changing `replicaCount`
took effect within ArgoCD's default ~3 min poll interval.

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
release name have caused install conflicts before (`helm uninstall` was
needed once to give ArgoCD a clean slate).

## Recreating the ArgoCD install from scratch

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8081:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Login at `https://localhost:8081`, user `admin`, password from the command above.
