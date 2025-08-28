# Phase 4: Application Development & Deployment

## Learning Objectives
- Build a Node.js/TypeScript backend API with database connectivity
- Create a React/TypeScript frontend application
- Containerize both applications with Docker
- Deploy applications to Kubernetes using Deployments and Services
- Understand ConfigMaps, Secrets, and environment variables
- Implement health checks and resource management
- Set up application-to-database communication

## Prerequisites
- Completed Phase 3: Essential Cluster Components
- PostgreSQL database running in the cluster
- Persistent storage configured
- Basic understanding of Node.js and React

## Project Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Architecture                 │
│                                                             │
│  ┌─────────────┐    HTTP/REST    ┌─────────────────────────┐ │
│  │   React     │───────────────▶ │     Node.js API        │ │
│  │  Frontend   │                 │    (TypeScript)        │ │
│  │             │◀─────────────── │  /health, /hello       │ │
│  └─────────────┘    JSON         └─────────────────────────┘ │
│        │                                      │             │
│        │                                      │ SQL         │
│        ▼                                      ▼             │
│  ┌─────────────┐                    ┌─────────────────────┐ │
│  │  NGINX      │                    │    PostgreSQL      │ │
│  │ (Static     │                    │    Database        │ │
│  │  Files)     │                    │                    │ │
│  └─────────────┘                    └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Create Development Directory Structure

```bash
# Run all commands in this section on your development machine (laptop/desktop)
# Create project directory on your development machine
mkdir -p ~/k8s-learning-apps
cd ~/k8s-learning-apps

# Create application directories
mkdir -p backend frontend k8s-manifests

# Initialize Git repository
git init
```

## Step 2: Build Backend Application

### Initialize Node.js Backend

```bash
cd backend

# Initialize npm project
npm init -y

# Install dependencies
npm install express cors helmet morgan dotenv
npm install pg  # PostgreSQL client
npm install --save-dev @types/node @types/express @types/cors @types/morgan @types/pg
npm install --save-dev typescript ts-node nodemon

# Create TypeScript configuration
npx tsc --init --target es2020 --module commonjs --outDir dist --rootDir src --strict --esModuleInterop
```

### Create Backend Source Code

```bash
# Create source directory
mkdir -p src

# Create main application file
cat > src/app.ts << 'EOF'
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import { Pool } from 'pg';

const app = express();
const port = process.env.PORT || 3000;

// Database configuration
const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME || 'appdb',
  user: process.env.DB_USER || 'appuser',
  password: process.env.DB_PASSWORD || 'secure-password-123',
});

// Middleware
app.use(helmet());
app.use(cors());
app.use(morgan('combined'));
app.use(express.json());

// Health check endpoint
app.get('/health', async (req, res) => {
  try {
    // Check database connectivity
    const client = await pool.connect();
    await client.query('SELECT 1');
    client.release();
    
    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      database: 'connected',
      version: process.env.npm_package_version || '1.0.0'
    });
  } catch (error) {
    console.error('Health check failed:', error);
    res.status(503).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      database: 'disconnected',
      error: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

// Hello endpoint with database interaction
app.get('/hello', async (req, res) => {
  try {
    const client = await pool.connect();
    
    // Insert a greeting record
    await client.query(`
      INSERT INTO greetings (message, timestamp) 
      VALUES ($1, $2)
    `, ['Hello from Kubernetes!', new Date()]);
    
    // Get recent greetings
    const result = await client.query(`
      SELECT id, message, timestamp 
      FROM greetings 
      ORDER BY timestamp DESC 
      LIMIT 10
    `);
    
    client.release();
    
    res.json({
      message: 'Hello from Kubernetes API!',
      timestamp: new Date().toISOString(),
      environment: process.env.NODE_ENV || 'development',
      hostname: process.env.HOSTNAME,
      recentGreetings: result.rows
    });
  } catch (error) {
    console.error('Hello endpoint error:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

// Initialize database schema
async function initializeDatabase() {
  try {
    const client = await pool.connect();
    
    await client.query(`
      CREATE TABLE IF NOT EXISTS greetings (
        id SERIAL PRIMARY KEY,
        message TEXT NOT NULL,
        timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );
    `);
    
    client.release();
    console.log('Database schema initialized');
  } catch (error) {
    console.error('Database initialization failed:', error);
    process.exit(1);
  }
}

// Start server
app.listen(port, async () => {
  console.log(`Backend API listening on port ${port}`);
  await initializeDatabase();
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  pool.end(() => {
    process.exit(0);
  });
});

export default app;
EOF

# Create package.json scripts
cat > package.json << 'EOF'
{
  "name": "k8s-learning-backend",
  "version": "1.0.0",
  "description": "Backend API for Kubernetes learning project",
  "main": "dist/app.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/app.js",
    "dev": "nodemon --exec ts-node src/app.ts",
    "test": "echo \"No tests yet\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "helmet": "^7.0.0",
    "morgan": "^1.10.0",
    "dotenv": "^16.3.1",
    "pg": "^8.11.3"
  },
  "devDependencies": {
    "@types/node": "^20.5.0",
    "@types/express": "^4.17.17",
    "@types/cors": "^2.8.13",
    "@types/morgan": "^1.9.4",
    "@types/pg": "^8.10.2",
    "typescript": "^5.1.6",
    "ts-node": "^10.9.1",
    "nodemon": "^3.0.1"
  }
}
EOF
```

### Create Backend Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of app directory
RUN chown -R nodejs:nodejs /app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) }).on('error', () => process.exit(1))"

# Start application
CMD ["npm", "start"]
EOF

# Create .dockerignore
cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
dist
.git
.gitignore
README.md
.env
.nyc_output
coverage
.coverage
EOF
```

## Step 3: Build Frontend Application

### Initialize React Frontend

```bash
cd ../frontend

# Create React app with TypeScript
npx create-react-app . --template typescript

# Install additional dependencies
npm install axios

# Remove unnecessary files
rm -rf public/favicon.ico public/logo192.png public/logo512.png public/manifest.json public/robots.txt
```

### Create Frontend Source Code

```bash
# Replace src/App.tsx
cat > src/App.tsx << 'EOF'
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './App.css';

interface HealthResponse {
  status: string;
  timestamp: string;
  database: string;
  version: string;
}

interface HelloResponse {
  message: string;
  timestamp: string;
  environment: string;
  hostname: string;
  recentGreetings: Array<{
    id: number;
    message: string;
    timestamp: string;
  }>;
}

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:3000';

function App() {
  const [health, setHealth] = useState<HealthResponse | null>(null);
  const [hello, setHello] = useState<HelloResponse | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const checkHealth = async () => {
    try {
      setLoading(true);
      setError(null);
      const response = await axios.get<HealthResponse>(`${API_BASE_URL}/health`);
      setHealth(response.data);
    } catch (err) {
      setError('Failed to check health');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  const sayHello = async () => {
    try {
      setLoading(true);
      setError(null);
      const response = await axios.get<HelloResponse>(`${API_BASE_URL}/hello`);
      setHello(response.data);
    } catch (err) {
      setError('Failed to say hello');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    checkHealth();
  }, []);

  return (
    <div className="App">
      <header className="App-header">
        <h1>Kubernetes Learning App</h1>
        <p>Frontend React Application</p>
        
        <div className="button-container">
          <button onClick={checkHealth} disabled={loading}>
            {loading ? 'Checking...' : 'Check Health'}
          </button>
          <button onClick={sayHello} disabled={loading}>
            {loading ? 'Loading...' : 'Say Hello'}
          </button>
        </div>

        {error && (
          <div className="error">
            <p>Error: {error}</p>
          </div>
        )}

        {health && (
          <div className="health-status">
            <h2>Backend Health Status</h2>
            <p><strong>Status:</strong> <span className={health.status === 'healthy' ? 'healthy' : 'unhealthy'}>{health.status}</span></p>
            <p><strong>Database:</strong> <span className={health.database === 'connected' ? 'healthy' : 'unhealthy'}>{health.database}</span></p>
            <p><strong>Version:</strong> {health.version}</p>
            <p><strong>Last Check:</strong> {new Date(health.timestamp).toLocaleString()}</p>
          </div>
        )}

        {hello && (
          <div className="hello-response">
            <h2>Hello Response</h2>
            <p><strong>Message:</strong> {hello.message}</p>
            <p><strong>Environment:</strong> {hello.environment}</p>
            <p><strong>Hostname:</strong> {hello.hostname}</p>
            <p><strong>Timestamp:</strong> {new Date(hello.timestamp).toLocaleString()}</p>
            
            {hello.recentGreetings.length > 0 && (
              <div className="recent-greetings">
                <h3>Recent Greetings</h3>
                <ul>
                  {hello.recentGreetings.map((greeting) => (
                    <li key={greeting.id}>
                      {greeting.message} - {new Date(greeting.timestamp).toLocaleString()}
                    </li>
                  ))}
                </ul>
              </div>
            )}
          </div>
        )}
      </header>
    </div>
  );
}

export default App;
EOF

# Update CSS
cat > src/App.css << 'EOF'
.App {
  text-align: center;
}

.App-header {
  background-color: #282c34;
  padding: 20px;
  color: white;
  min-height: 100vh;
}

.button-container {
  margin: 20px 0;
}

.button-container button {
  margin: 0 10px;
  padding: 10px 20px;
  font-size: 16px;
  background-color: #61dafb;
  color: #282c34;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.button-container button:hover:not(:disabled) {
  background-color: #21a0c4;
}

.button-container button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.health-status, .hello-response {
  background-color: #3d4149;
  margin: 20px auto;
  padding: 20px;
  border-radius: 8px;
  max-width: 600px;
  text-align: left;
}

.healthy {
  color: #4caf50;
  font-weight: bold;
}

.unhealthy {
  color: #f44336;
  font-weight: bold;
}

.error {
  background-color: #f44336;
  color: white;
  padding: 10px;
  border-radius: 4px;
  margin: 20px auto;
  max-width: 600px;
}

.recent-greetings ul {
  list-style-type: none;
  padding: 0;
}

.recent-greetings li {
  background-color: #4a4d56;
  margin: 5px 0;
  padding: 8px;
  border-radius: 4px;
}
EOF

# Update public/index.html
cat > public/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta name="description" content="Kubernetes Learning Application" />
    <title>K8s Learning App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
EOF
```

### Create Frontend Dockerfile

```bash
# Create multi-stage Dockerfile for production
cat > Dockerfile << 'EOF'
# Build stage
FROM node:18-alpine as build

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build the app
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy built app to nginx
COPY --from=build /app/build /usr/share/nginx/html

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Create non-root user
RUN addgroup -g 1001 -S nginx && \
    adduser -S nginx -u 1001 -G nginx

# Change ownership of nginx directories
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    chown -R nginx:nginx /etc/nginx/conf.d

# Create nginx pid directory
RUN mkdir -p /var/run/nginx && \
    chown -R nginx:nginx /var/run/nginx

# Switch to non-root user
USER nginx

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ || exit 1

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
EOF

# Create nginx configuration
cat > nginx.conf << 'EOF'
server {
    listen 8080;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Serve static files
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF

# Create .dockerignore
cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
build
.git
.gitignore
README.md
.env
coverage
.nyc_output
EOF
```

## Step 4: Build and Push Container Images

### Build Images Locally (for testing)

```bash
# Run these Docker commands on your development machine where Docker is installed
# Build backend image
cd ../backend
docker build -t k8s-learning-backend:v1.0.0 .

# Build frontend image
cd ../frontend
docker build -t k8s-learning-frontend:v1.0.0 .

# Test images locally (optional)
# docker run --rm -p 3000:3000 k8s-learning-backend:v1.0.0
# docker run --rm -p 8080:8080 k8s-learning-frontend:v1.0.0
```

### Tag and Push Images to a Registry

For this learning setup, we'll use a local registry or save images directly to nodes:

```bash
# Option 1: Save images and load them on cluster nodes
cd ..
docker save k8s-learning-backend:v1.0.0 | gzip > backend-v1.0.0.tar.gz
docker save k8s-learning-frontend:v1.0.0 | gzip > frontend-v1.0.0.tar.gz

# Copy to cluster nodes
scp backend-v1.0.0.tar.gz k8sadmin@k8s-worker-1:/tmp/
scp backend-v1.0.0.tar.gz k8sadmin@k8s-worker-2:/tmp/
scp frontend-v1.0.0.tar.gz k8sadmin@k8s-worker-1:/tmp/
scp frontend-v1.0.0.tar.gz k8sadmin@k8s-worker-2:/tmp/

# Load images on each worker node
ssh k8sadmin@k8s-worker-1 'sudo ctr -n k8s.io images import /tmp/backend-v1.0.0.tar.gz'
ssh k8sadmin@k8s-worker-1 'sudo ctr -n k8s.io images import /tmp/frontend-v1.0.0.tar.gz'
ssh k8sadmin@k8s-worker-2 'sudo ctr -n k8s.io images import /tmp/backend-v1.0.0.tar.gz'
ssh k8sadmin@k8s-worker-2 'sudo ctr -n k8s.io images import /tmp/frontend-v1.0.0.tar.gz'

# Verify images are loaded
ssh k8sadmin@k8s-worker-1 'sudo ctr -n k8s.io images list | grep k8s-learning'
```

## Step 5: Create Kubernetes Manifests

### Create Application Namespace

```bash
cd k8s-manifests

# Create namespace
cat > namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-learning-app
  labels:
    name: k8s-learning-app
EOF

kubectl apply -f namespace.yaml
```

### Create ConfigMaps and Secrets

```bash
# Create ConfigMap for backend configuration
cat > backend-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: k8s-learning-app
data:
  NODE_ENV: "production"
  PORT: "3000"
  DB_HOST: "postgres-service.database.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "appdb"
  DB_USER: "appuser"
---
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
  namespace: k8s-learning-app
type: Opaque
data:
  DB_PASSWORD: c2VjdXJlLXBhc3N3b3JkLTEyMw==  # base64 encoded "secure-password-123"
EOF

kubectl apply -f backend-config.yaml
```

### Create Backend Deployment and Service

```bash
cat > backend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: k8s-learning-app
  labels:
    app: backend
    tier: api
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
        image: k8s-learning-backend:v1.0.0
        imagePullPolicy: Never  # Use local image
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
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: k8s-learning-app
  labels:
    app: backend
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

kubectl apply -f backend-deployment.yaml
```

### Create Frontend Deployment and Service

```bash
cat > frontend-deployment.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: k8s-learning-app
data:
  REACT_APP_API_URL: "http://backend-service.k8s-learning-app.svc.cluster.local:3000"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: k8s-learning-app
  labels:
    app: frontend
    tier: web
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
        image: k8s-learning-frontend:v1.0.0
        imagePullPolicy: Never  # Use local image
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
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: k8s-learning-app
  labels:
    app: frontend
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

kubectl apply -f frontend-deployment.yaml
```

## Step 6: Deploy and Verify Applications

### Deploy Applications

```bash
# Apply all manifests
kubectl apply -f .

# Wait for deployments to be ready
kubectl wait --for=condition=available deployment/backend -n k8s-learning-app --timeout=300s
kubectl wait --for=condition=available deployment/frontend -n k8s-learning-app --timeout=300s
```

### Verify Deployment

```bash
# Check all resources
kubectl get all -n k8s-learning-app

# Check pod logs
kubectl logs -l app=backend -n k8s-learning-app --tail=20
kubectl logs -l app=frontend -n k8s-learning-app --tail=20

# Check service endpoints
kubectl get endpoints -n k8s-learning-app

# Describe deployments
kubectl describe deployment backend -n k8s-learning-app
kubectl describe deployment frontend -n k8s-learning-app
```

### Test Internal Communication

```bash
# Test backend health from within cluster
kubectl run test-pod --rm -i --tty --image=curlimages/curl -- sh
# Inside the pod:
# curl http://backend-service.k8s-learning-app.svc.cluster.local:3000/health
# curl http://backend-service.k8s-learning-app.svc.cluster.local:3000/hello
# exit

# Test frontend from within cluster
kubectl run nginx-test --rm -i --tty --image=nginx:alpine -- sh
# Inside the pod:
# wget -qO- http://frontend-service.k8s-learning-app.svc.cluster.local:8080/health
# exit
```

### Set Up Port Forwarding for Testing

```bash
# Forward backend port to local machine
kubectl port-forward service/backend-service 3000:3000 -n k8s-learning-app &

# Forward frontend port to local machine
kubectl port-forward service/frontend-service 8080:8080 -n k8s-learning-app &

# Test endpoints locally
curl http://localhost:3000/health
curl http://localhost:3000/hello

# Open frontend in browser
# http://localhost:8080

# Stop port forwarding
pkill -f "kubectl port-forward"
```

## Step 7: Implement Rolling Updates

### Update Backend Application

```bash
cd ../backend

# Make a small change to the API response
sed -i 's/Hello from Kubernetes!/Hello from Kubernetes v1.1!/g' src/app.ts

# Build new version
npm run build
docker build -t k8s-learning-backend:v1.1.0 .

# Save and load on cluster nodes
cd ..
docker save k8s-learning-backend:v1.1.0 | gzip > backend-v1.1.0.tar.gz
scp backend-v1.1.0.tar.gz k8sadmin@k8s-worker-1:/tmp/
scp backend-v1.1.0.tar.gz k8sadmin@k8s-worker-2:/tmp/
ssh k8sadmin@k8s-worker-1 'sudo ctr -n k8s.io images import /tmp/backend-v1.1.0.tar.gz'
ssh k8sadmin@k8s-worker-2 'sudo ctr -n k8s.io images import /tmp/backend-v1.1.0.tar.gz'
```

### Perform Rolling Update

```bash
cd k8s-manifests

# Update the image in deployment
kubectl patch deployment backend -n k8s-learning-app -p='{"spec":{"template":{"spec":{"containers":[{"name":"backend","image":"k8s-learning-backend:v1.1.0"}]}}}}'

# Watch the rolling update
kubectl rollout status deployment/backend -n k8s-learning-app

# Verify the update
kubectl get pods -n k8s-learning-app -l app=backend
kubectl port-forward service/backend-service 3000:3000 -n k8s-learning-app &
curl http://localhost:3000/hello  # Should show v1.1 message
pkill -f "kubectl port-forward"
```

### Rollback if Needed

```bash
# View rollout history
kubectl rollout history deployment/backend -n k8s-learning-app

# Rollback to previous version
kubectl rollout undo deployment/backend -n k8s-learning-app

# Or rollback to specific revision
# kubectl rollout undo deployment/backend --to-revision=1 -n k8s-learning-app
```

## Step 8: Monitoring and Troubleshooting

### Create Application Monitoring Script

```bash
cat > ../app-monitor.sh << 'EOF'
#!/bin/bash

echo "=== Application Status Monitor ==="
echo

echo "1. Namespace Resources:"
kubectl get all -n k8s-learning-app
echo

echo "2. Pod Status and Node Distribution:"
kubectl get pods -n k8s-learning-app -o wide
echo

echo "3. Service Endpoints:"
kubectl get endpoints -n k8s-learning-app
echo

echo "4. Resource Usage:"
kubectl top pods -n k8s-learning-app 2>/dev/null || echo "Metrics not available"
echo

echo "5. Recent Events:"
kubectl get events -n k8s-learning-app --sort-by='.lastTimestamp' | tail -10
echo

echo "6. Backend Health Check:"
kubectl run health-check --rm -i --tty --image=curlimages/curl --restart=Never -- \
  curl -s http://backend-service.k8s-learning-app.svc.cluster.local:3000/health || echo "Health check failed"
echo

echo "=== Monitor Complete ==="
EOF

chmod +x ../app-monitor.sh
../app-monitor.sh
```

## Troubleshooting

### Common Issues

**Pods stuck in Pending state:**
```bash
# Check node resources
kubectl describe nodes

# Check pod events
kubectl describe pod <pod-name> -n k8s-learning-app

# Check resource requests vs available
kubectl top nodes
```

**Image pull errors:**
```bash
# Verify images exist on nodes
ssh k8sadmin@k8s-worker-1 'sudo ctr -n k8s.io images list | grep k8s-learning'

# Check pod events for image pull errors
kubectl describe pod <pod-name> -n k8s-learning-app
```

**Database connection issues:**
```bash
# Test database connectivity
kubectl exec -it <backend-pod> -n k8s-learning-app -- sh
# Inside pod: nc -zv postgres-service.database.svc.cluster.local 5432

# Check DNS resolution
kubectl exec dns-utils -- nslookup postgres-service.database.svc.cluster.local
```

### Log Analysis

```bash
# View application logs
kubectl logs -f deployment/backend -n k8s-learning-app
kubectl logs -f deployment/frontend -n k8s-learning-app

# View previous container logs (if pod restarted)
kubectl logs <pod-name> -n k8s-learning-app --previous

# Stream logs from all backend pods
kubectl logs -f -l app=backend -n k8s-learning-app
```

## Next Steps

With applications deployed and running, you're ready for **Phase 5: External Access & Security**. You should now have:

- ✅ Node.js/TypeScript backend API with database connectivity
- ✅ React/TypeScript frontend application
- ✅ Both applications containerized and deployed to Kubernetes
- ✅ Inter-service communication working
- ✅ Health checks and resource limits configured
- ✅ Rolling updates and rollback capability
- ✅ Application monitoring tools

**Key Commands for Application Management:**
```bash
# Monitor applications
../app-monitor.sh

# Scale applications
kubectl scale deployment backend --replicas=3 -n k8s-learning-app

# Update applications
kubectl patch deployment backend -n k8s-learning-app -p='{"spec":{"template":{"spec":{"containers":[{"name":"backend","image":"new-image:tag"}]}}}}'

# View logs
kubectl logs -f -l app=backend -n k8s-learning-app

# Port forward for testing
kubectl port-forward service/frontend-service 8080:8080 -n k8s-learning-app
```

**Project Structure Created:**
```
~/k8s-learning-apps/
├── backend/                 # Node.js/TypeScript API
├── frontend/                # React/TypeScript SPA
├── k8s-manifests/          # Kubernetes YAML files
└── app-monitor.sh          # Application monitoring script
```