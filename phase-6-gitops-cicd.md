# Phase 6: GitOps & CI/CD

## Learning Objectives
- Understand GitOps principles and workflows
- Install and configure ArgoCD
- Set up automated deployments from Git repositories
- Implement CI/CD pipelines with GitHub Actions
- Create separate repositories for application code and Kubernetes manifests
- Configure automated testing, building, and deployment
- Implement environment-specific deployments (staging/production)

## Prerequisites
- Completed Phase 5: External Access & Security
- Applications accessible via HTTPS
- GitHub account or similar Git hosting service
- Basic understanding of Git workflows

## Understanding GitOps Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitOps Workflow                         │
│                                                             │
│  Developer ──▶ Git Repo ──▶ CI Pipeline ──▶ Container      │
│     │            │              │            Registry      │
│     │            │              │               │          │
│     │            │              │               ▼          │
│     │            │              │         ┌─────────────┐  │
│     │            │              │         │  Updated    │  │
│     │            │              │         │  Image      │  │
│     │            │              │         └─────────────┘  │
│     │            │              │               │          │
│     │            ▼              ▼               │          │
│     │      ┌─────────────┐ ┌─────────────┐      │          │
│     │      │   Config    │ │   Image     │      │          │
│     │      │   Repo      │ │   Build     │      │          │
│     │      │(Manifests)  │ │             │      │          │
│     │      └─────────────┘ └─────────────┘      │          │
│     │            │              │               │          │
│     │            │              └───────────────┼──────────┘
│     │            │                              │
│     │            ▼                              ▼
│     │      ┌─────────────────────────────────────────────┐
│     │      │              ArgoCD                        │
│     │      │        (GitOps Controller)                 │
│     │      └─────────────────────────────────────────────┘
│     │                           │
│     │                           ▼
│     │              ┌─────────────────────────┐
│     │              │    Kubernetes Cluster   │
│     │              │                         │
│     │              │  ┌─────┐ ┌─────┐ ┌─────┐│
│     │              │  │ Pod │ │ Pod │ │ Pod ││
│     │              │  └─────┘ └─────┘ └─────┘│
│     │              └─────────────────────────┘
│     │                           │
│     └───────────────────────────┘
│               Feedback Loop
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Set Up Git Repositories

### Create Application Repository

```bash
# On your development machine, create and initialize app repo
cd ~/k8s-learning-apps
git add .
git commit -m "Initial commit - K8s learning applications"

# Create GitHub repository and push
# (Follow GitHub instructions to create repository)
git remote add origin https://github.com/your-username/k8s-learning-apps.git
git branch -M main
git push -u origin main
```

### Create GitOps Configuration Repository

```bash
# Create separate repository for Kubernetes manifests
mkdir -p ~/k8s-learning-gitops
cd ~/k8s-learning-gitops

# Initialize repository structure
mkdir -p {environments/{staging,production},applications/{backend,frontend,database}}

# Create base application configurations
mkdir -p applications/backend/{base,overlays/{staging,production}}
mkdir -p applications/frontend/{base,overlays/{staging,production}}
mkdir -p applications/database/{base,overlays/{staging,production}}

# Initialize git
git init
```

### Create Kustomize Base Configurations

```bash
cd ~/k8s-learning-gitops

# Create backend base configuration
cat > applications/backend/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml
- secret.yaml

commonLabels:
  app: backend
  tier: api
EOF

# Copy current backend manifests to base
cp ~/k8s-learning-apps/k8s-manifests/backend-deployment.yaml applications/backend/base/deployment.yaml
cp ~/k8s-learning-apps/k8s-manifests/backend-config.yaml applications/backend/base/

# Split the backend-config.yaml into separate files
cat > applications/backend/base/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  DB_HOST: "postgres-service.database.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "appdb"
  DB_USER: "appuser"
EOF

cat > applications/backend/base/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
type: Opaque
data:
  DB_PASSWORD: c2VjdXJlLXBhc3N3b3JkLTEyMw==  # base64 encoded
EOF

cat > applications/backend/base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - name: http
    port: 3000
    targetPort: 3000
    protocol: TCP
  type: ClusterIP
EOF

# Update deployment to use consistent labels
cat > applications/backend/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: backend
        image: k8s-learning-backend:latest  # Will be updated by CI/CD
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
          name: http
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
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
      restartPolicy: Always
EOF

# Create staging overlay
cat > applications/backend/overlays/staging/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: k8s-learning-app-staging

resources:
- ../../base

patchesStrategicMerge:
- replica-patch.yaml
- env-patch.yaml

namePrefix: staging-
EOF

cat > applications/backend/overlays/staging/replica-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
EOF

cat > applications/backend/overlays/staging/env-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  NODE_ENV: "staging"
EOF

# Create production overlay
cat > applications/backend/overlays/production/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: k8s-learning-app

resources:
- ../../base

patchesStrategicMerge:
- replica-patch.yaml

namePrefix: prod-
EOF

cat > applications/backend/overlays/production/replica-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
EOF
```

### Create Frontend Base Configuration

```bash
# Create frontend base configuration
cat > applications/frontend/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app: frontend
  tier: web
EOF

cat > applications/frontend/base/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  REACT_APP_API_URL: "https://api.your-cluster.duckdns.org"
EOF

cat > applications/frontend/base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
EOF

cat > applications/frontend/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: k8s-learning-frontend:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: frontend-config
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# Create frontend overlays
cat > applications/frontend/overlays/staging/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: k8s-learning-app-staging

resources:
- ../../base

patchesStrategicMerge:
- env-patch.yaml

namePrefix: staging-
EOF

cat > applications/frontend/overlays/staging/env-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  REACT_APP_API_URL: "https://staging-api.your-cluster.duckdns.org"
EOF

cat > applications/frontend/overlays/production/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: k8s-learning-app

resources:
- ../../base

namePrefix: prod-
EOF
```

### Create Environment Configurations

```bash
# Create staging environment
cat > environments/staging/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- namespace.yaml
- ../../applications/backend/overlays/staging
- ../../applications/frontend/overlays/staging
EOF

cat > environments/staging/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-learning-app-staging
  labels:
    name: k8s-learning-app-staging
    environment: staging
EOF

# Create production environment
cat > environments/production/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../applications/backend/overlays/production
- ../../applications/frontend/overlays/production
EOF

# Commit and push GitOps repository
git add .
git commit -m "Initial GitOps configuration with Kustomize"
git remote add origin https://github.com/your-username/k8s-learning-gitops.git
git branch -M main
git push -u origin main
```

## Step 2: Install ArgoCD

### Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Verify installation
kubectl get pods -n argocd
```

### Configure ArgoCD Access

```bash
# Get initial admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGOCD_PASSWORD"

# Create ingress for ArgoCD
cat > argocd-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    nginx.ingress.kubernetes.io/grpc-backend: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.your-cluster.duckdns.org
    secretName: argocd-tls
  rules:
  - host: argocd.your-cluster.duckdns.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF

kubectl apply -f argocd-ingress.yaml

# Alternative: Port forward for local access
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
echo "ArgoCD available at: https://localhost:8080"
echo "Username: admin"
echo "Password: $ARGOCD_PASSWORD"
```

### Install ArgoCD CLI (Optional)

```bash
# Install ArgoCD CLI on your development machine
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login to ArgoCD
argocd login argocd.your-cluster.duckdns.org --username admin --password $ARGOCD_PASSWORD

# Or login to local port-forward
# argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD --insecure
```

## Step 3: Configure ArgoCD Applications

### Create ArgoCD Application for Staging

```bash
cat > argocd-app-staging.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: k8s-learning-staging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/k8s-learning-gitops.git
    targetRevision: HEAD
    path: environments/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: k8s-learning-app-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF

kubectl apply -f argocd-app-staging.yaml
```

### Create ArgoCD Application for Production

```bash
cat > argocd-app-production.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: k8s-learning-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/k8s-learning-gitops.git
    targetRevision: HEAD
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: k8s-learning-app
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  # Manual sync for production
EOF

kubectl apply -f argocd-app-production.yaml
```

### Verify ArgoCD Applications

```bash
# Check application status
kubectl get applications -n argocd

# View application details
argocd app list
argocd app get k8s-learning-staging
argocd app get k8s-learning-production

# Manually sync if needed
argocd app sync k8s-learning-staging
```

## Step 4: Set Up CI/CD with GitHub Actions

### Create GitHub Actions Workflow for Backend

```bash
cd ~/k8s-learning-apps

# Create GitHub Actions directory
mkdir -p .github/workflows

# Create backend CI/CD workflow
cat > .github/workflows/backend-ci-cd.yaml << 'EOF'
name: Backend CI/CD

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'backend/**'
      - '.github/workflows/backend-ci-cd.yaml'
  pull_request:
    branches: [ main ]
    paths:
      - 'backend/**'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/backend

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: backend/package-lock.json

    - name: Install dependencies
      working-directory: backend
      run: npm ci

    - name: Run tests
      working-directory: backend
      run: npm test

    - name: Run linting
      working-directory: backend
      run: npm run lint || echo "No lint script defined"

    - name: Build TypeScript
      working-directory: backend
      run: npm run build

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    outputs:
      image: ${{ steps.image.outputs.image }}
      digest: ${{ steps.build.outputs.digest }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: backend
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Output image
      id: image
      run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}" >> $GITHUB_OUTPUT

  update-gitops:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: ${{ github.repository_owner }}/k8s-learning-gitops
        token: ${{ secrets.GITOPS_TOKEN }}
        path: gitops

    - name: Update staging image
      run: |
        cd gitops
        sed -i 's|image: k8s-learning-backend:.*|image: ${{ needs.build-and-push.outputs.image }}|' \
          applications/backend/base/deployment.yaml
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Update backend image to ${{ github.sha }}" || exit 0
        git push

  deploy-staging:
    needs: update-gitops
    runs-on: ubuntu-latest
    steps:
    - name: Trigger ArgoCD sync (staging)
      run: |
        echo "ArgoCD will automatically sync staging environment"
        echo "Image: ${{ needs.build-and-push.outputs.image }}"
EOF
```

### Create Frontend CI/CD Workflow

```bash
cat > .github/workflows/frontend-ci-cd.yaml << 'EOF'
name: Frontend CI/CD

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend-ci-cd.yaml'
  pull_request:
    branches: [ main ]
    paths:
      - 'frontend/**'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/frontend

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json

    - name: Install dependencies
      working-directory: frontend
      run: npm ci

    - name: Run tests
      working-directory: frontend
      run: npm test -- --coverage --watchAll=false

    - name: Run linting
      working-directory: frontend
      run: npm run lint || echo "No lint script defined"

    - name: Build application
      working-directory: frontend
      run: npm run build

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    outputs:
      image: ${{ steps.image.outputs.image }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: frontend
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Output image
      id: image
      run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}" >> $GITHUB_OUTPUT

  update-gitops:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: ${{ github.repository_owner }}/k8s-learning-gitops
        token: ${{ secrets.GITOPS_TOKEN }}
        path: gitops

    - name: Update staging image
      run: |
        cd gitops
        sed -i 's|image: k8s-learning-frontend:.*|image: ${{ needs.build-and-push.outputs.image }}|' \
          applications/frontend/base/deployment.yaml
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Update frontend image to ${{ github.sha }}" || exit 0
        git push
EOF
```

### Set Up GitHub Secrets

You'll need to create these secrets in your GitHub repository:

1. **GITOPS_TOKEN**: Personal Access Token with repo access for updating GitOps repository
2. **GITHUB_TOKEN**: Automatically provided by GitHub Actions

```bash
# In your GitHub repository settings, add these secrets:
# 1. Go to Settings > Secrets and variables > Actions
# 2. Add new repository secret:
#    Name: GITOPS_TOKEN
#    Value: [Your GitHub Personal Access Token]

echo "Create a GitHub Personal Access Token with 'repo' scope at:"
echo "https://github.com/settings/tokens"
echo "Then add it as GITOPS_TOKEN secret in your repository"
```

## Step 5: Implement Multi-Environment Strategy

### Create Development Branch Workflow

```bash
# Create feature branch workflow
cat > .github/workflows/feature-branch.yaml << 'EOF'
name: Feature Branch CI

on:
  push:
    branches-ignore: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test-backend:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: backend/package-lock.json
    - run: cd backend && npm ci && npm test && npm run build

  test-frontend:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json
    - run: cd frontend && npm ci && npm test -- --watchAll=false && npm run build

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run security audit
      run: |
        cd backend && npm audit --audit-level high
        cd ../frontend && npm audit --audit-level high
EOF
```

### Create Production Deployment Workflow

```bash
cat > .github/workflows/production-deploy.yaml << 'EOF'
name: Production Deployment

on:
  workflow_dispatch:
    inputs:
      backend_image:
        description: 'Backend image tag to deploy'
        required: true
        type: string
      frontend_image:
        description: 'Frontend image tag to deploy'
        required: true
        type: string

jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment: production
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: ${{ github.repository_owner }}/k8s-learning-gitops
        token: ${{ secrets.GITOPS_TOKEN }}
        path: gitops

    - name: Update production images
      run: |
        cd gitops
        
        # Update backend image
        sed -i 's|image: .*backend:.*|image: ${{ inputs.backend_image }}|' \
          applications/backend/base/deployment.yaml
        
        # Update frontend image
        sed -i 's|image: .*frontend:.*|image: ${{ inputs.frontend_image }}|' \
          applications/frontend/base/deployment.yaml
        
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Deploy to production: backend ${{ inputs.backend_image }}, frontend ${{ inputs.frontend_image }}"
        git push

    - name: Create deployment record
      run: |
        echo "Production deployment completed at $(date)"
        echo "Backend: ${{ inputs.backend_image }}"
        echo "Frontend: ${{ inputs.frontend_image }}"
EOF
```

## Step 6: Configure ArgoCD Notifications

### Set Up Slack/Discord Notifications (Optional)

```bash
# Create ArgoCD notification configuration
cat > argocd-notifications.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  config.yaml: |
    triggers:
      - name: on-sync-succeeded
        condition: app.status.operationState.phase in ['Succeeded']
        template: app-sync-succeeded
      - name: on-sync-failed
        condition: app.status.operationState.phase in ['Error', 'Failed']
        template: app-sync-failed
    templates:
      - name: app-sync-succeeded
        title: "Application {{.app.metadata.name}} sync succeeded"
        body: |
          Application {{.app.metadata.name}} sync is succeeded.
          Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
      - name: app-sync-failed
        title: "Application {{.app.metadata.name}} sync failed"
        body: |
          Application {{.app.metadata.name}} sync is failed.
          Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
    subscriptions:
      - recipients:
        - webhook:my-webhook
        triggers:
        - on-sync-succeeded
        - on-sync-failed
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
data:
  webhook-url: # base64 encoded webhook URL
EOF

# Apply notifications (uncomment and configure webhook URL first)
# kubectl apply -f argocd-notifications.yaml
```

## Step 7: Create GitOps Monitoring Dashboard

### Create GitOps Status Script

```bash
cat > gitops-status.sh << 'EOF'
#!/bin/bash

echo "=== GitOps Status Dashboard ==="
echo

echo "1. ArgoCD Applications:"
kubectl get applications -n argocd -o custom-columns="NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REVISION:.status.sync.revision"
echo

echo "2. Application Sync Status:"
for app in $(kubectl get applications -n argocd -o name); do
    app_name=$(echo $app | cut -d'/' -f2)
    sync_status=$(kubectl get $app -n argocd -o jsonpath='{.status.sync.status}')
    health_status=$(kubectl get $app -n argocd -o jsonpath='{.status.health.status}')
    revision=$(kubectl get $app -n argocd -o jsonpath='{.status.sync.revision}' | cut -c1-8)
    echo "$app_name: Sync=$sync_status, Health=$health_status, Revision=$revision"
done
echo

echo "3. Recent ArgoCD Events:"
kubectl get events -n argocd --sort-by='.lastTimestamp' | tail -5
echo

echo "4. Staging Environment:"
kubectl get pods -n k8s-learning-app-staging -o wide 2>/dev/null || echo "Staging namespace not found"
echo

echo "5. Production Environment:"
kubectl get pods -n k8s-learning-app -o wide
echo

echo "6. Image Versions in Use:"
echo "Staging Backend:"
kubectl get deployment -n k8s-learning-app-staging -o jsonpath='{.items[*].spec.template.spec.containers[*].image}' 2>/dev/null | tr ' ' '\n' | grep backend || echo "Not deployed"
echo "Production Backend:"
kubectl get deployment -n k8s-learning-app -o jsonpath='{.items[*].spec.template.spec.containers[*].image}' | tr ' ' '\n' | grep backend || echo "Not deployed"
echo

echo "=== GitOps Status Complete ==="
EOF

chmod +x gitops-status.sh
./gitops-status.sh
```

## Step 8: Test GitOps Workflow

### Test Automated Deployment

```bash
# Make a change to the backend application
cd ~/k8s-learning-apps/backend/src
sed -i 's/Hello from Kubernetes v1.1!/Hello from GitOps v1.2!/g' app.ts

# Commit and push the change
cd ~/k8s-learning-apps
git add .
git commit -m "Update greeting message to test GitOps"
git push origin main

# Monitor the CI/CD pipeline
echo "Watch GitHub Actions at: https://github.com/your-username/k8s-learning-apps/actions"
echo "Watch ArgoCD at: https://argocd.your-cluster.duckdns.org"

# Monitor ArgoCD sync
watch kubectl get applications -n argocd
```

### Test Manual Production Deployment

```bash
# Get latest built images
echo "Check GitHub Container Registry for latest images:"
echo "https://github.com/your-username/k8s-learning-apps/pkgs/container/backend"

# Trigger manual production deployment via GitHub Actions
echo "Go to: https://github.com/your-username/k8s-learning-apps/actions/workflows/production-deploy.yaml"
echo "Click 'Run workflow' and specify image tags"

# Or manually sync production in ArgoCD
argocd app sync k8s-learning-production
```

## Troubleshooting

### Common GitOps Issues

**ArgoCD can't access Git repository:**
```bash
# Check ArgoCD server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server

# Verify repository credentials
argocd repo list
```

**GitHub Actions failing:**
```bash
# Check workflow logs in GitHub
# Verify secrets are configured correctly
# Check container registry permissions
```

**Image pull errors in cluster:**
```bash
# For public repositories, update imagePullPolicy
kubectl patch deployment backend -n k8s-learning-app-staging -p='{"spec":{"template":{"spec":{"containers":[{"name":"backend","imagePullPolicy":"Always"}]}}}}'

# For private repositories, create imagePullSecrets
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=your-username \
  --docker-password=your-token \
  -n k8s-learning-app-staging
```

**Kustomize build errors:**
```bash
# Test kustomize builds locally
cd ~/k8s-learning-gitops
kustomize build environments/staging
kustomize build environments/production

# Check for syntax errors in YAML files
kubectl apply --dry-run=client -f environments/staging
```

## Next Steps

With GitOps and CI/CD configured, you're ready for **Phase 7: Advanced Topics**. You should now have:

- ✅ GitOps workflow with ArgoCD
- ✅ Separate repositories for code and configuration
- ✅ Automated CI/CD pipelines with GitHub Actions
- ✅ Multi-environment deployments (staging/production)
- ✅ Container registry integration
- ✅ Automated testing and building
- ✅ GitOps monitoring and status tracking

**Key Commands for GitOps Management:**
```bash
# Monitor GitOps status
./gitops-status.sh

# ArgoCD CLI commands
argocd app list
argocd app sync k8s-learning-staging
argocd app get k8s-learning-production

# Manual image updates
kubectl patch deployment backend -n k8s-learning-app -p='{"spec":{"template":{"spec":{"containers":[{"name":"backend","image":"new-image:tag"}]}}}}'

# Check ArgoCD
kubectl get applications -n argocd
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```

**GitOps Repository Structure:**
```
k8s-learning-gitops/
├── applications/
│   ├── backend/
│   │   ├── base/
│   │   └── overlays/{staging,production}/
│   └── frontend/
│       ├── base/
│       └── overlays/{staging,production}/
└── environments/
    ├── staging/
    └── production/
```

**Workflow URLs:**
- ArgoCD: https://argocd.your-cluster.duckdns.org
- GitHub Actions: https://github.com/your-username/k8s-learning-apps/actions
- Container Registry: https://github.com/your-username/k8s-learning-apps/pkgs