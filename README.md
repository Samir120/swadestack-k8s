# SwadeStack Kubernetes / GitOps

This repository contains the Kubernetes and GitOps configuration for SwadeStack.

Application source code lives in:
https://github.com/Samir120/swadestack

## Repository structure

- `apps/` - application Kubernetes manifests and environment overlays
- `argocd/` - Argo CD applications and bootstrap configuration
- `infrastructure/` - shared cluster infrastructure
- `BOOTSTRAP.md` - cluster bootstrap and recovery procedure

Application backend/frontend source code must not be committed to this repository.
