# Chapter 11: Maintenance and Backup

A production-grade cluster requires a multi-layer backup strategy. We follow the **Four-Layer Security Model** to ensure that even in a total site failure, we can recover our cluster, data, and applications.

---

## 1. Layer 1: K3s etcd Snapshots (Cluster State)

The `etcd` database stores all Kubernetes objects (Deployments, Secrets, ConfigMaps). Protecting this is the first priority.

### Automatic S3 Snapshots
K3s can natively take snapshots and upload them to S3-compatible storage.

**Configuration (K3s Config):**
Add these to `/etc/rancher/k3s/config.yaml` on your master nodes:
```yaml
etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 10
etcd-s3: true
etcd-s3-endpoint: "<S3_ENDPOINT_URL>"
etcd-s3-bucket: "k3s-backups"
etcd-s3-access-key: "<ACCESS_KEY>"
etcd-s3-secret-key: "<SECRET_KEY>"
```

**Manual Snapshot:**
```bash
sudo k3s etcd-snapshot save --s3
```

---

## 2. Layer 2: Longhorn Backups (Volume Data)

Longhorn protects your Persistent Volumes (PVs). While replicas protect against disk failure, backups protect against data corruption or accidental deletion.

### Configuration
1.  **Backup Target**: In Longhorn UI, set the `Backup Target` to `s3://<bucket>@<region>/`.
2.  **Backup Secret**: Create a secret in `longhorn-system` with S3 credentials.
3.  **Recurring Jobs**: Create a `RecurringJob` in the Longhorn UI to automate snapshots and backups daily.

---

## 3. Layer 3: Velero (Application-Aware Backups)

Velero is the industry standard for backing up the entire cluster, including both YAML manifests and volume data.

### Installation
```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.9.0 \
    --bucket k3s-velero-backups \
    --secret-file ./credentials-velero \
    --use-node-agent \
    --uploader-type restic \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=<S3_URL>
```

---

## 4. Layer 4: CloudNative PG (Database Consistency)

Databases require "application-consistent" backups. CNPG uses **Barman Cloud** to stream WAL logs to S3.

### Implementation
Add the `backup` section to your `Cluster` manifest (Chapter 10):
```yaml
spec:
  backup:
    barmanObjectStore:
      destinationPath: s3://postgres-backups/
      endpointURL: <S3_URL>
      s3Credentials:
        accessKeyId:
          name: s3-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: s3-creds
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
```

---

## 5. Disaster Recovery (DR)

### Restoring K3s from etcd Snapshot
If the control plane is lost, re-initialize the cluster using a snapshot:
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --cluster-reset \
  --cluster-reset-restore-path=<PATH_TO_SNAPSHOT>
```

### Restoring Velero Backups
To restore an entire namespace to a new cluster:
```bash
velero restore create --from-backup <BACKUP_NAME>
```

---

> [!TIP]
> **Pro Tip**: Always verify your backups. A backup that hasn't been tested for recovery is not a backup!

**Full Reference:** [K3s.guide Backup Strategy](https://k3s.guide/docs/kubernetes/k3s-backup)
