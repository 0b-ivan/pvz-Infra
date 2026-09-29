# PVZ Staging online

The first public staging environment uses the same K3s/Flux/SOPS/Cloudflare operating model as the existing blog stack.

## Target

```text
Browser
  |
  v
Cloudflare
  |
  v
existing Cloudflare Tunnel
  |
  v
Traefik (kube-system)
  |
  +-- Host pvz-staging.obivan.org + /api --> pvz-backend:3000
  |
  '-- Host pvz-staging.obivan.org + /    --> pvz-game:8080
```

No NodePort, LoadBalancer or router port-forward is required.

## Staging hostname

```text
https://pvz-staging.obivan.org
```

The frontend and backend runtime configuration both use this same public origin.

## Prerequisites

The cluster must already have:

- K3s with Traefik
- Flux controllers in `flux-system`
- `flux-system/sops-age` containing the existing age identity
- the existing `cloudflare/cloudflared` deployment and remotely managed tunnel

Check:

```bash
flux check
kubectl -n kube-system get svc traefik
kubectl -n flux-system get secret sops-age
kubectl -n cloudflare get deployment cloudflared
```

## One-time Flux bootstrap

From a checkout of this repository:

```bash
kubectl apply -f bootstrap/flux-staging.yaml
```

Then wait for reconciliation:

```bash
flux reconcile source git pvz-infra
flux reconcile kustomization pvz-staging --with-source
flux get sources git
flux get kustomizations
```

Verify the namespace:

```bash
kubectl -n pvz-staging get pods,svc,ingress,pvc
kubectl -n pvz-staging rollout status deployment/pvz-backend
kubectl -n pvz-staging rollout status deployment/pvz-game
```

## Internal smoke test before Cloudflare

First prove Traefik routing from inside the cluster:

```bash
kubectl -n cloudflare run pvz-curl-test \
  --image=curlimages/curl:8.16.0 \
  --restart=Never --attach --rm -- \
  curl -fsS -H 'Host: pvz-staging.obivan.org' \
  http://traefik.kube-system.svc.cluster.local/healthz
```

Backend route:

```bash
kubectl -n cloudflare run pvz-api-test \
  --image=curlimages/curl:8.16.0 \
  --restart=Never --attach --rm -- \
  curl -fsS -H 'Host: pvz-staging.obivan.org' \
  http://traefik.kube-system.svc.cluster.local/api/health
```

Do not configure the public hostname until both requests succeed.

## Cloudflare Tunnel route

Reuse the existing remotely managed tunnel.

Create one published application route:

```text
Public hostname:  pvz-staging.obivan.org
Service URL:      http://traefik.kube-system.svc.cluster.local:80
HTTP Host Header: pvz-staging.obivan.org
```

The explicit Host override makes Traefik's host-based Ingress match deterministic.

## External smoke test

After the Cloudflare route is active:

```bash
curl -fsS https://pvz-staging.obivan.org/healthz
curl -fsS https://pvz-staging.obivan.org/api/health
curl -fsS https://pvz-staging.obivan.org/runtime-config.js
curl -fsSI https://pvz-staging.obivan.org/game/
```

Expected runtime config contains:

```text
https://pvz-staging.obivan.org
```

Only after all four checks pass should browser gameplay testing begin.

## Rollback

Flux owns the Kubernetes state. Roll back by reverting the infrastructure commit or restoring the previous image digest and letting Flux reconcile.

Do not edit the live Deployment image by hand as the normal rollback mechanism.
