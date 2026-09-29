# pvz-Infra

Infrastructure for the `0b-ivan` Plants vs. Zombies browser-game stack.

## Responsibilities

This repository owns:

- local frontend/backend integration with Docker Compose
- immutable application image references
- K3s/Kubernetes manifests
- Kustomize environment overlays
- environment-specific runtime configuration
- deployment validation

Application source remains in `pvz-game` and `pvz-backend`. Architecture documentation remains in `pvz-docs`.

## Published baseline

The current infrastructure baseline is pinned by OCI digest:

```text
pvz-game
ghcr.io/0b-ivan/pvz-game@sha256:ad652f80c6d7df441cdfcb298db328cfe67bc103ba794c60f10085c4d2af7cbd

pvz-backend
ghcr.io/0b-ivan/pvz-backend@sha256:81bb473de4ce31a2a2fdd34eb95f8e817fe47daea7a943eacb97371505bb5642
```

No deployment depends on `latest`.

## Local integration

```bash
cp .env.example .env
docker compose --env-file .env up -d --wait
```

Then:

```text
Game:    http://localhost:8080/game/
Backend: http://localhost:3000/api/health
```

Stop and remove the local stack:

```bash
docker compose down
```

Use `docker compose down -v` only when you intentionally want to delete the local backend data volume.

## Kubernetes

The target runtime is K3s. Manifests use Kustomize:

```text
kubernetes/
├── base/
│   ├── game/
│   ├── backend/
│   └── ingress.yaml
└── overlays/
    ├── staging/
    └── production/
```

Render without applying:

```bash
kubectl kustomize kubernetes/overlays/staging
kubectl kustomize kubernetes/overlays/production
```

Staging is configured for `staging-pvz.obivan.org`. Production is prepared for `pvz.obivan.org`, but remains undeployed until the staging gate has passed.

### Traffic model

Kubernetes uses one browser-facing host per environment:

```text
https://staging-pvz.obivan.org/
  ├── /api  -> pvz-backend:3000
  └── /     -> pvz-game:8080
```

This keeps browser requests same-origin while the backend remains an independent Kubernetes Service.

### Backend persistence

The backend starts with:

- one replica
- `Recreate` deployment strategy
- one ReadWriteOnce PVC mounted at `/data`
- the cluster default StorageClass

This is intentional while the backend still uses SQLite, filesystem level storage and in-memory sessions.


## Staging online

The staging environment is reconciled by Flux from this repository and is publicly exposed through the existing Cloudflare Tunnel.

Public hostname:

```text
https://staging-pvz.obivan.org
```

Flux staging follows the repository `staging` branch. Production remains tied to promoted `main` state.

One-time cluster bootstrap and Cloudflare routing are documented in [docs/staging-online.md](docs/staging-online.md).
