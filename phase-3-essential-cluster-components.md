# Phase 3: Essential Cluster Components

## Learning Objectives
- Install and configure cluster monitoring (metrics-server)
- Set up persistent storage solutions
- Deploy a database with persistent storage
- Understand StatefulSets vs Deployments
- Configure cluster DNS and service discovery
- Set up basic logging and monitoring

## Prerequisites
- Completed Phase 2: Kubernetes Cluster Setup
- Functional 3-node cluster with CNI networking
- kubectl configured and working
- All nodes in "Ready" state

## Understanding Kubernetes Storage and Persistence

Before diving into components, let's understand Kubernetes storage concepts:

```
┌─────────────────────────────────────────────────────────────┐
│                    Storage Architecture                     │
│                                                             │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐ │
│  │   Pod       │    │     PVC      │    │ StorageClass    │ │
│  │ Container   │───▶│(PersistentVol│───▶│   (Driver)      │ │
│  │             │    │ umeClaim)    │    │                 │ │
│  └─────────────┘    └──────────────┘    └─────────────────┘ │
│                              │                             │
│                              ▼                             │
│                      ┌──────────────┐                      │
│                      │     PV       │                      │
│                      │(PersistentVol│                      │
│                      │   ume)       │                      │
│                      └──────────────┘                      │
│                              │                             │
│                              ▼                             │
│                   ┌─────────────────────┐                  │
│                   │  Physical Storage   │                  │
│                   │(Local, NFS, etc.)   │                  │
│                   └─────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Install Metrics Server

**Why metrics-server?**
- Enables `kubectl top` commands
- Required for Horizontal Pod Autoscaling (HPA)
- Provides basic resource monitoring
- Foundation for more advanced monitoring

### Install metrics-server

**Run from Control Plane Node (k8s-master):**

```bash
# Download and install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics-server to be ready
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=300s

# Verify installation
kubectl get pods -n kube-system | grep metrics-server
```

### Configure metrics-server for Local Cluster

Since we're using self-signed certificates, we need to configure metrics-server to work with our setup:

**Run from Control Plane Node (k8s-master):**

```bash
# Edit metrics-server deployment to add insecure flags
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Wait for rollout to complete
kubectl rollout status deployment/metrics-server -n kube-system
```

### Test Metrics Server

**Run from Control Plane Node (k8s-master):**

```bash
# Wait a moment for metrics to be collected
sleep 30

# Check node metrics
kubectl top nodes

# Check system pod metrics
kubectl top pods -n kube-system
```

## Step 2: Set Up Local Persistent Storage

For learning purposes, we'll use local storage on each node. In production, you'd typically use network storage.

### Create StorageClass for Local Storage

**Run from Control Plane Node (k8s-master):**

```bash
# Create local storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
EOF
```

### Create Local Persistent Volumes

**Run from Control Plane Node (k8s-master):**

```bash
# Create directories on each worker node for local storage
ssh k8sadmin@k8s-worker-1 'sudo mkdir -p /mnt/local-storage/pv1 /mnt/local-storage/pv2'
ssh k8sadmin@k8s-worker-2 'sudo mkdir -p /mnt/local-storage/pv1 /mnt/local-storage/pv2'

# Set appropriate permissions
ssh k8sadmin@k8s-worker-1 'sudo chmod 755 /mnt/local-storage/pv*'
ssh k8sadmin@k8s-worker-2 'sudo chmod 755 /mnt/local-storage/pv*'
```

### Create Persistent Volumes

**Run from Control Plane Node (k8s-master):**

```bash
# Create PV for worker-1
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-1
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/local-storage/pv1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-worker-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-2
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/local-storage/pv2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-worker-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-3
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/local-storage/pv1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-worker-2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-4
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/local-storage/pv2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-worker-2
EOF

# Verify PVs are created
kubectl get pv
```

## Step 3: Deploy PostgreSQL Database

We'll deploy PostgreSQL as a StatefulSet to demonstrate persistent storage and database management in Kubernetes.

### Create Namespace for Database

**Run from Control Plane Node (k8s-master):**

```bash
# Create dedicated namespace
kubectl create namespace database

# Set default namespace for convenience
kubectl config set-context --current --namespace=database
```

### Create ConfigMap for PostgreSQL Configuration

**Run from Control Plane Node (k8s-master):**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: database
data:
  POSTGRES_DB: "appdb"
  POSTGRES_USER: "appuser"
  POSTGRES_PASSWORD: "secure-password-123"
  PGDATA: "/var/lib/postgresql/data/pgdata"
EOF
```

### Create PostgreSQL StatefulSet

**Run from Control Plane Node (k8s-master):**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - configMapRef:
            name: postgres-config
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - appuser
            - -d
            - appdb
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - appuser
            - -d
            - appdb
          initialDelaySeconds: 5
          timeoutSeconds: 1
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: local-storage
      resources:
        requests:
          storage: 5Gi
EOF
```

### Create PostgreSQL Service

**Run from Control Plane Node (k8s-master):**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: database
spec:
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
  type: ClusterIP
EOF
```

### Verify PostgreSQL Deployment

**Run from Control Plane Node (k8s-master):**

```bash
# Check StatefulSet status
kubectl get statefulsets -n database

# Check pods
kubectl get pods -n database

# Check PVC creation
kubectl get pvc -n database

# Check which node the pod is running on
kubectl get pods -o wide -n database

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod/postgres-0 -n database --timeout=300s
```

### Test Database Connection

**Run from Control Plane Node (k8s-master):**

```bash
# Connect to PostgreSQL pod
kubectl exec -it postgres-0 -n database -- psql -U appuser -d appdb

# Inside PostgreSQL shell, create a test table:
```
CREATE TABLE test_table (id SERIAL PRIMARY KEY, name VARCHAR(50));

INSERT INTO test_table (name) VALUES ('test-data');

SELECT * FROM test_table;

\q
```

# Or test connection from outside the pod
kubectl run postgres-client --rm -i --tty --image postgres:15 -- bash
# Inside the client pod:
# psql -h postgres-service.database.svc.cluster.local -U appuser -d appdb
```

## Step 4: Set Up Cluster DNS Testing

Kubernetes includes CoreDNS by default. Let's verify and test DNS resolution.

### Check CoreDNS Status

**Run from Control Plane Node (k8s-master):**

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system | grep coredns

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml
```

### Test DNS Resolution

**Run from Control Plane Node (k8s-master):**

```bash
# Create a test pod for DNS testing (this will automatically connect you to the pod's shell)
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- sh

# You are now inside the pod's shell. Test DNS resolution:
nslookup kubernetes.default.svc.cluster.local
nslookup postgres-service.database.svc.cluster.local

# Exit the pod (this will also delete it due to --rm flag)
exit
```

### Create DNS Test Pod (Persistent)

**Run from Control Plane Node (k8s-master):**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dns-utils
  namespace: default
spec:
  containers:
  - name: dns-utils
    image: tutum/dnsutils
    command:
      - sleep
      - "infinity"
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

# Test DNS from this pod
kubectl exec dns-utils -- nslookup postgres-service.database.svc.cluster.local
kubectl exec dns-utils -- dig +short postgres-service.database.svc.cluster.local
```

## Step 5: Install and Configure Basic Logging

### Deploy Fluent Bit for Log Collection

**Run from Control Plane Node (k8s-master):**

```bash
# Create logging namespace
kubectl create namespace logging

# Create Fluent Bit configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [OUTPUT]
        Name  stdout
        Match *

  parsers.conf: |
    [PARSER]
        Name   docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    metadata:
      labels:
        name: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  - pods/logs
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: logging
EOF
```

### Verify Logging Setup

**Run from Control Plane Node (k8s-master):**

```bash
# Check Fluent Bit pods
kubectl get pods -n logging

# View Fluent Bit logs
kubectl logs -l name=fluent-bit -n logging --tail=50
```

## Step 6: Set Up Resource Quotas and Limits

### Create Resource Quota for Database Namespace

**Run from Control Plane Node (k8s-master):**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: database-quota
  namespace: database
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    persistentvolumeclaims: "4"
    services: "5"
    pods: "10"
EOF

# Check quota usage
kubectl describe quota database-quota -n database
```

### Create LimitRange for Default Resource Limits

**Run from Control Plane Node (k8s-master):**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: database
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF
```

## Step 7: Verification and Health Checks

### Create Comprehensive Health Check Script

**Run from Control Plane Node (k8s-master):**

```bash
# Create a health check script
cat <<'EOF' > k8s-health-check.sh
#!/bin/bash

echo "=== Kubernetes Cluster Health Check ==="
echo

echo "1. Node Status:"
kubectl get nodes -o wide
echo

echo "2. System Pods Status:"
kubectl get pods -n kube-system
echo

echo "3. Metrics Server Status:"
kubectl top nodes 2>/dev/null || echo "Metrics server not ready yet"
echo

echo "4. Storage Status:"
kubectl get pv
echo
kubectl get pvc -A
echo

echo "5. Database Status:"
kubectl get all -n database
echo

echo "6. DNS Test:"
kubectl exec dns-utils -- nslookup kubernetes.default.svc.cluster.local 2>/dev/null || echo "DNS test pod not ready"
echo

echo "7. Resource Usage:"
kubectl describe quota -A
echo

echo "=== Health Check Complete ==="
EOF

chmod +x k8s-health-check.sh

# Run the health check
./k8s-health-check.sh
```

### Test Database Persistence

**Run from Control Plane Node (k8s-master):**

```bash
# Create test data in PostgreSQL
kubectl exec postgres-0 -n database -- psql -U appuser -d appdb -c "
CREATE TABLE IF NOT EXISTS test_persistence (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    message TEXT
);
INSERT INTO test_persistence (message) VALUES ('Persistence test data');
"

# Verify data exists
kubectl exec postgres-0 -n database -- psql -U appuser -d appdb -c "SELECT * FROM test_persistence;"

# Simulate pod restart
kubectl delete pod postgres-0 -n database

# Wait for pod to restart
kubectl wait --for=condition=ready pod/postgres-0 -n database --timeout=300s

# Verify data persisted
kubectl exec postgres-0 -n database -- psql -U appuser -d appdb -c "SELECT * FROM test_persistence;"
```

## Troubleshooting

### Storage Issues

**Run from Control Plane Node (k8s-master):**

```bash
# Check PV and PVC status
kubectl get pv,pvc -A

# Check node storage
ssh k8sadmin@k8s-worker-1 'df -h /mnt/local-storage'

# Check PVC events
kubectl describe pvc -n database
```

### Database Issues

**Run from Control Plane Node (k8s-master):**

```bash
# Check PostgreSQL logs
kubectl logs postgres-0 -n database

# Check StatefulSet events
kubectl describe statefulset postgres -n database

# Check service endpoints
kubectl get endpoints -n database
```

### Metrics Server Issues

**Run from Control Plane Node (k8s-master):**

```bash
# Check metrics-server logs
kubectl logs -l k8s-app=metrics-server -n kube-system

# Verify node kubelet certificates
kubectl describe node k8s-worker-1 | grep -A 20 "System Info"
```

## Next Steps

With essential cluster components configured, you're ready for **Phase 4: Application Development & Deployment**. You should now have:

- ✅ Metrics server for resource monitoring
- ✅ Local persistent storage configured
- ✅ PostgreSQL database with persistent storage
- ✅ DNS resolution working
- ✅ Basic logging collection
- ✅ Resource quotas and limits configured
- ✅ Health monitoring tools in place

**Key Commands for Cluster Management:**
```bash
# Check overall cluster health
kubectl get nodes
kubectl get pods -A

# Monitor resources
kubectl top nodes
kubectl top pods -A

# Check storage
kubectl get pv,pvc -A

# Database access
kubectl exec -it postgres-0 -n database -- psql -U appuser -d appdb

# Run health check
./k8s-health-check.sh
```

**Important Files Created:**
- `k8s-health-check.sh` - Cluster health verification script
- Various YAML manifests for storage, database, and monitoring components
