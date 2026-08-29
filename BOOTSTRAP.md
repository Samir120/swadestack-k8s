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

---

## 24. Production backup architecture

Production persistent data is backed up independently from GitOps.

GitOps can recreate Kubernetes resources, but it cannot recreate PostgreSQL
records or uploaded files. The backup flow is therefore:

    Production PostgreSQL PVC
            |
            v
    postgres-backup CronJob
            |
            v
    swadestack-backups PVC
            |
            +----------------------+
            |                      |
            v                      v
    database/*.dump        uploads/*.tar.gz
            |                      |
            +----------+-----------+
                       |
                       v
             homeserver export
                       |
                       v
        /home/samir/swadestack-backup-export
                       |
                       v
                 rsync over SSH
                       |
                       v
                off-machine PC
                       |
                       v
    ~/.config/swadestack-backups/production

The current backup schedule is:

    02:00  PostgreSQL backup
    02:15  uploaded-files backup
    03:00  export Kubernetes backups to homeserver filesystem
    03:15  pull homeserver backups to the PC
    06:00  homeserver backup-freshness check
    06:15  PC off-machine backup-freshness check

All times are intended to use Europe/Stockholm local time.

The Kubernetes backup PVC is:

    swadestack-backups

Namespace:

    swadestack-production

StorageClass:

    local-path

Requested size:

    10Gi

Do not treat the backup PVC as an off-machine backup. It is stored on the
same Kubernetes host as the application data.

---

## 25. Kubernetes PostgreSQL backups

The production PostgreSQL backup CronJob is:

    postgres-backup

Verify it:

    kubectl -n swadestack-production get cronjob postgres-backup

Inspect its schedule:

    kubectl -n swadestack-production get cronjob postgres-backup \
      -o jsonpath='{.spec.schedule}{"\n"}'

Expected schedule:

    0 2 * * *

Verify timezone:

    kubectl -n swadestack-production get cronjob postgres-backup \
      -o jsonpath='{.spec.timeZone}{"\n"}'

Expected:

    Europe/Stockholm

The job writes custom-format PostgreSQL dumps under:

    /backups/database

inside the backup PVC.

Backup filenames follow:

    swadestack-YYYY-MM-DD_HH-MM-SS.dump

The backup job validates each dump with:

    pg_restore -l

Backups older than approximately 14 days are removed by the CronJob.

Check recent PostgreSQL backup jobs:

    kubectl -n swadestack-production get jobs \
      -l job-name \
      --sort-by=.metadata.creationTimestamp

A simpler general check is:

    kubectl -n swadestack-production get jobs \
      --sort-by=.metadata.creationTimestamp

Inspect the most recent backup job logs by first finding the job:

    kubectl -n swadestack-production get jobs \
      --sort-by=.metadata.creationTimestamp

Then run:

    kubectl -n swadestack-production logs job/<postgres-backup-job-name>

A successful backup should contain a message similar to:

    Backup verified successfully.

---

## 26. Kubernetes uploaded-files backups

The production uploads backup CronJob is:

    uploads-backup

Verify it:

    kubectl -n swadestack-production get cronjob uploads-backup

Inspect its schedule:

    kubectl -n swadestack-production get cronjob uploads-backup \
      -o jsonpath='{.spec.schedule}{"\n"}'

Expected:

    15 2 * * *

Verify timezone:

    kubectl -n swadestack-production get cronjob uploads-backup \
      -o jsonpath='{.spec.timeZone}{"\n"}'

Expected:

    Europe/Stockholm

The job archives the production uploads PVC into:

    /backups/uploads

Backup filenames follow:

    uploads-YYYY-MM-DD_HH-MM-SS.tar.gz

The source uploads volume is mounted read-only by the backup job.

The job validates each archive with:

    tar -tzf

Backups older than approximately 14 days are removed by the CronJob.

Inspect the latest uploads backup job:

    kubectl -n swadestack-production get jobs \
      --sort-by=.metadata.creationTimestamp

Then:

    kubectl -n swadestack-production logs job/<uploads-backup-job-name>

A successful backup should contain a message similar to:

    Backup verified successfully.

---

## 27. Inspect the Kubernetes backup PVC

Verify the backup PVC:

    kubectl -n swadestack-production get pvc swadestack-backups

Expected state:

    Bound

To inspect the backup files without modifying the PVC, create a temporary
helper pod.

Example:

    cat <<'EOF' | kubectl apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: backup-inspector
      namespace: swadestack-production
    spec:
      restartPolicy: Never
      containers:
        - name: inspector
          image: postgres:16-alpine
          command:
            - sh
            - -c
            - sleep 3600
          volumeMounts:
            - name: backups
              mountPath: /backups
              readOnly: true
      volumes:
        - name: backups
          persistentVolumeClaim:
            claimName: swadestack-backups
    EOF

Wait for it:

    kubectl -n swadestack-production wait \
      --for=condition=Ready \
      pod/backup-inspector \
      --timeout=120s

List database backups:

    kubectl -n swadestack-production exec backup-inspector -- \
      sh -c 'ls -lh /backups/database'

List uploads backups:

    kubectl -n swadestack-production exec backup-inspector -- \
      sh -c 'ls -lh /backups/uploads'

Validate all database dumps:

    kubectl -n swadestack-production exec backup-inspector -- \
      sh -c '
        set -e
        for f in /backups/database/*.dump; do
          echo "Checking $f"
          pg_restore -l "$f" >/dev/null
        done
      '

Validate all uploads archives:

    kubectl -n swadestack-production exec backup-inspector -- \
      sh -c '
        set -e
        for f in /backups/uploads/*.tar.gz; do
          echo "Checking $f"
          tar -tzf "$f" >/dev/null
        done
      '

Delete the helper pod afterward:

    kubectl -n swadestack-production delete pod backup-inspector

Do not modify files directly inside the k3s local-path storage directory.

---

## 28. Homeserver backup export

The Kubernetes backup PVC is exported to the homeserver filesystem by:

    /usr/local/sbin/export-swadestack-backups

The systemd service is:

    swadestack-backup-export.service

The timer is:

    swadestack-backup-export.timer

The export destination is:

    /home/samir/swadestack-backup-export

Expected directories:

    /home/samir/swadestack-backup-export/database
    /home/samir/swadestack-backup-export/uploads

A successful run updates:

    /home/samir/swadestack-backup-export/.last-success

Verify the service:

    systemctl status swadestack-backup-export.service --no-pager

For a oneshot service, this is normal after a successful run:

    Active: inactive (dead)
    status=0/SUCCESS

Verify the timer:

    systemctl list-timers swadestack-backup-export.timer --all

Run a manual export:

    sudo systemctl start swadestack-backup-export.service

Check logs:

    journalctl -u swadestack-backup-export.service \
      --since "30 minutes ago" \
      --no-pager

Verify the success marker:

    stat /home/samir/swadestack-backup-export/.last-success

The marker must only be touched after a successful export.

---

## 29. Off-machine PC backup pull

The PC pulls exported backups from the homeserver using rsync over SSH.

SSH uses port:

    7222

The user-side script is:

    ~/.local/bin/pull-swadestack-backups

The user systemd service is:

    swadestack-backup-pull.service

The user timer is:

    swadestack-backup-pull.timer

The local destination is:

    ~/.config/swadestack-backups/production

Expected directories:

    ~/.config/swadestack-backups/production/database
    ~/.config/swadestack-backups/production/uploads

A successful pull updates:

    ~/.config/swadestack-backups/production/.last-success

The service should use:

    UMask=0077

This keeps newly created backup-related files private.

Verify passwordless SSH before relying on the timer:

    ssh -p 7222 \
      -o BatchMode=yes \
      samir@192.168.0.100 \
      true

The command should complete without prompting for a password.

Run a manual pull:

    systemctl --user start swadestack-backup-pull.service

Check status:

    systemctl --user status \
      swadestack-backup-pull.service \
      --no-pager

For a successful oneshot user service, this is normal:

    Active: inactive (dead)
    status=0/SUCCESS

Check logs:

    journalctl --user \
      -u swadestack-backup-pull.service \
      --since "30 minutes ago" \
      --no-pager

Verify downloaded files:

    find ~/.config/swadestack-backups/production \
      -type f \
      -exec ls -lh {} \;

Verify the success marker:

    stat ~/.config/swadestack-backups/production/.last-success

The PC user manager must be allowed to run without an interactive login.

Enable lingering:

    sudo loginctl enable-linger samir

Verify:

    loginctl show-user samir -p Linger

Expected:

    Linger=yes

The current PC-side retention policy removes local database and uploads backup
files older than approximately 30 days.

---

## 30. Backup freshness monitoring

The homeserver checks whether the export success marker is recent enough.

Script:

    /usr/local/sbin/check-swadestack-backup-freshness

Service:

    swadestack-backup-freshness.service

Timer:

    swadestack-backup-freshness.timer

Expected timer time:

    06:00

The current freshness threshold is approximately:

    26 hours

Run the check manually:

    sudo systemctl start swadestack-backup-freshness.service

Inspect:

    systemctl status \
      swadestack-backup-freshness.service \
      --no-pager

A healthy result contains:

    SwadeStack backup export is fresh:

The PC performs the same check against the off-machine success marker.

Script:

    ~/.local/bin/check-swadestack-backup-freshness

User service:

    swadestack-backup-freshness.service

User timer:

    swadestack-backup-freshness.timer

Expected timer time:

    06:15

Run manually:

    systemctl --user start swadestack-backup-freshness.service

Inspect:

    systemctl --user status \
      swadestack-backup-freshness.service \
      --no-pager

A healthy result contains:

    Off-machine SwadeStack backup is fresh:

Verify all related timers on the homeserver:

    systemctl list-timers \
      swadestack-backup-export.timer \
      swadestack-backup-freshness.timer \
      --all

Verify all related timers on the PC:

    systemctl --user list-timers \
      swadestack-backup-pull.timer \
      swadestack-backup-freshness.timer \
      --all

---

## 31. Backup failure email alerts

Backup monitoring uses msmtp to deliver infrastructure alert email.

Do not store SMTP credentials in Git.

Homeserver msmtp configuration:

    /etc/msmtprc

Recommended permissions:

    sudo chmod 600 /etc/msmtprc

The configuration must use a dedicated infrastructure SMTP credential.

Example structure:

    defaults
    auth on
    tls on
    tls_starttls on
    syslog LOG_MAIL

    account brevo
    host smtp-relay.brevo.com
    port 587
    from alerts@swadestack.com
    user YOUR_BREVO_SMTP_LOGIN
    password YOUR_DEDICATED_INFRA_SMTP_KEY

    account default : brevo

Test the homeserver SMTP configuration:

    printf 'Subject: SwadeStack monitoring test\n\nSMTP test\n' \
      | sudo msmtp YOUR_ALERT_EMAIL_ADDRESS

Check for successful delivery:

    journalctl --since "5 minutes ago" \
      | grep 'msmtp.*smtpstatus=250'

The homeserver alert script is:

    /usr/local/sbin/swadestack-alert

The systemd template is:

    /etc/systemd/system/swadestack-alert@.service

The export and freshness services use:

    OnFailure=swadestack-alert@%n

This means failures in either service automatically trigger an email.

The PC uses its own private msmtp configuration:

    ~/.msmtprc

Recommended permissions:

    chmod 600 ~/.msmtprc

Do not rely on the root-only `/etc/msmtprc` from a systemd user service.

Test PC user SMTP delivery without sudo:

    printf 'Subject: SwadeStack PC monitoring test\n\nSMTP test\n' \
      | msmtp YOUR_ALERT_EMAIL_ADDRESS

The PC alert script is:

    ~/.local/bin/swadestack-backup-alert

Recommended permissions:

    chmod 700 ~/.local/bin/swadestack-backup-alert

The user service template is:

    ~/.config/systemd/user/swadestack-alert@.service

Both of these PC services should contain an OnFailure dependency:

    swadestack-backup-pull.service
    swadestack-backup-freshness.service

Expected:

    OnFailure=swadestack-alert@%n

Never commit:

- `/etc/msmtprc`
- `~/.msmtprc`
- SMTP passwords or API keys
- Sealed Secrets controller private keys
- raw production Secrets

---

## 32. Controlled backup-alert test

The alert path can be tested without damaging actual backup data.

On the homeserver, temporarily move the success marker:

    sudo mv \
      /home/samir/swadestack-backup-export/.last-success \
      /home/samir/swadestack-backup-export/.last-success.test

Run the freshness service:

    sudo systemctl start swadestack-backup-freshness.service || true

Verify that it failed:

    systemctl status \
      swadestack-backup-freshness.service \
      --no-pager

Expected output includes:

    ERROR: SwadeStack backup success marker does not exist.
    Triggering OnFailure= dependencies.

An email alert should be delivered automatically.

Immediately restore the marker:

    sudo mv \
      /home/samir/swadestack-backup-export/.last-success.test \
      /home/samir/swadestack-backup-export/.last-success

Reset the intentional systemd failure:

    sudo systemctl reset-failed swadestack-backup-freshness.service

Run the healthy check again:

    sudo systemctl start swadestack-backup-freshness.service

Repeat the same pattern on the PC if required:

    mv \
      ~/.config/swadestack-backups/production/.last-success \
      ~/.config/swadestack-backups/production/.last-success.test

    systemctl --user start swadestack-backup-freshness.service || true

    systemctl --user status \
      swadestack-backup-freshness.service \
      --no-pager

    mv \
      ~/.config/swadestack-backups/production/.last-success.test \
      ~/.config/swadestack-backups/production/.last-success

    systemctl --user reset-failed swadestack-backup-freshness.service

    systemctl --user start swadestack-backup-freshness.service

---

## 33. Restore PostgreSQL from an automated backup

A backup is not considered sufficient until it can be restored.

Prefer testing restores into a temporary database before using the procedure on
production.

First identify the newest dump.

On the PC:

    ls -1t \
      ~/.config/swadestack-backups/production/database/*.dump \
      | head -1

Or on the homeserver:

    ls -1t \
      /home/samir/swadestack-backup-export/database/*.dump \
      | head -1

Before a production restore:

1. stop or scale down application writers
2. preserve the current database with an additional emergency dump
3. verify the selected backup using `pg_restore -l`
4. restore into a temporary database when possible
5. compare important table counts
6. only then replace production data

A safe validation command on a host with PostgreSQL client tools is:

    pg_restore -l /path/to/swadestack-YYYY-MM-DD_HH-MM-SS.dump >/dev/null

A zero exit status means PostgreSQL can read the archive structure.

### Temporary restore validation inside Kubernetes

Copy the selected dump into a temporary PostgreSQL pod or other controlled
restore environment.

Do not restore over the live production database merely to test a backup.

Example high-level process:

    BACKUP=/path/to/latest.dump

    pg_restore -l "$BACKUP"

Create a temporary database:

    createdb swadestack_restore_test

Restore:

    pg_restore \
      --no-owner \
      --no-privileges \
      --dbname=swadestack_restore_test \
      "$BACKUP"

Verify schema and important application data:

    psql \
      --dbname=swadestack_restore_test \
      -c '\dt'

Then compare important tables such as:

    Orders
    OrderItems
    Payments
    Emails
    LoginAttempts

After successful validation:

    dropdb swadestack_restore_test

The exact connection parameters depend on where the restore test is performed.
Do not place PostgreSQL passwords directly into shell history.

### Production database restore

For an actual disaster recovery, ensure the replacement PostgreSQL instance is
available and application writers are stopped.

Validate the chosen backup:

    pg_restore -l /secure/path/latest.dump >/dev/null

Restore it using the credentials and connection information from the recovered
Kubernetes Secret.

A typical pattern is:

    pg_restore \
      --clean \
      --if-exists \
      --no-owner \
      --no-privileges \
      --dbname="$DATABASE_URL" \
      /secure/path/latest.dump

Do not run `--clean` against the live production database unless the restore
has been explicitly planned and current data has been preserved.

After the restore:

    kubectl -n swadestack-production get pods

    kubectl -n argocd get application swadestack-production

    curl https://swadestack.com/api/health

Then verify real application data through the application itself.

---

## 34. Restore uploaded files

Uploaded files are restored independently from PostgreSQL.

Identify the newest archive.

On the PC:

    ls -1t \
      ~/.config/swadestack-backups/production/uploads/*.tar.gz \
      | head -1

On the homeserver:

    ls -1t \
      /home/samir/swadestack-backup-export/uploads/*.tar.gz \
      | head -1

Validate before restoring:

    tar -tzf /path/to/uploads-YYYY-MM-DD_HH-MM-SS.tar.gz >/dev/null

A zero exit status means the gzip/tar archive is readable.

Before restoring production uploads:

1. stop application processes that may write uploaded files
2. verify the target PVC is correct
3. preserve existing files if they may contain newer data
4. extract only the selected verified archive
5. verify file count and application access

Never blindly extract into an unknown host path.

The target Kubernetes PVC is:

    swadestack-uploads

Namespace:

    swadestack-production

A temporary restore helper can mount both the uploads PVC and a source backup
location. The restore procedure should clear or replace production uploads only
when a full replacement is intended.

After restoring, verify the mounted application path:

    kubectl -n swadestack-production exec deployment/backend -- \
      sh -c 'du -sh /app/uploads && find /app/uploads -type f | wc -l'

Then verify one or more known uploaded files through the public application.

---

## 35. Complete backup verification checklist

Use this checklist periodically and after infrastructure changes.

    kubectl -n swadestack-production get cronjob postgres-backup
    kubectl -n swadestack-production get cronjob uploads-backup
    kubectl -n swadestack-production get pvc swadestack-backups

    systemctl list-timers \
      swadestack-backup-export.timer \
      swadestack-backup-freshness.timer \
      --all

    systemctl --user list-timers \
      swadestack-backup-pull.timer \
      swadestack-backup-freshness.timer \
      --all

    stat /home/samir/swadestack-backup-export/.last-success

    stat ~/.config/swadestack-backups/production/.last-success

    find ~/.config/swadestack-backups/production/database \
      -type f \
      -exec ls -lh {} \;

    find ~/.config/swadestack-backups/production/uploads \
      -type f \
      -exec ls -lh {} \;

A healthy system should have:

- recent successful Kubernetes backup Jobs
- a Bound `swadestack-backups` PVC
- recent database and uploads backup files
- a recent homeserver export success marker
- a recent PC off-machine success marker
- enabled systemd timers
- a working email-alert path
- a database backup that has been restore-tested
- an uploads archive that has been extraction-tested

---

## 36. Disaster recovery with backup data

For a complete host loss, use this order:

    Fresh Ubuntu Server
            |
            v
    Configure networking and SSH
            |
            v
    Install and configure k3s
            |
            v
    Clone swadestack-k8s
            |
            v
    Install pinned Argo CD
            |
            v
    Apply root Application
            |
            v
    Wait for Sealed Secrets controller
            |
            v
    Restore Sealed Secrets controller private key
            |
            v
    Restart Sealed Secrets controller
            |
            v
    Verify application Secrets decrypt
            |
            v
    Verify Traefik / Cloudflared / Reloader
            |
            v
    Wait for production PostgreSQL and uploads PVCs
            |
            v
    Stop production application writers
            |
            v
    Restore PostgreSQL from verified off-machine backup
            |
            v
    Restore uploaded files from verified off-machine backup
            |
            v
    Start / sync production workloads
            |
            v
    Verify database records
            |
            v
    Verify uploaded files
            |
            v
    Verify public API and frontend
            |
            v
    Recreate homeserver backup-export services
            |
            v
    Recreate PC backup-pull services if required
            |
            v
    Test backup freshness and email alerts

Do not delete the old source data, emergency backups, or previous recovery
material until the recovered production system has been validated.

---

## 37. Files that are intentionally outside Git

The following operational files contain machine-specific configuration or
credentials and must not be committed to the GitOps repository.

Homeserver examples:

    /etc/rancher/k3s/config.yaml
    /etc/msmtprc
    /usr/local/sbin/export-swadestack-backups
    /usr/local/sbin/check-swadestack-backup-freshness
    /usr/local/sbin/swadestack-alert
    /etc/systemd/system/swadestack-backup-export.service
    /etc/systemd/system/swadestack-backup-export.timer
    /etc/systemd/system/swadestack-backup-freshness.service
    /etc/systemd/system/swadestack-backup-freshness.timer
    /etc/systemd/system/swadestack-alert@.service

PC examples:

    ~/.msmtprc
    ~/.local/bin/pull-swadestack-backups
    ~/.local/bin/check-swadestack-backup-freshness
    ~/.local/bin/swadestack-backup-alert
    ~/.config/systemd/user/swadestack-backup-pull.service
    ~/.config/systemd/user/swadestack-backup-pull.timer
    ~/.config/systemd/user/swadestack-backup-freshness.service
    ~/.config/systemd/user/swadestack-backup-freshness.timer
    ~/.config/systemd/user/swadestack-alert@.service

The Sealed Secrets controller private-key backup must also remain outside Git.

Keep secure off-machine copies of the configuration needed to reconstruct
these files.

---

## 38. Final operational state

The SwadeStack infrastructure is considered recoverable when all of the
following are true:

- the GitOps repository can recreate the Kubernetes objects
- Argo CD and all child Applications become Synced and Healthy
- the Sealed Secrets controller key is available off-machine
- PostgreSQL backups exist off-machine
- uploaded-files backups exist off-machine
- PostgreSQL restore has been tested
- uploaded-files restore has been tested
- backup export and off-machine pull timers are enabled
- freshness timers are enabled
- backup failures produce email alerts
- public staging and production health checks succeed
- important production records and uploaded files have been verified

The backup design should be reviewed whenever storage, database topology,
hosting hardware, SSH access, Cloudflare configuration, or secret-management
strategy changes.

