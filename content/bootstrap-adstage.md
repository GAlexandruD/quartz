# Bootstrap adstage

Single-node k3s cluster (Ubuntu, AMD A4-6210). Staging cluster. New images
are tested here before being promoted to maxipi. Services are exposed via
Cloudflare Tunnels; no ports are opened on the host. Uses Calico instead of
Flannel for NetworkPolicy support.

## 1. Install k3s

```sh
# Remove existing installation if present
/usr/local/bin/k3s-uninstall.sh

# Install with Flannel disabled
curl -sfL https://get.k3s.io | sh -s - server \
    --cluster-init \
    --disable=helm-controller \
    --flannel-backend=none \
    --disable-network-policy
```

## 2. Install Calico (before Flux, CNI must exist for pods to run)

```sh
# Install Tigera operator. Version must match infrastructure/controllers/adstage/calico/release.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml

# Apply Installation CR
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - cidr: 10.42.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
EOF

# Wait for Calico to be ready
kubectl wait --for=condition=Ready pods --all -n calico-system --timeout=120s
```

## 3. Bootstrap Flux

```sh
flux bootstrap github \
    --owner=GAlexandruD \
    --repository=Homelab \
    --branch=main \
    --path=clusters/adstage \
    --personal
```

## 4. Create the Infisical credentials secret and SOPS Age key

Log in to eu.infisical.com → adstage organization → Machine Identities →
`adstage` → Authentication → Add Client Secret.

```sh
kubectl create namespace external-secrets
kubectl create secret generic infisical-credentials \
    --from-literal=clientId=<your-client-id> \
    --from-literal=clientSecret=<your-client-secret> \
    -n external-secrets
```

Create the SOPS Age key (required for existing apps that use SOPS-encrypted secrets):

```sh
# Option A: paste the key directly
kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-literal=age.agekey=AGE-SECRET-KEY-XXXXX

# Option B: read from a key file
cat age.agekey | kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-file=age.agekey=/dev/stdin
```

Force an immediate reconcile so Flux picks up the new secrets:

```sh
flux reconcile kustomization flux-system --with-source
```

## 5. Fix inotify limits

```sh
cat >> /etc/sysctl.d/99-inotify.conf <<EOF
fs.inotify.max_user_instances=512
fs.inotify.max_user_watches=524288
EOF
sysctl -p /etc/sysctl.d/99-inotify.conf
```

## 6. Velero - create the MinIO bucket

Wait for MinIO to be running (`kubectl get pods -n minio`), then:

```sh
kubectl exec -n minio deploy/minio -- sh -c \
  'mc alias set local http://localhost:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD && mc mb local/velero'
```

Check Velero can reach MinIO:

```sh
kubectl get backupstoragelocation -n velero
# NAME      PHASE       DEFAULT
# default   Available   true
```

## 7. Restore from Cloudflare R2 backup (replaces steps 8 and 9)

> If this is a fresh cluster with no prior backup, skip to step 8.

Restoring from R2 brings back Grafana's SQLite database (including the public
dashboard configuration) and correct PVC file ownership. Steps 8 and 9 are
not needed after a successful restore.

Wait until `kubectl get externalsecret -n velero` shows `rclone-credentials`
as `SecretSynced` before continuing.

### Copy R2 → MinIO

Run a one-off rclone job that reverses the nightly sync direction:

```sh
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: rclone-r2-to-minio
  namespace: velero
spec:
  template:
    spec:
      restartPolicy: OnFailure
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
        - name: rclone
          image: rclone/rclone:1.74.2
          command:
            - rclone
            - copy
            - r2:$(R2_BUCKET)
            - minio:velero
            - --progress
          env:
            - name: HOME
              value: /tmp
            - name: RCLONE_CONFIG_MINIO_TYPE
              value: s3
            - name: RCLONE_CONFIG_MINIO_PROVIDER
              value: Minio
            - name: RCLONE_CONFIG_MINIO_ENDPOINT
              value: http://minio.minio.svc.cluster.local:9000
            - name: RCLONE_CONFIG_MINIO_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: rclone-credentials
                  key: MINIO_ACCESS_KEY_ID
            - name: RCLONE_CONFIG_MINIO_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: rclone-credentials
                  key: MINIO_SECRET_ACCESS_KEY
            - name: RCLONE_CONFIG_R2_TYPE
              value: s3
            - name: RCLONE_CONFIG_R2_PROVIDER
              value: Cloudflare
            - name: RCLONE_CONFIG_R2_REGION
              value: auto
            - name: RCLONE_CONFIG_R2_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: rclone-credentials
                  key: R2_ACCESS_KEY_ID
            - name: RCLONE_CONFIG_R2_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: rclone-credentials
                  key: R2_SECRET_ACCESS_KEY
            - name: RCLONE_CONFIG_R2_ENDPOINT
              valueFrom:
                secretKeyRef:
                  name: rclone-credentials
                  key: R2_ENDPOINT
            - name: R2_BUCKET
              valueFrom:
                secretKeyRef:
                  name: rclone-credentials
                  key: R2_BUCKET
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            seccompProfile:
              type: RuntimeDefault
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
EOF

kubectl logs -n velero -l job-name=rclone-r2-to-minio -f
```

### Restore

Wait for Velero to detect the backups, then restore:

```sh
# List available backups
kubectl exec -n velero deploy/velero -- /velero get backups

# Scale down Grafana before restore so kopia can write into the PVC cleanly
kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=0

# Restore and exclude namespaces Flux manages to avoid conflicts
kubectl exec -n velero deploy/velero -- /velero restore create \
  --from-backup <backup-name> \
  --exclude-namespaces flux-system,velero,minio,external-secrets \
  --existing-resource-policy=update \
  --wait
```

Grafana will come back up on its own once Flux reconciles. The public
dashboard will be present without any manual `curl` call. Clean up:

```sh
kubectl delete job rclone-r2-to-minio -n velero
```

## 8. Fix Grafana PVC ownership (skip if restored from backup)

The local-path provisioner creates the Grafana PVC directory owned by root.
Grafana (UID 472) cannot write to it and will crash on first start.

```sh
kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=0

PV=$(kubectl get pvc -n monitoring kube-prometheus-stack-grafana -o jsonpath='{.spec.volumeName}')
PV_PATH=$(kubectl get pv $PV -o jsonpath='{.spec.local.path}')
chown -R 472:472 $PV_PATH

kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=1
```

## 9. Recreate the public Grafana dashboard (skip if restored from backup)

The public dashboard UID and access token are fixed values so the public URL
stays stable across cluster recreations.

```sh
export GRAFANA_PASS=$(kubectl get secret grafana-admin-secret -n monitoring -o jsonpath='{.data.adminPassword}' | base64 -d)
export GRAFANA_USER=$(kubectl get secret grafana-admin-secret -n monitoring -o jsonpath='{.data.adminUser}' | base64 -d)
curl -X POST https://grafana-adstage.alexandrud.com/api/dashboards/uid/42f78c614ade459cabf8c235e2b14819/public-dashboards \
    -H "Content-Type: application/json" \
    -u $GRAFANA_USER:$GRAFANA_PASS \
    -d '{"uid": "42f78c614ade459cabf8c235e2b14819", "isEnabled": true, "accessToken": "42f78c614ade459cabf8c235e2b14819"}'
```

## 10. Trigger the first Quartz build

Quartz builds are triggered by a CronJob. On a fresh cluster the first build
must be started manually, otherwise the site won't appear until the next
scheduled run.

```sh
kubectl create job quartz-build-init --from=cronjob/quartz-build -n quartz
# To rebuild after pushing new content:
# kubectl create job quartz-build-$(date +%s) --from=cronjob/quartz-build -n quartz
```
