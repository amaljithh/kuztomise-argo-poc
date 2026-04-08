# ArgoCD Canary — Kustomize

Converted from Helm to Kustomize with **dev / qa / prod** overlays.

## Repository Structure

```
.
├── app-of-apps/                  # ArgoCD App-of-Apps bootstrap
│   ├── app-of-apps.yaml
│   └── kustomization.yaml
├── base/                         # Shared manifests (env-agnostic)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── service.yaml              # active + preview services
│   ├── rollout.yaml              # Argo Rollout (canary)
│   ├── gateway.yaml              # Istio Gateway API
│   ├── httproute.yaml
│   ├── analysis-template.yaml   # AnalysisTemplate: blue-green-check
│   └── infra-apps.yaml          # ArgoCD Apps for gateway-api CRDs + Istio
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── argocd-application.yaml
    │   ├── rollout-patch.yaml    # 1 replica, fast 2-step canary
    │   ├── configmap-patch.yaml
    │   └── httproute-patch.yaml  # dev.myapp.example.com
    ├── qa/
    │   ├── kustomization.yaml
    │   ├── argocd-application.yaml
    │   ├── rollout-patch.yaml    # 2 replicas, standard canary + analysis
    │   ├── configmap-patch.yaml
    │   └── httproute-patch.yaml  # qa.myapp.example.com
    └── prod/
        ├── kustomization.yaml
        ├── argocd-application.yaml  # Manual sync — no auto-deploy
        ├── rollout-patch.yaml    # 4 replicas, slow canary with 3x analysis gates
        ├── configmap-patch.yaml
        └── httproute-patch.yaml  # myapp.example.com
```

## Environment Differences

| Feature              | dev                    | qa                          | prod                              |
|----------------------|------------------------|-----------------------------|-----------------------------------|
| Namespace            | `myapp-dev`            | `myapp-qa`                  | `myapp-prod`                      |
| Replicas             | 1                      | 2                           | 4                                 |
| Canary steps         | 50% → 100% (fast)      | 25% → 50% → 100% + analysis | 10%→25%→50%→75%→100% + 3x analysis|
| ArgoCD auto-sync     | ✅ automated            | ✅ automated                 | ❌ manual approval required        |
| Hostname             | dev.myapp.example.com  | qa.myapp.example.com        | myapp.example.com                 |

## Bootstrap

### 1. Install infrastructure apps (once per cluster)
```bash
kubectl apply -f base/infra-apps.yaml
```

### 2. Bootstrap App-of-Apps
```bash
kubectl apply -f app-of-apps/app-of-apps.yaml
```
This creates the ArgoCD Applications for all three environments automatically.

### 3. Update image per environment
Edit the `image` field in the relevant `overlays/<env>/rollout-patch.yaml`:
```yaml
containers:
  - name: nginx-chart
    image: amaljithh/green-page:<new-tag>
```

## Manual Kustomize Preview

```bash
# Preview rendered manifests for any overlay
kubectl kustomize overlays/dev
kubectl kustomize overlays/qa
kubectl kustomize overlays/prod

# Apply directly (bypassing ArgoCD)
kubectl apply -k overlays/dev
```

## Prod Deployment

Prod has **no automated sync**. To deploy:
1. Commit your changes and open a PR.
2. After merge, trigger a manual sync in the ArgoCD UI (or via CLI):
   ```bash
   argocd app sync myapp-prod
   ```
3. Monitor the canary rollout:
   ```bash
   kubectl argo rollouts get rollout prod-nginx -n myapp-prod --watch
   ```
