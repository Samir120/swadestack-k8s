# SwadeStack Kubernetes Bootstrap and Disaster Recovery

This document describes how to rebuild the SwadeStack Kubernetes infrastructure from a fresh k3s installation.

The infrastructure is managed through GitOps using Argo CD.

---

## Architecture

The deployment flow is:

    GitHub
      |
      v
    GitHub Actions
      |
      v
    GHCR immutable commit-SHA images
      |
      v
    swadestack-k8s GitOps repository
      |
      v
    Argo CD
      |
      +--> Argo CD configuration
      +--> Traefik
      +--> Cloudflared
      +--> Reloader
      +--> Sealed Secrets
      +--> SwadeStack staging
      +--> SwadeStack production

The root Argo CD Application manages:

    root
    ├── argocd-config
    ├── cloudflared
    ├── reloader
    ├── sealed-secrets
    ├── traefik
    ├── swadestack-staging
    └── swadestack-production

---

## 1. Install and configure k3s

Expected k3s configuration:

File:

    /etc/rancher/k3s/config.yaml

Contents:

    node-name: homeserver
    node-ip: 192.168.0.100

    tls-san:
      - 192.168.0.100
      - homeserver

    disable:
      - traefik
      - servicelb

    write-kubeconfig-mode: "0640"
    write-kubeconfig-group: k3s-admin

Verify the Kubernetes node:

    kubectl get nodes

Expected:

    homeserver   Ready   control-plane

Verify the StorageClass:

    kubectl get storageclass

Expected default StorageClass:

    local-path

Expected volume binding mode:

    WaitForFirstConsumer

---

## 2. Clone the GitOps repository

Clone the repository:

    git clone https://github.com/Samir120/swadestack-k8s.git
    cd swadestack-k8s

---

## 3. Install pinned Argo CD

Argo CD is pinned to:

    v3.5.2

The bootstrap manifests are located at:

    infrastructure/argocd-install

Install Argo CD using server-side apply:

    kubectl apply --server-side -k infrastructure/argocd-install

Server-side apply is required because some Argo CD CRDs are too large for
the normal kubectl client-side last-applied-configuration annotation.

On an existing cluster that was previously installed using client-side
apply, server-side apply may report field-manager conflicts. Do not use
--force-conflicts automatically on a working cluster.

Wait for the main Argo CD components:

    kubectl -n argocd rollout status deployment/argocd-server
    kubectl -n argocd rollout status deployment/argocd-repo-server
    kubectl -n argocd rollout status statefulset/argocd-application-controller

Verify all pods:

    kubectl -n argocd get pods

Verify required CRDs:

    kubectl get crd \
      applications.argoproj.io \
      applicationsets.argoproj.io \
      appprojects.argoproj.io

---

## 4. Bootstrap the root Argo CD Application

Apply the root Application:

    kubectl apply -f argocd/root-application.yaml

Verify:

    kubectl -n argocd get application root

Expected:

    root   Synced   Healthy

List all Argo Applications:

    kubectl -n argocd get applications

Expected Applications:

    argocd-config
    cloudflared
    reloader
    root
    sealed-secrets
    swadestack-production
    swadestack-staging
    traefik

Wait until Applications become:

    Synced   Healthy

If the root Application has not detected a recent Git commit:

    kubectl -n argocd annotate application root \
      argocd.argoproj.io/refresh=hard \
      --overwrite

---

## 5. Verify Sealed Secrets controller

The Sealed Secrets controller is managed by Argo CD.

Pinned controller image:

    docker.io/bitnami/sealed-secrets-controller:0.39.1

Verify the Argo Application:

    kubectl -n argocd get application sealed-secrets

Expected:

    sealed-secrets   Synced   Healthy

Verify the Deployment:

    kubectl -n kube-system get deployment sealed-secrets-controller

Verify the image:

    kubectl -n kube-system get deployment sealed-secrets-controller \
      -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

Expected:

    docker.io/bitnami/sealed-secrets-controller:0.39.1

Verify the controller pod:

    kubectl -n kube-system get pods | grep sealed-secrets

---

## 6. Restore the Sealed Secrets private key

The Sealed Secrets controller private key is NOT stored in Git.

A secure off-machine backup must exist.

Example backup filename:

    sealed-secrets-controller-keys.yaml

Restore the backed-up key:

    kubectl apply -f /secure/path/sealed-secrets-controller-keys.yaml

Restart the controller so it reloads available keys:

    kubectl -n kube-system rollout restart deployment sealed-secrets-controller

Wait for the rollout:

    kubectl -n kube-system rollout status deployment sealed-secrets-controller

Verify controller keys:

    kubectl -n kube-system get secrets \
      -l sealedsecrets.bitnami.com/sealed-secrets-key

Never commit the Sealed Secrets private key to Git.

---

## 7. Verify SealedSecrets

Check staging:

    kubectl -n swadestack-staging get sealedsecret
    kubectl -n swadestack-staging get secret

Check production:

    kubectl -n swadestack-production get sealedsecret
    kubectl -n swadestack-production get secret

Application Secrets should be generated from the committed SealedSecret
resources.

Do not print secret values during normal verification.

---

## 8. Verify Argo CD configuration

The Argo CD ConfigMap is GitOps-managed by:

    argocd-config

Verify:

    kubectl -n argocd get application argocd-config

Expected:

    argocd-config   Synced   Healthy

Verify that argocd-cm is tracked by Argo CD:

    kubectl -n argocd get configmap argocd-cm \
      -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'

Expected:

    argocd-config:/ConfigMap:argocd/argocd-cm

Verify the custom Ingress health rule:

    kubectl -n argocd get configmap argocd-cm \
      -o jsonpath='{.data.resource\.customizations\.health\.networking\.k8s\.io_Ingress}{"\n"}'

The custom rule treats configured Ingress resources as Healthy even when
.status.loadBalancer is empty.

This is required because public traffic reaches Traefik through Cloudflare
Tunnel rather than through a Kubernetes LoadBalancer.

---

## 9. Verify Traefik

Verify Argo:

    kubectl -n argocd get application traefik

Expected:

    traefik   Synced   Healthy

Verify pods:

    kubectl get pods -n traefik

Verify service:

    kubectl get svc -n traefik

Traefik should use a ClusterIP Service.

k3s bundled Traefik and ServiceLB are disabled.

Public traffic does not require NodePort or LoadBalancer exposure.

---

## 10. Verify Cloudflare Tunnel

Verify Argo:

    kubectl -n argocd get application cloudflared

Expected:

    cloudflared   Synced   Healthy

Verify pods:

    kubectl get pods -n cloudflared

Cloudflare Tunnel forwards traffic internally to Traefik.

The expected architecture is:

    Internet
       |
       v
    Cloudflare
       |
       v
    Cloudflare Tunnel
       |
       v
    Traefik ClusterIP
       |
       v
    Kubernetes Ingress
       |
       +--> frontend
       +--> backend

No host port 80 or 443 exposure is required for the Kubernetes workloads.

---

## 11. Verify Reloader

Verify Argo:

    kubectl -n argocd get application reloader

Expected:

    reloader   Synced   Healthy

Verify:

    kubectl get pods -n reloader

The SwadeStack backend Deployment is annotated with:

    reloader.stakater.com/auto: "true"

This allows backend pods to roll automatically when referenced ConfigMaps
or Secrets change.

Verify staging:

    kubectl -n swadestack-staging get deployment backend \
      -o jsonpath='{.metadata.annotations.reloader\.stakater\.com/auto}{"\n"}'

Verify production:

    kubectl -n swadestack-production get deployment backend \
      -o jsonpath='{.metadata.annotations.reloader\.stakater\.com/auto}{"\n"}'

Expected:

    true

Reloader should not be used as a substitute for handling PostgreSQL
credential rotation correctly.

Changing POSTGRES_PASSWORD environment variables does not automatically
change the password stored inside an already initialized PostgreSQL
database.

---

## 12. Verify staging

Check Argo:

    kubectl -n argocd get application swadestack-staging

Expected:

    swadestack-staging   Synced   Healthy

Check workloads:

    kubectl get pods -n swadestack-staging

Expected main components:

    backend
    frontend
    postgres

Verify public API:

    curl https://lab.swadestack.com/api/health

Expected response contains:

    "success": true
    "message": "API is running"

---

## 13. Verify production

Check Argo:

    kubectl -n argocd get application swadestack-production

Expected:

    swadestack-production   Synced   Healthy

Check workloads:

    kubectl get pods -n swadestack-production

Expected main components:

    backend
    frontend
    postgres

Verify production API:

    curl https://swadestack.com/api/health

Expected response contains:

    "success": true
    "message": "API is running"

---

## 14. Verify production image versions

Check backend:

    kubectl -n swadestack-production get deployment backend \
      -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

Check frontend:

    kubectl -n swadestack-production get deployment frontend \
      -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

Backend and frontend should normally use the same promoted Git commit SHA.

Images follow this format:

    ghcr.io/samir120/swadestack-backend:<commit-sha>
    ghcr.io/samir120/swadestack-frontend:<commit-sha>

---

## 15. Staging deployment flow

Application changes follow this flow:

    push to application main
            |
            v
    GitHub Actions
            |
            +--> build backend image
            |
            +--> build frontend image
            |
            v
    GHCR
            |
            v
    immutable commit-SHA images
            |
            v
    staging GitOps overlay updated
            |
            v
    Argo CD
            |
            v
    lab.swadestack.com

The staging workflow updates only the staging overlay.

---

## 16. Production promotion flow

Production images are not rebuilt during promotion.

The deployment flow is:

    staging-tested SHA
            |
            v
    manual Promote workflow
            |
            v
    production GitOps overlay updated
            |
            v
    same backend/frontend image SHA
            |
            v
    Argo CD
            |
            v
    swadestack.com

The exact staging-tested SHA is promoted to production.

---

## 17. PostgreSQL persistent data

GitOps restores Kubernetes resources but does NOT restore PostgreSQL data.

PostgreSQL currently uses local persistent storage.

Verify PVCs:

    kubectl get pvc -n swadestack-staging
    kubectl get pvc -n swadestack-production

The PostgreSQL PVC normally follows a name similar to:

    postgres-data-postgres-0

Do not delete the production PostgreSQL PVC unless:

1. the data is intentionally disposable, or
2. a verified backup exists and a restore is planned.

With the local-path StorageClass and Delete reclaim policy, deleting a PVC
can permanently remove the corresponding persistent data.

---

## 18. Uploaded files

Application uploads are also persistent data and are NOT recreated by
GitOps.

Verify upload PVCs:

    kubectl get pvc -n swadestack-staging
    kubectl get pvc -n swadestack-production

The uploads volume normally follows a name similar to:

    swadestack-uploads

Uploaded files require a separate backup and restore procedure.

---

## 19. Required off-machine recovery material

The following must exist outside the Kubernetes server:

- Sealed Secrets controller private key backup
- PostgreSQL backups
- uploaded files backup
- GitHub repository access
- GHCR access
- Cloudflare account access
- Cloudflare Tunnel recovery information
- SSH access information
- k3s configuration
- DNS configuration knowledge

Do not store raw application secrets or Sealed Secrets private keys in Git.

---

## 20. Useful health commands

All Argo Applications:

    kubectl -n argocd get applications

Argo pods:

    kubectl -n argocd get pods

Staging:

    kubectl get pods -n swadestack-staging

Production:

    kubectl get pods -n swadestack-production

Traefik:

    kubectl get pods -n traefik

Cloudflared:

    kubectl get pods -n cloudflared

Reloader:

    kubectl get pods -n reloader

Sealed Secrets:

    kubectl -n kube-system get deployment sealed-secrets-controller

Public health:

    curl https://lab.swadestack.com/api/health
    curl https://swadestack.com/api/health

---

## 21. GitOps repository validation

Render staging:

    kubectl kustomize apps/swadestack/overlays/staging >/dev/null

Render production:

    kubectl kustomize apps/swadestack/overlays/production >/dev/null

Render Argo Applications:

    kubectl kustomize argocd/applications >/dev/null

Render Argo bootstrap:

    kubectl kustomize infrastructure/argocd-install >/dev/null

Render Sealed Secrets:

    kubectl kustomize infrastructure/sealed-secrets >/dev/null

A successful command should exit with status 0.

---

## 22. Full disaster-recovery order

Use this order for a complete rebuild:

    Fresh Ubuntu Server
            |
            v
    Configure networking and SSH
            |
            v
    Install/configure k3s
            |
            v
    Verify node and local-path StorageClass
            |
            v
    Clone swadestack-k8s
            |
            v
    Install pinned Argo CD v3.5.2
            |
            v
    Apply root Application
            |
            v
    Wait for sealed-secrets controller
            |
            v
    Restore Sealed Secrets private key
            |
            v
    Restart sealed-secrets controller
            |
            v
    Verify SealedSecrets decrypt
            |
            v
    Verify Traefik
            |
            v
    Verify Cloudflared
            |
            v
    Verify Reloader
            |
            v
    Restore PostgreSQL data
            |
            v
    Restore uploaded files
            |
            v
    Verify staging
            |
            v
    Verify production

---

## 23. Final recovery verification

Run:

    kubectl get nodes

    kubectl -n argocd get applications

    kubectl get pods -n traefik

    kubectl get pods -n cloudflared

    kubectl get pods -n reloader

    kubectl -n kube-system get deployment sealed-secrets-controller

    kubectl get pods -n swadestack-staging

    kubectl get pods -n swadestack-production

    curl https://lab.swadestack.com/api/health

    curl https://swadestack.com/api/health

The recovery is complete when:

- the Kubernetes node is Ready
- all Argo Applications are Synced and Healthy
- infrastructure pods are Running
- staging workloads are Running
- production workloads are Running
- both public health endpoints succeed
- restored application data has been verified
