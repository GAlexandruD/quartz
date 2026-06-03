# Bootstrap maxipi

Single-node k3s cluster (Ubuntu, Celeron J3455). Runs on the local network;
accessible externally via Tailscale. Uses Calico instead of Flannel for
NetworkPolicy support.

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
# Install Tigera operator. Version must match infrastructure/controllers/maxipi/calico/release.yaml
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
    --path=clusters/maxipi \
    --personal
```

## 4. Create the Infisical credentials secret

Log in to eu.infisical.com → maxipi organization → Machine Identities →
`maxipi` → Authentication → Add Client Secret.

```sh
kubectl create namespace external-secrets
kubectl create secret generic infisical-credentials \
    --from-literal=clientId=<your-client-id> \
    --from-literal=clientSecret=<your-client-secret> \
    -n external-secrets
```

> ESO will now sync all secrets from Infisical. Verify the following secrets
> exist in the maxipi Infisical project before proceeding:
> - `/minio/ROOT_USER`, `/minio/ROOT_PASSWORD`
> - `/velero/KOOFR_ACCESS_KEY_ID`, `/velero/KOOFR_SECRET_ACCESS_KEY`
> - `/velero/GDRIVE_TOKEN` (full OAuth JSON from `rclone authorize "drive"`)

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
# NAME    PHASE       DEFAULT
# local   Available   true
```

Trigger a test backup:

```sh
kubectl create -f - <<EOF
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: test-backup
  namespace: velero
spec:
  storageLocation: local
  includedNamespaces:
    - minio
  defaultVolumesToFsBackup: true
EOF

kubectl get backup test-backup -n velero -w
```

## 7. Velero - test offsite rclone sync

Before the 3am scheduled run, trigger manually to verify Koofr and Google
Drive are reachable. Requires the `velero-maxipi` folder to exist in both
Koofr and Google Drive beforehand.

```sh
kubectl create job -n velero --from=cronjob/rclone-offsite-sync rclone-test
kubectl logs -n velero -l job-name=rclone-test -f
```

Confirm backup chunks appear in Koofr `velero-maxipi` and Google Drive
`velero-maxipi`.
