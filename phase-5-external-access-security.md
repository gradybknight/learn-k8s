# Phase 5: External Access & Security

## Learning Objectives
- Install and configure NGINX Ingress Controller
- Set up SSL/TLS encryption with Let's Encrypt and cert-manager
- Configure dynamic DNS for home network access
- Implement network security policies
- Set up secure external access to cluster services
- Understand ingress, load balancing, and routing concepts

## Prerequisites
- Completed Phase 4: Application Development & Deployment
- Frontend and backend applications running in cluster
- Router with port forwarding capabilities
- Domain name or dynamic DNS service access

## Understanding External Access Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 External Access Flow                        │
│                                                             │
│  Internet ──▶ Router ──▶ k8s-control ──▶ Ingress Controller │
│     │            │            │                │            │
│     │            │            │                ▼            │
│     │            │            │         ┌─────────────────┐ │
│     │            │            │         │   Services      │ │
│     │            │            │         │                 │ │
│     │            │            │         │ ┌─────────────┐ │ │
│  (HTTPS)      (Port         (HTTP)      │ │  Frontend   │ │ │
│  Port 443      Forward)     Port 80     │ │  Service    │ │ │
│               Ports 80,443             │ └─────────────┘ │ │
│                                         │                 │ │
│                                         │ ┌─────────────┐ │ │
│                                         │ │  Backend    │ │ │
│                                         │ │  Service    │ │ │
│                                         │ └─────────────┘ │ │
│                                         └─────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Install NGINX Ingress Controller

### Why NGINX Ingress?
- Production-ready and widely adopted
- Excellent performance and feature set
- Strong community support and documentation
- Works well with cert-manager for SSL/TLS

### Install NGINX Ingress Controller

```bash
# Add NGINX Ingress Helm repository (we'll install manually for learning)
# Instead, we'll use the official YAML manifests

# Download and apply NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml

# Wait for the ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

### Configure Ingress Controller for Home Network

Since we're running on bare metal, we need to modify the service to use specific node ports:

```bash
# Patch the ingress-nginx-controller service to use specific NodePorts
kubectl patch service ingress-nginx-controller -n ingress-nginx -p='
{
  "spec": {
    "type": "NodePort",
    "ports": [
      {
        "name": "http",
        "port": 80,
        "protocol": "TCP",
        "targetPort": "http",
        "nodePort": 30080
      },
      {
        "name": "https", 
        "port": 443,
        "protocol": "TCP",
        "targetPort": "https",
        "nodePort": 30443
      }
    ]
  }
}'

# Verify the service configuration
kubectl get service ingress-nginx-controller -n ingress-nginx -o wide
```

### Update Firewall Rules

```bash
# Add firewall rules for ingress traffic on all nodes
ssh k8sadmin@k8s-control 'sudo ufw allow 30080/tcp && sudo ufw allow 30443/tcp'
ssh k8sadmin@k8s-worker-1 'sudo ufw allow 30080/tcp && sudo ufw allow 30443/tcp'
ssh k8sadmin@k8s-worker-2 'sudo ufw allow 30080/tcp && sudo ufw allow 30443/tcp'

# Verify firewall rules
ssh k8sadmin@k8s-control 'sudo ufw status numbered'
```

## Step 2: Set Up DNS for Your Domain

You have two options for DNS setup. Choose the one that fits your situation:

### Option A: Use Your Own Domain with AWS Route53 (Recommended)

If you own a domain registered with AWS Route53, this is the preferred approach for a professional setup.

#### Configure Route53 for Dynamic IP Updates

```bash
# Install AWS CLI if not already installed
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS credentials (you'll need an IAM user with Route53 permissions)
aws configure
# Enter your Access Key ID, Secret Access Key, region (e.g., us-east-1)
```

#### Create IAM Policy for Route53 Updates

In AWS Console, create an IAM user with this policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:GetHostedZone",
                "route53:ListResourceRecordSets"
            ],
            "Resource": "arn:aws:route53:::hostedzone/YOUR_HOSTED_ZONE_ID"
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones"
            ],
            "Resource": "*"
        }
    ]
}
```

#### Create Dynamic DNS Update Script for Route53

```bash
# SSH to k8s-control node (better than dev machine since it's always on)
ssh k8sadmin@k8s-control

# Install AWS CLI on k8s-control node
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws/

# Configure AWS credentials
aws configure
# Enter your Access Key ID, Secret Access Key, region (e.g., us-east-1)

# Get your hosted zone ID
aws route53 list-hosted-zones --query "HostedZones[?Name=='yourdomain.com.'].Id" --output text

# Create Route53 update script
cat > update-route53.sh << 'EOF'
#!/bin/bash

# Configuration - Update these values
DOMAIN="yourdomain.com"
SUBDOMAIN="k8s"  # This will create k8s.yourdomain.com
HOSTED_ZONE_ID="YOUR_HOSTED_ZONE_ID"  # Get from AWS console or CLI
TTL=300

# Get current public IP
CURRENT_IP=$(curl -s https://checkip.amazonaws.com/)

if [ -z "$CURRENT_IP" ]; then
    echo "Failed to get current IP"
    exit 1
fi

# Get current IP from Route53
ROUTE53_IP=$(aws route53 list-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --query "ResourceRecordSets[?Name=='${SUBDOMAIN}.${DOMAIN}.'].ResourceRecords[0].Value" \
    --output text)

# Remove quotes if present
ROUTE53_IP=$(echo $ROUTE53_IP | tr -d '"')

if [ "$CURRENT_IP" != "$ROUTE53_IP" ]; then
    echo "IP changed from $ROUTE53_IP to $CURRENT_IP. Updating Route53..."
    
    # Create change batch JSON
    cat > /tmp/route53-change.json << JSON
{
    "Changes": [
        {
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "${SUBDOMAIN}.${DOMAIN}",
                "Type": "A",
                "TTL": ${TTL},
                "ResourceRecords": [
                    {
                        "Value": "${CURRENT_IP}"
                    }
                ]
            }
        }
    ]
}
JSON

    # Apply the change
    aws route53 change-resource-record-sets \
        --hosted-zone-id $HOSTED_ZONE_ID \
        --change-batch file:///tmp/route53-change.json

    echo "$(date): Route53 updated $SUBDOMAIN.$DOMAIN to $CURRENT_IP"
    
    # Clean up
    rm /tmp/route53-change.json
else
    echo "IP unchanged: $CURRENT_IP"
fi
EOF

chmod +x update-route53.sh

# Test the script
./update-route53.sh

# Set up cron job to run every 5 minutes on k8s-control node
echo "*/5 * * * * /home/k8sadmin/update-route53.sh >> /var/log/route53-update.log 2>&1" | crontab -
```

#### Create Wildcard DNS Record (Optional but Recommended)

```bash
# Create wildcard subdomain for services like api.k8s.yourdomain.com, grafana.k8s.yourdomain.com
cat > create-wildcard.sh << 'EOF'
#!/bin/bash

DOMAIN="yourdomain.com"
SUBDOMAIN="k8s"
HOSTED_ZONE_ID="YOUR_HOSTED_ZONE_ID"
TTL=300

# Get current IP
CURRENT_IP=$(curl -s https://checkip.amazonaws.com/)

# Create wildcard record
cat > /tmp/wildcard-change.json << JSON
{
    "Changes": [
        {
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "*.${SUBDOMAIN}.${DOMAIN}",
                "Type": "A",
                "TTL": ${TTL},
                "ResourceRecords": [
                    {
                        "Value": "${CURRENT_IP}"
                    }
                ]
            }
        }
    ]
}
JSON

aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch file:///tmp/wildcard-change.json

echo "Wildcard DNS created: *.${SUBDOMAIN}.${DOMAIN} -> ${CURRENT_IP}"
rm /tmp/wildcard-change.json
EOF

chmod +x create-wildcard.sh
./create-wildcard.sh
```

### Option B: Use DuckDNS (Free Alternative)

If you don't want to use your own domain, DuckDNS provides a free alternative:

```bash
# Run this on your development machine or home router with internet access
# Create a script to update DuckDNS (run this on your main computer or router)
cat > update-duckdns.sh << 'EOF'
#!/bin/bash

# Configuration
DUCKDNS_DOMAIN="your-cluster"  # Change this to your subdomain
DUCKDNS_TOKEN="your-token"     # Change this to your token from duckdns.org

# Update DuckDNS
curl -s "https://www.duckdns.org/update?domains=${DUCKDNS_DOMAIN}&token=${DUCKDNS_TOKEN}&ip="

# Log the update
echo "$(date): DuckDNS updated for ${DUCKDNS_DOMAIN}.duckdns.org" >> /var/log/dyndns.log
EOF

chmod +x update-duckdns.sh
./update-duckdns.sh

# Set up cron job to run every 5 minutes
echo "*/5 * * * * /path/to/update-duckdns.sh" | crontab -
```

### Configure Router Port Forwarding

Regardless of which DNS option you choose, configure your router:

- **External Port 80** → **Internal IP** (k8s-control: 192.168.1.10) **Port 30080**
- **External Port 443** → **Internal IP** (k8s-control: 192.168.1.10) **Port 30443**

**Note:** Configuration varies by router manufacturer. Look for "Port Forwarding," "Virtual Servers," or "NAT" settings.

## Step 3: Install cert-manager for SSL/TLS

### Install cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s

# Verify installation
kubectl get pods -n cert-manager
```

### Create Let's Encrypt ClusterIssuer

```bash
# Create Let's Encrypt staging issuer (for testing)
cat > letsencrypt-staging.yaml << 'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Change this to your email
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Create Let's Encrypt production issuer
cat > letsencrypt-production.yaml << 'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Change this to your email
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Apply both issuers
kubectl apply -f letsencrypt-staging.yaml
kubectl apply -f letsencrypt-production.yaml

# Verify issuers
kubectl get clusterissuers
```

## Step 4: Create Ingress Resources

### Create Ingress for Frontend Application

```bash
cd ~/k8s-learning-apps/k8s-manifests

# Create frontend ingress with SSL
# Choose ONE of the following based on your DNS setup:

# Option A: For your own domain (recommended)
cat > frontend-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: k8s-learning-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-staging"  # Use staging first
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - k8s.yourdomain.com  # Replace with your domain
    - www.k8s.yourdomain.com
    secretName: frontend-tls
  rules:
  - host: k8s.yourdomain.com  # Replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 8080
  - host: www.k8s.yourdomain.com  # Optional www subdomain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 8080
EOF

# Option B: For DuckDNS (alternative)
# cat > frontend-ingress.yaml << 'EOF'
# apiVersion: networking.k8s.io/v1
# kind: Ingress
# metadata:
#   name: frontend-ingress
#   namespace: k8s-learning-app
#   annotations:
#     nginx.ingress.kubernetes.io/rewrite-target: /
#     cert-manager.io/cluster-issuer: "letsencrypt-staging"
#     nginx.ingress.kubernetes.io/ssl-redirect: "true"
#     nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
# spec:
#   ingressClassName: nginx
#   tls:
#   - hosts:
#     - your-cluster.duckdns.org
#     secretName: frontend-tls
#   rules:
#   - host: your-cluster.duckdns.org
#     http:
#       paths:
#       - path: /
#         pathType: Prefix
#         backend:
#           service:
#             name: frontend-service
#             port:
#               number: 8080
# EOF

# Apply the ingress
kubectl apply -f frontend-ingress.yaml
```

### Create Ingress for Backend API

```bash
# Option A: For your own domain (recommended)
cat > backend-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: k8s-learning-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.k8s.yourdomain.com  # Replace with your domain
    secretName: backend-tls
  rules:
  - host: api.k8s.yourdomain.com  # Replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 3000
EOF

# Option B: For DuckDNS (alternative)
# cat > backend-ingress.yaml << 'EOF'
# apiVersion: networking.k8s.io/v1
# kind: Ingress
# metadata:
#   name: backend-ingress
#   namespace: k8s-learning-app
#   annotations:
#     nginx.ingress.kubernetes.io/rewrite-target: /
#     cert-manager.io/cluster-issuer: "letsencrypt-staging"
#     nginx.ingress.kubernetes.io/ssl-redirect: "true"
#     nginx.ingress.kubernetes.io/cors-allow-origin: "*"
#     nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
#     nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range"
# spec:
#   ingressClassName: nginx
#   tls:
#   - hosts:
#     - api.your-cluster.duckdns.org
#     secretName: backend-tls
#   rules:
#   - host: api.your-cluster.duckdns.org
#     http:
#       paths:
#       - path: /
#         pathType: Prefix
#         backend:
#           service:
#             name: backend-service
#             port:
#               number: 3000
# EOF

# Apply the backend ingress
kubectl apply -f backend-ingress.yaml
```

### Update Frontend Configuration for External API

```bash
# Update frontend configmap to use external API endpoint
# Choose the option that matches your DNS setup:

# Option A: For your own domain
kubectl patch configmap frontend-config -n k8s-learning-app -p='
{
  "data": {
    "REACT_APP_API_URL": "https://api.k8s.yourdomain.com"
  }
}'

# Option B: For DuckDNS
# kubectl patch configmap frontend-config -n k8s-learning-app -p='
# {
#   "data": {
#     "REACT_APP_API_URL": "https://api.your-cluster.duckdns.org"
#   }
# }'

# Restart frontend deployment to pick up new config
kubectl rollout restart deployment/frontend -n k8s-learning-app
```

## Step 5: Verify External Access

### Check Certificate Generation

```bash
# Check certificate status
kubectl get certificates -n k8s-learning-app

# Check certificate details
kubectl describe certificate frontend-tls -n k8s-learning-app
kubectl describe certificate backend-tls -n k8s-learning-app

# Check certificate issuer challenges
kubectl get challenges -n k8s-learning-app

# If certificates are not ready, check cert-manager logs
kubectl logs -n cert-manager -l app.kubernetes.io/instance=cert-manager
```

### Test External Access

```bash
# Test HTTP to HTTPS redirect (replace with your domain)
# For your own domain:
curl -I http://k8s.yourdomain.com

# For DuckDNS:
# curl -I http://your-cluster.duckdns.org

# Test HTTPS access (may show certificate warning with staging issuer)
# For your own domain:
curl -k https://k8s.yourdomain.com

# For DuckDNS:
# curl -k https://your-cluster.duckdns.org

# Test API endpoint
# For your own domain:
curl -k https://api.k8s.yourdomain.com/health

# For DuckDNS:
# curl -k https://api.your-cluster.duckdns.org/health

# Check ingress status
kubectl get ingress -n k8s-learning-app
kubectl describe ingress frontend-ingress -n k8s-learning-app
```

### Switch to Production Certificates (Once Testing Works)

```bash
# Update ingress to use production issuer
kubectl patch ingress frontend-ingress -n k8s-learning-app -p='
{
  "metadata": {
    "annotations": {
      "cert-manager.io/cluster-issuer": "letsencrypt-production"
    }
  }
}'

kubectl patch ingress backend-ingress -n k8s-learning-app -p='
{
  "metadata": {
    "annotations": {
      "cert-manager.io/cluster-issuer": "letsencrypt-production"
    }
  }
}'

# Delete existing certificates to force renewal with production issuer
kubectl delete certificate frontend-tls backend-tls -n k8s-learning-app

# Wait for new certificates to be issued
kubectl wait --for=condition=ready certificate frontend-tls -n k8s-learning-app --timeout=300s
kubectl wait --for=condition=ready certificate backend-tls -n k8s-learning-app --timeout=300s
```

## Step 6: Implement Network Security Policies

### Enable Network Policy Support

Flannel doesn't support NetworkPolicies by default. For learning purposes, we'll install Calico alongside Flannel for policy support:

```bash
# Install Calico for network policies (alongside Flannel)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico-policy-only.yaml

# Wait for Calico to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=300s
```

### Create Network Policies

```bash
# Create default deny-all policy
cat > network-policies.yaml << 'EOF'
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: k8s-learning-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow ingress controller to reach frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: k8s-learning-app
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
---
# Allow ingress controller to reach backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-backend
  namespace: k8s-learning-app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
---
# Allow backend to reach database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: k8s-learning-app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
      podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
---
# Allow database ingress from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-from-backend
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: k8s-learning-app
      podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Label namespaces for network policies
kubectl label namespace ingress-nginx name=ingress-nginx
kubectl label namespace k8s-learning-app name=k8s-learning-app
kubectl label namespace database name=database

# Apply network policies
kubectl apply -f network-policies.yaml

# Verify network policies
kubectl get networkpolicies -A
```

## Step 7: Set Up Monitoring Dashboard

### Create Simple Monitoring Ingress

```bash
# Expose metrics-server via ingress (for demonstration)
cat > monitoring-ingress.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: metrics-server-external
  namespace: kube-system
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - name: https
    port: 443
    targetPort: 4443
    protocol: TCP
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - Monitoring'
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - monitoring.your-cluster.duckdns.org
    secretName: monitoring-tls
  rules:
  - host: monitoring.your-cluster.duckdns.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: metrics-server-external
            port:
              number: 443
EOF

# Create basic auth secret for monitoring
htpasswd -c auth admin  # You'll be prompted for a password
kubectl create secret generic basic-auth --from-file=auth -n kube-system
rm auth

# Apply monitoring ingress
kubectl apply -f monitoring-ingress.yaml
```

## Step 8: Security Hardening

### Implement Pod Security Standards

```bash
# Create pod security policy
cat > pod-security.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-learning-app-secure
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Security context for applications
apiVersion: v1
kind: SecurityContext
spec:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  fsGroup: 1001
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop:
    - ALL
EOF
```

### Create Security-Hardened Application Deployment

```bash
# Create secure version of backend deployment
cat > backend-secure.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-secure
  namespace: k8s-learning-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-secure
  template:
    metadata:
      labels:
        app: backend-secure
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: backend
        image: k8s-learning-backend:v1.1.0
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        envFrom:
        - configMapRef:
            name: backend-config
        - secretRef:
            name: backend-secrets
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: tmp
        emptyDir: {}
EOF

# Apply secure deployment (optional)
# kubectl apply -f backend-secure.yaml
```

## Step 9: Create External Access Monitoring

### Create External Access Health Check Script

```bash
cat > external-access-check.sh << 'EOF'
#!/bin/bash

# Configuration - Update these with your actual domains
# Choose ONE set based on your DNS setup:

# Option A: For your own domain
DOMAIN="k8s.yourdomain.com"  # Replace with your actual domain
API_DOMAIN="api.k8s.yourdomain.com"

# Option B: For DuckDNS (comment out the above and uncomment below)
# DOMAIN="your-cluster.duckdns.org"
# API_DOMAIN="api.your-cluster.duckdns.org"

echo "=== External Access Health Check ==="
echo "Testing domains: ${DOMAIN} and ${API_DOMAIN}"
echo

echo "1. DNS Resolution:"
echo "Frontend: $(dig +short ${DOMAIN})"
echo "API: $(dig +short ${API_DOMAIN})"
echo

echo "2. HTTP to HTTPS Redirect:"
HTTP_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://${DOMAIN})
echo "HTTP Response Code: ${HTTP_RESPONSE}"
echo

echo "3. HTTPS Certificate:"
echo | openssl s_client -servername ${DOMAIN} -connect ${DOMAIN}:443 2>/dev/null | openssl x509 -noout -subject -dates
echo

echo "4. Frontend Application:"
FRONTEND_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" https://${DOMAIN})
echo "Frontend Response Code: ${FRONTEND_RESPONSE}"
echo

echo "5. API Health Check:"
API_HEALTH=$(curl -s https://${API_DOMAIN}/health | jq -r '.status' 2>/dev/null || echo "unreachable")
echo "API Health Status: ${API_HEALTH}"
echo

echo "6. Ingress Controller Status:"
kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller --no-headers | awk '{print $1 ": " $3}'
echo

echo "7. Certificate Status:"
kubectl get certificates -n k8s-learning-app --no-headers | awk '{print $1 ": " $2}'
echo

echo "8. Current Public IP:"
CURRENT_IP=$(curl -s https://checkip.amazonaws.com/)
echo "Public IP: ${CURRENT_IP}"
echo

echo "=== External Access Check Complete ==="
EOF

chmod +x external-access-check.sh

# Run the check
./external-access-check.sh
```

## Troubleshooting

### Common Issues

**Certificates not issuing:**
```bash
# Check cert-manager logs
kubectl logs -n cert-manager -l app.kubernetes.io/instance=cert-manager

# Check ACME challenges
kubectl get challenges -A
kubectl describe challenge <challenge-name> -n k8s-learning-app

# Verify DNS resolution from outside
dig your-cluster.duckdns.org

# Check if ports 80/443 are accessible from internet
# Use online port checkers or ask a friend to test
```

**Ingress not routing traffic:**
```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Verify ingress resources
kubectl get ingress -A
kubectl describe ingress frontend-ingress -n k8s-learning-app

# Check service endpoints
kubectl get endpoints -n k8s-learning-app
```

**Network policies blocking traffic:**
```bash
# Temporarily disable network policies for testing
kubectl delete networkpolicies --all -n k8s-learning-app

# Check Calico status
kubectl get pods -n kube-system -l k8s-app=calico-node

# View network policy logs (if available)
kubectl logs -n kube-system -l k8s-app=calico-node
```

### Router Configuration Issues

**Port forwarding not working:**
1. Verify router has public IP (not behind CGNAT)
2. Check if ISP blocks ports 80/443
3. Test with different ports (8080/8443)
4. Verify router firewall settings
5. Check router logs for dropped connections

## Next Steps

With external access and security configured, you're ready for **Phase 6: GitOps & CI/CD**. You should now have:

- ✅ NGINX Ingress Controller installed and configured
- ✅ SSL/TLS certificates from Let's Encrypt
- ✅ Dynamic DNS for home network access
- ✅ External HTTPS access to frontend and API
- ✅ Network security policies implemented
- ✅ Basic security hardening applied
- ✅ External access monitoring tools

**Key Commands for External Access Management:**
```bash
# Check external access
./external-access-check.sh

# Monitor certificates
kubectl get certificates -A

# Check ingress status
kubectl get ingress -A

# View ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Test network policies
kubectl get networkpolicies -A
```

**External URLs:**

**Option A: Your Own Domain (recommended):**
- Frontend: https://k8s.yourdomain.com or https://www.k8s.yourdomain.com
- API: https://api.k8s.yourdomain.com
- Monitoring: https://monitoring.k8s.yourdomain.com

**Option B: DuckDNS (alternative):**
- Frontend: https://your-cluster.duckdns.org
- API: https://api.your-cluster.duckdns.org
- Monitoring: https://monitoring.your-cluster.duckdns.org

**Important Files Created:**
- `external-access-check.sh` - External access health monitoring
- Various ingress and security policy YAML files
- Dynamic DNS update script