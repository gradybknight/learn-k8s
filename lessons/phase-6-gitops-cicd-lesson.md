# Phase 6 Lesson: GitOps & CI/CD Concepts

## Introduction: Infrastructure as Code and Declarative Operations

Phase 6 represents a fundamental shift from manual operations to automated, code-driven infrastructure management. GitOps isn't just about automation—it's about applying software engineering best practices to infrastructure and operations, creating a reliable, auditable, and scalable approach to managing Kubernetes workloads.

This phase embodies the evolution from "pets" (manually managed servers) to "cattle" (programmatically managed infrastructure) at the operational level, where infrastructure changes follow the same rigor as application code changes.

## GitOps Philosophy and Principles

### Core GitOps Principles

**1. Declarative Infrastructure**
GitOps treats infrastructure configuration as code, stored in version control systems. Instead of imperative commands ("create this pod"), you declare desired state ("this pod should exist with these specifications").

```yaml
# Declarative approach - desired state
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: frontend
        image: myapp/frontend:v1.2.3
```

**2. Git as Single Source of Truth**
All infrastructure configuration lives in Git repositories. The cluster state should reflect what's defined in Git, and changes happen through Git workflows (pull requests, reviews, merges).

**3. Automated Synchronization**
Controllers continuously monitor Git repositories and automatically apply changes to maintain desired state. This eliminates configuration drift and ensures consistency.

**4. Operational Visibility**
All changes are tracked through Git history, providing complete audit trails, rollback capabilities, and change attribution.

### GitOps vs. Traditional CI/CD

**Traditional CI/CD (Push Model)**
```
Developer → Git Push → CI System → Deploy to Cluster
```
Problems:
- **Cluster credentials**: CI system needs direct cluster access
- **Security concerns**: Secrets management in CI systems
- **Limited observability**: Hard to track deployment state
- **Recovery complexity**: Difficult to handle failed deployments

**GitOps (Pull Model)**
```
Developer → Git Push → GitOps Controller (in cluster) → Apply Changes
```
Benefits:
- **Credential isolation**: No external systems need cluster access
- **Self-healing**: Controller continuously reconciles desired state
- **Change visibility**: All changes tracked in Git history
- **Disaster recovery**: Cluster can be rebuilt from Git state

## ArgoCD Architecture and Concepts

### ArgoCD Components

**ArgoCD Server**
- **API Server**: REST API for managing applications and projects
- **Web UI**: Dashboard for visualizing and managing deployments
- **RBAC**: Role-based access control for teams and environments
- **SSO Integration**: Authentication with external identity providers

**Application Controller**
- **State Reconciliation**: Continuously compares Git state with cluster state
- **Sync Operations**: Applies changes to maintain desired state
- **Health Assessment**: Monitors application health and status
- **Rollback Capabilities**: Can revert to previous known-good states

**Repository Server**
- **Git Operations**: Clones and monitors Git repositories
- **Manifest Generation**: Processes Helm charts, Kustomize, or plain YAML
- **Caching**: Optimizes performance by caching repository contents
- **Multi-Source**: Can combine multiple Git repositories for applications

**Redis**
- **Caching Layer**: Stores application state and repository cache
- **Session Management**: Handles user sessions and temporary data
- **Performance**: Reduces load on Git repositories and API server

### Application and Project Concepts

**ArgoCD Application**
An Application defines how to deploy a specific workload:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app
spec:
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: apps/frontend
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**ArgoCD Project**
Projects provide multi-tenancy and governance:
- **Repository restrictions**: Which Git repositories can be used
- **Destination restrictions**: Which clusters and namespaces are allowed
- **Resource restrictions**: Which Kubernetes resources can be managed
- **RBAC policies**: Who can access and modify applications

### Sync Strategies and Policies

**Manual Sync**
- **Human approval**: Changes require manual intervention
- **Change review**: Operators can review changes before applying
- **Risk mitigation**: Prevents unexpected changes in critical environments
- **Compliance**: Meets regulatory requirements for change approval

**Automated Sync**
```yaml
syncPolicy:
  automated:
    prune: true      # Remove resources not in Git
    selfHeal: true   # Correct manual changes automatically
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
```

**Sync Windows**
```yaml
syncWindows:
- kind: allow
  schedule: "0 9-17 * * MON-FRI"  # Business hours only
  duration: 8h
  applications:
  - frontend-app
```

## Repository Structure and Patterns

### Mono-repo vs. Multi-repo Strategies

**Application Repository (App Repo)**
Contains application source code and CI pipeline:
```
myapp/
├── src/                 # Application source code
├── Dockerfile          # Container build instructions
├── package.json        # Dependencies and scripts
└── .github/workflows/  # CI pipeline (build, test, push)
```

**Configuration Repository (Config Repo)**
Contains Kubernetes manifests and deployment configurations:
```
k8s-manifests/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── backend/
├── infrastructure/
│   ├── ingress/
│   └── monitoring/
└── environments/
    ├── staging/
    └── production/
```

**Benefits of Separation**
- **Security**: Application developers don't need cluster access
- **Governance**: Infrastructure changes follow different approval processes
- **Stability**: Application changes don't affect infrastructure configuration
- **Team boundaries**: Clear separation of responsibilities

### Environment Management Patterns

**Environment-Specific Overlays**
```
environments/
├── base/                # Common configuration
│   ├── deployment.yaml
│   └── kustomization.yaml
├── staging/            # Staging overrides
│   ├── kustomization.yaml
│   └── replica-count.yaml
└── production/         # Production overrides
    ├── kustomization.yaml
    ├── replica-count.yaml
    └── resource-limits.yaml
```

**Branch-Based Environments**
- **Feature branches**: Individual developer environments
- **Staging branch**: Integration testing environment
- **Main branch**: Production environment

**Advantages and Trade-offs**
- **Branch-based**: Simple model, clear promotion path
- **Directory-based**: All environments visible, easier cross-environment changes
- **Hybrid approach**: Branches for promotion, directories for configuration variants

## CI/CD Pipeline Design

### Continuous Integration (CI) Pipeline

**Build Stage**
```yaml
name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    - name: Install dependencies
      run: npm ci
    - name: Run tests
      run: npm test
    - name: Run linting
      run: npm run lint
```

**Security Scanning**
```yaml
    - name: Run security audit
      run: npm audit --audit-level high
    - name: Container image scanning
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp/frontend:${{ github.sha }}
        format: 'sarif'
```

**Image Building and Publishing**
```yaml
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          myregistry/myapp:${{ github.sha }}
          myregistry/myapp:latest
```

### Continuous Deployment (CD) Pipeline

**Configuration Update Pattern**
```yaml
  update-manifest:
    needs: [test, build]
    runs-on: ubuntu-latest
    steps:
    - name: Checkout config repo
      uses: actions/checkout@v3
      with:
        repository: myorg/k8s-manifests
        token: ${{ secrets.CONFIG_REPO_TOKEN }}
    
    - name: Update image tag
      run: |
        sed -i 's|image: myapp/frontend:.*|image: myapp/frontend:${{ github.sha }}|g' \
          apps/frontend/deployment.yaml
    
    - name: Commit and push changes
      run: |
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Update frontend image to ${{ github.sha }}"
        git push
```

**Pull Request-Based Updates**
```yaml
    - name: Create pull request
      uses: peter-evans/create-pull-request@v5
      with:
        token: ${{ secrets.CONFIG_REPO_TOKEN }}
        repository: myorg/k8s-manifests
        branch: update-frontend-${{ github.sha }}
        title: "Update frontend to ${{ github.sha }}"
        body: |
          Auto-generated update from CI pipeline
          - Image: myapp/frontend:${{ github.sha }}
          - Commit: ${{ github.sha }}
```

### Security and Secrets Management

**GitHub Secrets vs. Kubernetes Secrets**
- **GitHub Secrets**: For CI/CD pipeline credentials (registry access, Git tokens)
- **Kubernetes Secrets**: For application runtime secrets (database passwords, API keys)
- **External Secret Operators**: Integrate with external secret stores (HashiCorp Vault, AWS Secrets Manager)

**Image Security Best Practices**
```yaml
# Use specific base image versions
FROM node:18.17.0-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Copy and install dependencies as non-root
USER nextjs
COPY --chown=nextjs:nodejs package*.json ./
RUN npm ci --only=production

# Set security headers
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

## Advanced GitOps Patterns

### Multi-Environment Promotion

**Progressive Delivery**
```
Development → Staging → Canary → Production
```

**Automated Promotion Criteria**
```yaml
# .github/workflows/promote.yml
on:
  schedule:
    - cron: '0 */6 * * *'  # Check every 6 hours

jobs:
  promote:
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Check staging health
      run: |
        # Verify all health checks pass
        # Confirm zero error rate for 24 hours
        # Validate performance metrics
    
    - name: Promote to production
      if: success()
      run: |
        # Update production manifest
        # Create promotion PR
```

### Application of Applications Pattern

**App-of-Apps Structure**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-bootstrap
spec:
  source:
    repoURL: https://github.com/myorg/cluster-config
    path: applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: {}
```

**Benefits**
- **Centralized management**: Single point to deploy all applications
- **Dependency ordering**: Control application deployment sequence
- **Environment consistency**: Ensure all environments have same application set
- **Disaster recovery**: Rebuild entire cluster from single Git repository

### Config Management Tools Integration

**Kustomize Integration**
```yaml
# ArgoCD Application with Kustomize
spec:
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: apps/frontend
    kustomize:
      images:
      - myapp/frontend:v1.2.3
      namePrefix: prod-
      commonLabels:
        environment: production
```

**Helm Integration**
```yaml
# ArgoCD Application with Helm
spec:
  source:
    repoURL: https://github.com/myorg/helm-charts
    path: charts/myapp
    helm:
      valueFiles:
      - values-production.yaml
      parameters:
      - name: image.tag
        value: v1.2.3
      - name: replicas
        value: "5"
```

## Monitoring and Observability for GitOps

### Deployment Metrics

**Key GitOps Metrics**
- **Deployment frequency**: How often deployments occur
- **Lead time**: Time from code commit to production deployment
- **Mean time to recovery (MTTR)**: Time to recover from failed deployments
- **Change failure rate**: Percentage of deployments causing issues

**ArgoCD Metrics**
```yaml
# Prometheus ServiceMonitor for ArgoCD
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
```

### Alerting on GitOps Events

**Critical Alerts**
```yaml
# ArgoCD sync failure alert
- alert: ArgoCDSyncFailed
  expr: argocd_app_health_status{health_status!="Healthy"} == 1
  for: 5m
  annotations:
    summary: "ArgoCD application {{ $labels.name }} sync failed"
    description: "Application {{ $labels.name }} has been in unhealthy state for more than 5 minutes"

# Drift detection alert
- alert: ArgoCDOutOfSync
  expr: argocd_app_sync_total{phase!="Succeeded"} > 0
  for: 10m
  annotations:
    summary: "ArgoCD application {{ $labels.name }} is out of sync"
```

### Change Tracking and Audit

**Git-based Audit Trail**
Every change is tracked through Git:
- **Who**: Git commit author and ArgoCD user
- **What**: Exact configuration changes in diff format
- **When**: Git commit timestamp and ArgoCD sync time
- **Why**: Commit messages and pull request descriptions

**Compliance Reporting**
```bash
# Generate deployment report
git log --since="2023-01-01" --until="2023-12-31" \
  --pretty=format:"%h,%an,%ad,%s" \
  --date=iso apps/production/ > deployment-report.csv
```

## Disaster Recovery and Backup

### GitOps-Based Disaster Recovery

**Cluster Rebuild Process**
1. **Bootstrap ArgoCD**: Install ArgoCD in new cluster
2. **Add application-of-applications**: Point to Git repository
3. **Automatic restoration**: ArgoCD deploys all applications from Git
4. **Data restoration**: Restore persistent data from backups

**Recovery Time Benefits**
- **Infrastructure as Code**: Entire cluster configuration in Git
- **Automated restoration**: No manual configuration steps
- **Consistent environments**: Production identical to what's in Git
- **Rapid scaling**: Can create identical environments quickly

### Backup Strategies

**Configuration Backup**
```bash
# Backup ArgoCD applications
kubectl get applications -n argocd -o yaml > argocd-applications-backup.yaml

# Backup cluster configuration
kubectl get configmaps,secrets,rbac -o yaml > cluster-config-backup.yaml
```

**Git Repository Backup**
- **Multiple remotes**: Push to multiple Git hosting providers
- **Regular archives**: Automated Git bundle creation
- **Mirror repositories**: Keep synchronized copies of critical repositories

## Advanced Security Considerations

### RBAC and Multi-Tenancy

**Project-Based Isolation**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
spec:
  description: Frontend team applications
  sourceRepos:
  - 'https://github.com/myorg/frontend-*'
  destinations:
  - namespace: frontend-*
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Service
  - group: 'apps'
    kind: Deployment
```

**Role-Based Access Control**
```yaml
# ArgoCD RBAC configuration
policy.default: role:readonly
policy.csv: |
  p, role:admin, applications, *, */*, allow
  p, role:developer, applications, get, team-frontend/*, allow
  p, role:developer, applications, sync, team-frontend/*, allow
  g, frontend-team, role:developer
```

### Supply Chain Security

**Signed Commits and Tags**
```bash
# Configure Git signing
git config user.signingkey <GPG-KEY-ID>
git config commit.gpgsign true

# Sign releases
git tag -s v1.2.3 -m "Release version 1.2.3"
```

**Image Signing and Verification**
```yaml
# Cosign policy for ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    plugin:
      name: cosign-verify
      parameters:
      - name: cosign-key
        value: cosign.pub
```

**Admission Controllers**
```yaml
# OPA Gatekeeper policy
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: requiredlabels
spec:
  crd:
    spec:
      validation:
        properties:
          labels:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requiredlabels
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Missing required label: %v", [missing])
        }
```

## Preparing for Phase 7

### Advanced Automation Foundation

Phase 6's GitOps implementation provides the foundation for Phase 7's advanced topics:
- **Automated deployments**: Enable advanced deployment strategies (canary, blue-green)
- **Configuration management**: Support for complex multi-environment configurations
- **Change tracking**: Audit trail for compliance and troubleshooting
- **Self-healing infrastructure**: Automatic recovery from configuration drift

### Operational Excellence

**Monitoring and Alerting Integration**
- **Deployment metrics**: Track deployment frequency and success rates
- **Application health**: Monitor application status through GitOps controller
- **Change correlation**: Link application issues to recent deployments
- **Capacity planning**: Track resource usage trends across deployments

**Team Productivity**
- **Self-service deployments**: Developers can deploy without operations team
- **Environment consistency**: Identical configuration across all environments
- **Rapid rollbacks**: Quick recovery from problematic deployments
- **Change visibility**: All stakeholders can see deployment status and history

By completing Phase 6, you've implemented a production-grade GitOps workflow that brings software engineering rigor to infrastructure management, enabling rapid, reliable, and auditable deployments while maintaining security and operational visibility.