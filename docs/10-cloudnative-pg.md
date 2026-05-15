# Chapter 10: Database Layer (CloudNativePG)

In this chapter, we will deploy **CloudNativePG (CNPG)**, a cloud-native PostgreSQL operator. It simplifies the lifecycle management of HA PostgreSQL clusters, including bootstrapping, replication, and failover.

## 1. Install CloudNativePG Operator

We will use Helm to install the operator. By default, it manages its own internal TLS certificates for secure cluster communication using **Operator-Managed Mode**.

### Add Helm Repository
```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts --force-update

# Check for the latest versions
helm search repo cnpg/cloudnative-pg -l | head -n 5
```

### Install the Operator
We will install the operator in the `cnpg-system` namespace.

```bash
helm install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace \
  --version 0.28.2
```

**Verify the Operator Pods:**
```bash
kubectl rollout status deployment -n cnpg-system cnpg-cloudnative-pg
```

---

## 2. Storage Optimization (Longhorn)

Before deploying the cluster, we need to address **Write Amplification**. 

Since CloudNativePG performs its own synchronous/asynchronous replication at the PostgreSQL level, having Longhorn also replicate data 3 times at the block level can lead to unnecessary disk I/O and performance degradation.

We will create a specialized **StorageClass** for CNPG that ensures data resides on the same node as the Pod (**Strict Local**) to maximize performance.

```yaml
cat <<EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-cnpg
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "1" # Set to 1 because CNPG manages its own HA replication
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "xfs"
  dataLocality: "strict-local" # Critical: Ensures volume is on the same node as the Pod
EOF
```

## 3. Deploy a High-Availability Cluster

We will create a 3-instance PostgreSQL cluster in its own dedicated namespace.

### Create the Namespace
```bash
kubectl create namespace cnpg-database
```

### Deploy the Cluster
This configuration follows production best practices:
- **Separate WAL Storage**: Isolates log traffic for performance and reliability.
- **Pod Anti-Affinity**: Ensures instances are scheduled on different physical nodes.
- **Optimized Storage**: Uses our custom Longhorn class.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: cnpg-database
spec:
  instances: 3
  
  # Data Storage
  storage:
    size: 1Gi
    storageClass: longhorn-cnpg
    
  # Separate WAL Storage (Best Practice)
  walStorage:
    size: 1Gi
    storageClass: longhorn-cnpg

  bootstrap:
    initdb:
      database: main_db
      owner: cluster_admin
      
  # High Availability: Anti-Affinity
  affinity:
    enablePodAntiAffinity: true
    topologyKey: kubernetes.io/hostname

  # Resource Management
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1"
EOF
```

---

## 4. Management and Verification

The best way to interact with your Postgres clusters is by using the **CNPG kubectl plugin**.

### Install the CNPG Plugin
Run this on your **local machine**:
```bash
curl -sSfL https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-kubectl-cnpg.sh | sh -s -- -b /usr/local/bin
```

### Verify Cluster Status
The plugin provides the most detail, but you can also use standard `kubectl` to see the cluster health:

```bash
# Using the plugin (Detailed)
kubectl cnpg status postgres-cluster -n cnpg-database

# Using standard kubectl (Quick check)
kubectl get cluster -n cnpg-database
```

Expected `kubectl get cluster` output:
```text
NAME               AGE     INSTANCES   READY   STATUS                     PRIMARY
postgres-cluster   2m45s   3           3       Cluster in healthy state   postgres-cluster-1
```

---

## 5. Accessing the Database

CloudNativePG automatically creates three types of services in the `cnpg-database` namespace:
1. `postgres-cluster-rw`: Always points to the current Primary.
2. `postgres-cluster-ro`: Points to the Read-Only replicas.
3. `postgres-cluster-r`: Points to any instance (Read).

### Retrieve Application Credentials
The operator creates a secret with the `cluster_admin` credentials:

```bash
# Get the password
kubectl get secret postgres-cluster-app -n cnpg-database -o jsonpath="{.data.password}" | base64 -d
```

### Testing the Connection (Inside the cluster)
You can spawn a temporary pod to test the database connection:

```bash
kubectl run pg-client --rm -it --image postgres:18 -n cnpg-database -- \
  psql -h postgres-cluster-rw -U cluster_admin -d main_db
```

---

> [!TIP]
> In the next chapter, we will configure **automated backups** for this cluster using S3-compatible storage (like MinIO or AWS S3), leveraging CNPG's native WAL archiving capabilities.
