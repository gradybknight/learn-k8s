# Phase 7: Advanced Topics

## Learning Objectives
- Master advanced kubectl commands and cluster troubleshooting
- Implement comprehensive monitoring with Prometheus and Grafana
- Set up centralized logging with ELK/EFK stack
- Configure cluster backup and disaster recovery
- Implement horizontal pod autoscaling (HPA) and vertical pod autoscaling (VPA)
- Practice cluster upgrades and maintenance procedures
- Learn advanced security practices and policies
- Explore service mesh concepts with Istio (optional)

## Prerequisites
- Completed Phase 6: GitOps & CI/CD
- Functional GitOps workflow with ArgoCD
- Applications deployed and accessible externally
- Basic understanding of monitoring and logging concepts

## Advanced Kubernetes Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                Advanced Cluster Architecture                │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Prometheus │  │   Grafana   │  │      ArgoCD        │  │
│  │ (Metrics)   │  │(Dashboard)  │  │   (GitOps)         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │                 │                   │            │
│         └─────────────────┼───────────────────┘            │
│                           │                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │Elasticsearch│  │   Kibana    │  │     Fluent Bit     │  │
│  │   (Logs)    │  │ (Log Viz)   │  │   (Log Collector)  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │                 │                   │            │
│         └─────────────────┼───────────────────┘            │
│                           │                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │    Velero   │  │     HPA     │  │        VPA         │  │
│  │  (Backup)   │  │(Horizontal) │  │   (Vertical)       │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                             │
│              ┌─────────────────────────────────┐            │
│              │         Applications           │            │
│              │   (Auto-scaling & Monitored)   │            │
│              └─────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Advanced kubectl Mastery

### Create kubectl Aliases and Functions

```bash
# Create kubectl productivity aliases
cat >> ~/.bashrc << 'EOF'
# Kubernetes aliases
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kga='kubectl get all'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias klog='kubectl logs'
alias kexec='kubectl exec -it'
alias kdesc='kubectl describe'

# Advanced kubectl functions
kns() {
    kubectl config set-context --current --namespace=$1
}

kpods() {
    kubectl get pods -o wide --sort-by='.status.containerStatuses[0].restartCount'
}

ktop() {
    kubectl top pods --sort-by=cpu
}

kwatch() {
    watch kubectl get pods -o wide
}

# Get pod by partial name
kgetpod() {
    kubectl get pods | grep $1 | head -1 | awk '{print $1}'
}

# Quick pod shell access
ksh() {
    local pod=$(kgetpod $1)
    kubectl exec -it $pod -- /bin/sh
}
EOF

source ~/.bashrc
```

### Advanced Troubleshooting Commands

```bash
# Create comprehensive cluster troubleshooting script
cat > advanced-troubleshooting.sh << 'EOF'
#!/bin/bash

echo "=== Advanced Kubernetes Troubleshooting ==="
echo

echo "1. Cluster Resource Usage:"
kubectl top nodes
echo
kubectl top pods -A --sort-by=cpu | head -10
echo

echo "2. Node Conditions and Taints:"
kubectl get nodes -o custom-columns="NAME:.metadata.name,STATUS:.status.conditions[-1].type,TAINTS:.spec.taints[*].key"
echo

echo "3. Failed/Pending Pods:"
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
echo

echo "4. Recent Events (Last 1 hour):"
kubectl get events -A --sort-by='.lastTimestamp' | awk 'NR==1 || $5 ~ /[0-9]+m/ || $5 ~ /[0-5][0-9]s/'
echo

echo "5. Resource Quotas and Limits:"
kubectl get resourcequotas -A
echo

echo "6. Persistent Volume Status:"
kubectl get pv,pvc -A
echo

echo "7. Service Endpoints:"
kubectl get endpoints -A | grep -v '<none>'
echo

echo "8. Network Policies:"
kubectl get networkpolicies -A
echo

echo "9. Certificate Status:"
kubectl get certificates -A
echo

echo "10. ArgoCD Application Health:"
kubectl get applications -n argocd -o custom-columns="NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status"
echo

echo "=== Troubleshooting Complete ==="
EOF

chmod +x advanced-troubleshooting.sh
```

### Performance Analysis Tools

```bash
# Create performance analysis script
cat > performance-analysis.sh << 'EOF'
#!/bin/bash

NAMESPACE=${1:-"k8s-learning-app"}

echo "=== Performance Analysis for Namespace: $NAMESPACE ==="
echo

echo "1. Resource Consumption:"
kubectl top pods -n $NAMESPACE --sort-by=cpu
echo

echo "2. Pod Resource Requests vs Limits:"
kubectl get pods -n $NAMESPACE -o custom-columns="POD:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,CPU_LIM:.spec.containers[*].resources.limits.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory,MEM_LIM:.spec.containers[*].resources.limits.memory"
echo

echo "3. Pod Restart Count:"
kubectl get pods -n $NAMESPACE -o custom-columns="POD:.metadata.name,RESTARTS:.status.containerStatuses[*].restartCount,READY:.status.containerStatuses[*].ready"
echo

echo "4. HPA Status (if exists):"
kubectl get hpa -n $NAMESPACE
echo

echo "5. Service Response Times (sample):"
for service in $(kubectl get services -n $NAMESPACE -o name); do
    service_name=$(echo $service | cut -d'/' -f2)
    echo "Testing $service_name..."
    kubectl run curl-test --rm -i --tty --image=curlimages/curl --restart=Never -- \
        time curl -s http://$service_name.$NAMESPACE.svc.cluster.local/health > /dev/null 2>&1 || echo "Service not reachable"
done
echo

echo "=== Performance Analysis Complete ==="
EOF

chmod +x performance-analysis.sh
```

## Step 2: Comprehensive Monitoring with Prometheus and Grafana

### Install Prometheus Operator

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus Operator using manifests
curl -sL https://github.com/prometheus-operator/prometheus-operator/releases/download/v0.68.0/bundle.yaml | kubectl apply -f -

# Wait for operator to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=prometheus-operator -n default --timeout=300s

# Create Prometheus instance
cat > prometheus.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      app: prometheus
  ruleSelector:
    matchLabels:
      app: prometheus
  resources:
    requests:
      memory: 400Mi
      cpu: 100m
    limits:
      memory: 800Mi
      cpu: 200m
  retention: 24h
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: local-storage
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: ClusterIP
  ports:
  - name: web
    port: 9090
    targetPort: web
  selector:
    prometheus: prometheus
EOF

kubectl apply -f prometheus.yaml
```

### Configure Service Monitors

```bash
# Create service monitor for your applications
cat > app-service-monitor.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: k8s-learning-app-monitor
  namespace: monitoring
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      app: backend
  namespaceSelector:
    matchNames:
    - k8s-learning-app
    - k8s-learning-app-staging
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubernetes-nodes
  namespace: monitoring
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      kubernetes.io/os: linux
  endpoints:
  - port: https-metrics
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecureSkipVerify: true
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 30s
    relabelings:
    - sourceLabels: [__meta_kubernetes_node_name]
      regex: (.+)
      targetLabel: __address__
      replacement: ${1}:10250
EOF

kubectl apply -f app-service-monitor.yaml
```

### Install Grafana

```bash
cat > grafana.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
      - name: grafana
        image: grafana/grafana:10.1.0
        ports:
        - containerPort: 3000
          name: http-grafana
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /robots.txt
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 2
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 3000
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 250m
            memory: 750Mi
          limits:
            cpu: 500m
            memory: 1Gi
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana-pv
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin123"  # Change this password
        - name: GF_INSTALL_PLUGINS
          value: "grafana-kubernetes-app"
      volumes:
      - name: grafana-pv
        persistentVolumeClaim:
          claimName: grafana-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: http-grafana
  selector:
    app: grafana
  type: ClusterIP
EOF

kubectl apply -f grafana.yaml
```

### Create Grafana Ingress

```bash
cat > grafana-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - grafana.your-cluster.duckdns.org
    secretName: grafana-tls
  rules:
  - host: grafana.your-cluster.duckdns.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
EOF

kubectl apply -f grafana-ingress.yaml
```

### Configure Grafana Data Sources and Dashboards

```bash
# Wait for Grafana to be ready
kubectl wait --for=condition=ready pod -l app=grafana -n monitoring --timeout=300s

# Configure Prometheus data source
cat > grafana-datasource-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
      access: proxy
      isDefault: true
EOF

kubectl apply -f grafana-datasource-config.yaml

# Create custom dashboard for your applications
cat > app-dashboard.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "K8s Learning Applications",
    "tags": ["kubernetes", "applications"],
    "style": "dark",
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Pod CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"k8s-learning-app\"}[5m])) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Pod Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(container_memory_usage_bytes{namespace=\"k8s-learning-app\"}) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 12, "y": 0}
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
EOF
```

## Step 3: Centralized Logging with ELK Stack

### Install Elasticsearch

```bash
# Create logging namespace
kubectl create namespace logging

# Install Elasticsearch
cat > elasticsearch.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.9.0
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        - name: xpack.security.enabled
          value: "false"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: local-storage
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
  - port: 9200
    name: rest
  - port: 9300
    name: inter-node
EOF

kubectl apply -f elasticsearch.yaml
```

### Install Kibana

```bash
cat > kibana.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.9.0
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch:9200"
        - name: ELASTICSEARCH_URL
          value: "http://elasticsearch:9200"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
  type: ClusterIP
EOF

kubectl apply -f kibana.yaml
```

### Update Fluent Bit Configuration for Elasticsearch

```bash
# Update existing Fluent Bit configuration to send logs to Elasticsearch
kubectl patch configmap fluent-bit-config -n logging --patch='
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

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch.logging.svc.cluster.local
        Port            9200
        Index           kubernetes-logs
        Type            _doc
        Logstash_Format On
        Replace_Dots    On
        Retry_Limit     False
'

# Restart Fluent Bit to apply new configuration
kubectl rollout restart daemonset/fluent-bit -n logging
```

### Create Kibana Ingress

```bash
cat > kibana-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: logging
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: kibana-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - Kibana'
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - kibana.your-cluster.duckdns.org
    secretName: kibana-tls
  rules:
  - host: kibana.your-cluster.duckdns.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
EOF

# Create basic auth for Kibana
htpasswd -c kibana-auth admin
kubectl create secret generic kibana-basic-auth --from-file=auth=kibana-auth -n logging
rm kibana-auth

kubectl apply -f kibana-ingress.yaml
```

## Step 4: Implement Horizontal Pod Autoscaling (HPA)

### Configure HPA for Backend Application

```bash
# Create HPA for backend
cat > backend-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: k8s-learning-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max
EOF

kubectl apply -f backend-hpa.yaml
```

### Create Load Testing Tool

```bash
cat > load-test.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-test
  namespace: k8s-learning-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-test
  template:
    metadata:
      labels:
        app: load-test
    spec:
      containers:
      - name: load-test
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            wget -q -O- http://backend-service:3000/hello
            sleep 0.1
          done
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

# Deploy load test (optional)
# kubectl apply -f load-test.yaml

# Monitor HPA scaling
# watch kubectl get hpa -n k8s-learning-app
```

### Install Vertical Pod Autoscaler (VPA)

```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA
./hack/vpa-up.sh

# Create VPA for frontend
cat > frontend-vpa.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: frontend-vpa
  namespace: k8s-learning-app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: frontend
      maxAllowed:
        cpu: 500m
        memory: 512Mi
      minAllowed:
        cpu: 50m
        memory: 64Mi
      controlledResources: ["cpu", "memory"]
EOF

kubectl apply -f frontend-vpa.yaml
```

## Step 5: Cluster Backup and Disaster Recovery

### Install Velero

```bash
# Download Velero
curl -L https://github.com/vmware-tanzu/velero/releases/download/v1.11.1/velero-v1.11.1-linux-amd64.tar.gz | tar -xz
sudo mv velero-v1.11.1-linux-amd64/velero /usr/local/bin/

# For this learning environment, we'll use local filesystem backup
# In production, you'd use cloud storage (AWS S3, GCP, Azure)

# Create backup storage location on control plane
ssh k8sadmin@k8s-control 'sudo mkdir -p /mnt/velero-backups && sudo chmod 777 /mnt/velero-backups'

# Install Velero in cluster
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.7.1 \
    --bucket velero-backups \
    --secret-file /dev/null \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio:9000

# Create a simple MinIO instance for local object storage
cat > minio.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        hostPath:
          path: /mnt/velero-backups
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    app: minio
  ports:
  - port: 9000
    targetPort: 9000
EOF

kubectl apply -f minio.yaml
```

### Create Backup Schedules

```bash
# Create daily backup schedule
velero schedule create daily-backup \
    --schedule="0 2 * * *" \
    --include-namespaces k8s-learning-app,k8s-learning-app-staging,database \
    --ttl 720h

# Create application-specific backup
velero backup create app-backup-$(date +%Y%m%d) \
    --include-namespaces k8s-learning-app,database \
    --wait

# List backups
velero backup get

# Create restore test procedure
cat > restore-test.sh << 'EOF'
#!/bin/bash

BACKUP_NAME=$1

if [ -z "$BACKUP_NAME" ]; then
    echo "Usage: $0 <backup-name>"
    echo "Available backups:"
    velero backup get
    exit 1
fi

echo "Testing restore from backup: $BACKUP_NAME"

# Create test namespace
kubectl create namespace restore-test

# Restore to test namespace
velero restore create restore-test-$(date +%Y%m%d-%H%M) \
    --from-backup $BACKUP_NAME \
    --namespace-mappings k8s-learning-app:restore-test

echo "Restore initiated. Check status with:"
echo "velero restore get"
echo "kubectl get pods -n restore-test"
EOF

chmod +x restore-test.sh
```

## Step 6: Cluster Upgrades and Maintenance

### Create Cluster Upgrade Procedure

```bash
cat > cluster-upgrade.sh << 'EOF'
#!/bin/bash

CURRENT_VERSION=$(kubectl version --short | grep Server | awk '{print $3}')
echo "Current Kubernetes version: $CURRENT_VERSION"

echo "=== Pre-Upgrade Checklist ==="
echo "1. Create full cluster backup"
echo "2. Check node health"
echo "3. Verify all applications are healthy"
echo "4. Review release notes for breaking changes"
echo

echo "Creating pre-upgrade backup..."
velero backup create pre-upgrade-backup-$(date +%Y%m%d) \
    --include-namespaces k8s-learning-app,k8s-learning-app-staging,database,monitoring,logging,argocd \
    --wait

echo "Node health check:"
kubectl get nodes -o wide

echo "Application health check:"
kubectl get pods -A | grep -v Running

echo "=== Manual Upgrade Steps ==="
echo "1. Upgrade control plane:"
echo "   ssh k8s-control"
echo "   sudo apt update"
echo "   sudo apt-cache madison kubeadm"
echo "   sudo apt-mark unhold kubeadm"
echo "   sudo apt-get update && sudo apt-get install -y kubeadm=1.28.3-00"
echo "   sudo apt-mark hold kubeadm"
echo "   sudo kubeadm upgrade plan"
echo "   sudo kubeadm upgrade apply v1.28.3"
echo
echo "2. Upgrade kubelet and kubectl on control plane:"
echo "   sudo apt-mark unhold kubelet kubectl"
echo "   sudo apt-get update && sudo apt-get install -y kubelet=1.28.3-00 kubectl=1.28.3-00"
echo "   sudo apt-mark hold kubelet kubectl"
echo "   sudo systemctl daemon-reload"
echo "   sudo systemctl restart kubelet"
echo
echo "3. Upgrade worker nodes (one at a time):"
echo "   kubectl drain k8s-worker-1 --ignore-daemonsets"
echo "   ssh k8s-worker-1"
echo "   sudo apt-mark unhold kubeadm kubelet kubectl"
echo "   sudo apt-get update && sudo apt-get install -y kubeadm=1.28.3-00 kubelet=1.28.3-00 kubectl=1.28.3-00"
echo "   sudo apt-mark hold kubeadm kubelet kubectl"
echo "   sudo kubeadm upgrade node"
echo "   sudo systemctl daemon-reload"
echo "   sudo systemctl restart kubelet"
echo "   kubectl uncordon k8s-worker-1"
echo
echo "4. Repeat step 3 for k8s-worker-2"
echo
echo "5. Verify upgrade:"
echo "   kubectl get nodes"
echo "   kubectl version"
echo "   ./advanced-troubleshooting.sh"
EOF

chmod +x cluster-upgrade.sh
```

### Node Maintenance Procedures

```bash
cat > node-maintenance.sh << 'EOF'
#!/bin/bash

NODE_NAME=$1
OPERATION=$2

if [ -z "$NODE_NAME" ] || [ -z "$OPERATION" ]; then
    echo "Usage: $0 <node-name> <drain|uncordon|reboot>"
    echo "Available nodes:"
    kubectl get nodes -o name | cut -d'/' -f2
    exit 1
fi

case $OPERATION in
    drain)
        echo "Draining node $NODE_NAME..."
        kubectl drain $NODE_NAME --ignore-daemonsets --force --delete-emptydir-data
        echo "Node $NODE_NAME drained. Pods have been moved to other nodes."
        ;;
    uncordon)
        echo "Uncordoning node $NODE_NAME..."
        kubectl uncordon $NODE_NAME
        echo "Node $NODE_NAME is now schedulable again."
        ;;
    reboot)
        echo "Draining and rebooting node $NODE_NAME..."
        kubectl drain $NODE_NAME --ignore-daemonsets --force --delete-emptydir-data
        echo "Node drained. Rebooting..."
        ssh k8sadmin@$NODE_NAME 'sudo reboot'
        echo "Waiting for node to come back online..."
        sleep 60
        while ! kubectl get node $NODE_NAME | grep -q Ready; do
            echo "Waiting for node $NODE_NAME to be ready..."
            sleep 10
        done
        kubectl uncordon $NODE_NAME
        echo "Node $NODE_NAME rebooted and uncordoned."
        ;;
    *)
        echo "Invalid operation: $OPERATION"
        echo "Valid operations: drain, uncordon, reboot"
        exit 1
        ;;
esac
EOF

chmod +x node-maintenance.sh
```

## Step 7: Advanced Security Practices

### Implement Pod Security Standards

```bash
# Create security policy enforcement
cat > pod-security-policy.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-apps
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: secure-apps
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF

kubectl apply -f pod-security-policy.yaml
```

### Security Scanning with Trivy

```bash
# Install Trivy operator for vulnerability scanning
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/v0.15.1/deploy/static/trivy-operator.yaml

# Create vulnerability scan policy
cat > vulnerability-scan.yaml << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vulnerability-scan
  namespace: monitoring
spec:
  schedule: "0 2 * * 0"  # Weekly on Sunday at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: trivy
            image: aquasecurity/trivy:latest
            command:
            - sh
            - -c
            - |
              trivy image --severity HIGH,CRITICAL \
                ghcr.io/your-username/k8s-learning-apps/backend:latest \
                ghcr.io/your-username/k8s-learning-apps/frontend:latest
          restartPolicy: OnFailure
EOF

kubectl apply -f vulnerability-scan.yaml
```

### RBAC Best Practices

```bash
# Create least-privilege service accounts
cat > rbac-policies.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: k8s-learning-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: k8s-learning-app
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: k8s-learning-app
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: k8s-learning-app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f rbac-policies.yaml
```

## Step 8: Create Comprehensive Monitoring Dashboard

### Final Monitoring Script

```bash
cat > comprehensive-monitoring.sh << 'EOF'
#!/bin/bash

echo "=== Comprehensive Cluster Monitoring ==="
echo

echo "1. Cluster Overview:"
echo "Nodes: $(kubectl get nodes --no-headers | wc -l)"
echo "Namespaces: $(kubectl get namespaces --no-headers | wc -l)"
echo "Pods: $(kubectl get pods -A --no-headers | wc -l)"
echo "Services: $(kubectl get services -A --no-headers | wc -l)"
echo

echo "2. Resource Utilization:"
kubectl top nodes
echo

echo "3. Application Health:"
kubectl get deployments -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.readyReplicas,UP-TO-DATE:.status.updatedReplicas,AVAILABLE:.status.availableReplicas"
echo

echo "4. Storage:"
kubectl get pv -o custom-columns="NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase,CLAIM:.spec.claimRef.name"
echo

echo "5. Ingress Status:"
kubectl get ingress -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTS:.spec.rules[*].host,ADDRESS:.status.loadBalancer.ingress[*].ip"
echo

echo "6. Certificate Status:"
kubectl get certificates -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[?(@.type=='Ready')].status,AGE:.metadata.creationTimestamp"
echo

echo "7. Recent Events (Last 10):"
kubectl get events -A --sort-by='.lastTimestamp' | tail -10
echo

echo "8. Backup Status:"
velero backup get --output=table 2>/dev/null || echo "Velero not configured"
echo

echo "9. ArgoCD Applications:"
kubectl get applications -n argocd -o custom-columns="NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status" 2>/dev/null || echo "ArgoCD not configured"
echo

echo "10. External Access URLs:"
echo "Frontend: https://your-cluster.duckdns.org"
echo "API: https://api.your-cluster.duckdns.org"
echo "ArgoCD: https://argocd.your-cluster.duckdns.org"
echo "Grafana: https://grafana.your-cluster.duckdns.org"
echo "Kibana: https://kibana.your-cluster.duckdns.org"
echo

echo "=== Monitoring Complete ==="
EOF

chmod +x comprehensive-monitoring.sh
```

## Congratulations!

You have successfully completed all 7 phases of the Kubernetes learning journey! You now have:

- ✅ **Production-ready 3-node Kubernetes cluster**
- ✅ **Full application stack with database, backend, and frontend**
- ✅ **External HTTPS access with SSL certificates**
- ✅ **GitOps workflow with automated CI/CD**
- ✅ **Comprehensive monitoring with Prometheus and Grafana**
- ✅ **Centralized logging with ELK stack**
- ✅ **Auto-scaling capabilities (HPA and VPA)**
- ✅ **Backup and disaster recovery procedures**
- ✅ **Security policies and best practices**
- ✅ **Maintenance and upgrade procedures**

## Key Skills Mastered

### Infrastructure & Operations
- Linux server administration
- Container runtime configuration
- Network security and firewall management
- SSL/TLS certificate management
- Dynamic DNS configuration

### Kubernetes Administration
- Cluster setup and configuration
- Pod, Service, and Ingress management
- Storage and persistent volumes
- Resource quotas and limits
- Network policies
- RBAC and security policies

### DevOps & GitOps
- Container image building and management
- CI/CD pipeline design and implementation
- GitOps workflow with ArgoCD
- Multi-environment deployment strategies
- Infrastructure as Code practices

### Monitoring & Observability
- Metrics collection with Prometheus
- Dashboard creation with Grafana
- Centralized logging with ELK stack
- Performance monitoring and alerting
- Application and infrastructure monitoring

### Operational Excellence
- Backup and disaster recovery
- Cluster upgrades and maintenance
- Auto-scaling configuration
- Security scanning and compliance
- Troubleshooting and performance analysis

## Next Learning Opportunities

Consider exploring these advanced topics:
- **Service Mesh** (Istio, Linkerd)
- **Multi-cluster management** (Cluster API, Rancher)
- **Advanced networking** (CNI development, BGP)
- **Custom operators** (Operator SDK, Kubebuilder)
- **Cloud-native security** (Falco, OPA Gatekeeper)
- **Kubernetes certification** (CKA, CKAD, CKS)

## Essential Commands Reference

```bash
# Daily operations
./comprehensive-monitoring.sh
./advanced-troubleshooting.sh
./performance-analysis.sh k8s-learning-app

# Cluster maintenance
./cluster-upgrade.sh
./node-maintenance.sh k8s-worker-1 drain

# GitOps operations
argocd app list
argocd app sync k8s-learning-staging

# Backup operations
velero backup create manual-backup-$(date +%Y%m%d)
velero backup get
```

You've built a production-grade Kubernetes learning environment that demonstrates industry best practices. This foundation will serve you well as you continue your Kubernetes and cloud-native journey!