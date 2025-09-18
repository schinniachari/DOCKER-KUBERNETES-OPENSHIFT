# IBM Docker Kubernetes OpenShift - Complete Guide with Examples

## Table of Contents
1. [Introduction](#introduction)
2. [Docker Fundamentals with Examples](#docker-fundamentals)
3. [Kubernetes Core Concepts with Manifests](#kubernetes-core-concepts)
4. [OpenShift Platform with Configurations](#openshift-platform)
5. [Sample Applications with Full Code](#sample-applications)
6. [Hands-on Labs with Step-by-Step Instructions](#hands-on-labs)
7. [Best Practices with Implementation Examples](#best-practices)
8. [Troubleshooting Guide with Commands](#troubleshooting)

## Introduction

This comprehensive guide combines theoretical explanations with practical code examples from the IBM Docker/Kubernetes/OpenShift repository, providing both understanding and implementation details.

## Docker Fundamentals with Examples

### What is Docker?
Docker is a platform for developing, shipping, and running applications in containers. Containers package an application with all its dependencies, ensuring consistency across environments.

### Dockerfile Examples with Explanations

**Simple Node.js Application Dockerfile:**
```dockerfile
# Use official Node.js runtime as base image
FROM node:14-alpine

# Set working directory inside container
WORKDIR /app

# Copy package files first to leverage Docker cache
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application source code
COPY . .

# Expose application port
EXPOSE 3000

# Command to run the application
CMD ["npm", "start"]
```

**Explanation:** This Dockerfile uses a multi-stage approach where package files are copied first to optimize layer caching. The Alpine Linux base keeps the image small and secure.

**Multi-stage Build for Production:**
```dockerfile
# Build stage - includes build tools
FROM node:16 as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production stage - minimal image
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

# Run as non-root user for security
USER node

EXPOSE 3000
CMD ["node", "server.js"]
```

**Explanation:** Multi-stage builds reduce final image size by separating build dependencies from runtime environment, enhancing security and efficiency.

### Docker Compose for Multi-container Apps

**docker-compose.yml with Explanation:**
```yaml
# Docker Compose file for multi-container application
# Defines web application, Redis cache, and PostgreSQL database services
version: '3.8'
services:
  # Web application service - main application container
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - REDIS_HOST=redis
    depends_on:
      - redis
    volumes:
      - ./logs:/app/logs
    networks:
      - app-network

  # Redis cache service - in-memory data store for caching
  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    networks:
      - app-network

  # Database service - persistent data storage
  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

# Named volumes for persistent data storage
volumes:
  redis_data:
  postgres_data:

# Custom network for inter-container communication
networks:
  app-network:
    driver: bridge
```

**Explanation:** This Compose file defines a complete microservices environment with networking, volume persistence, and service dependencies.

## Kubernetes Core Concepts with Manifests

### Kubernetes Architecture Overview
Kubernetes is a portable, extensible open-source platform for managing containerized workloads. It provides service discovery, load balancing, storage orchestration, automated rollouts and rollbacks, and self-healing capabilities.

### Complete Deployment Manifest

**web-deployment.yaml with Annotations:**
```yaml
# Kubernetes Deployment for web application
# Manages replica set of pods and provides rolling updates
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
    environment: production
spec:
  replicas: 3
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
        tier: frontend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      containers:
      - name: web-app
        image: my-registry/web-app:v1.2.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      restartPolicy: Always
```

**Explanation:** This deployment includes resource limits, health checks, config maps, and rollout strategy for production-ready applications.

### Service and Ingress Configuration

**Complete Service Definition:**
```yaml
# Kubernetes Service for web application
# Provides stable network endpoint and load balancing for pods
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
  type: LoadBalancer
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

**Ingress with TLS Termination:**
```yaml
# Kubernetes Ingress for external access
# Manages external HTTP/HTTPS routing to services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## OpenShift Platform with Configurations

### OpenShift Specific Resources

**BuildConfig for Source-to-Image:**
```yaml
# OpenShift BuildConfig for automated builds
# Defines how to build container images from source code
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: nodejs-app
  labels:
    app: nodejs-app
    component: backend
spec:
  triggers:
  - type: ConfigChange
  - type: ImageChange
  - type: GitHub
    github:
      secret: github-webhook-secret
  runPolicy: Serial
  source:
    git:
      uri: https://github.com/your-org/nodejs-app.git
      ref: main
    contextDir: src
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:16-ubi8
        namespace: openshift
      env:
      - name: NPM_RUN
        value: "build"
  output:
    to:
      kind: ImageStreamTag
      name: nodejs-app:latest
  resources:
    limits:
      cpu: "2"
      memory: 4Gi
    requests:
      cpu: "1"
      memory: 2Gi
```

**Route with Advanced Configuration:**
```yaml
# OpenShift Route for external access
# Provides HTTP/HTTPS routing with TLS termination
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nodejs-app-route
  annotations:
    haproxy.router.openshift.io/balance: roundrobin
    haproxy.router.openshift.io/disable_cookies: "true"
    openshift.io/host.generated: "false"
spec:
  host: app.example.com
  to:
    kind: Service
    name: nodejs-app-service
    weight: 100
  port:
    targetPort: 3000-tcp
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
    certificate: |
      -----BEGIN CERTIFICATE-----
      MIIE... [certificate content]
      -----END CERTIFICATE-----
    key: |
      -----BEGIN PRIVATE KEY-----
      MIIE... [private key content]
      -----END PRIVATE KEY-----
  wildcardPolicy: None
```

## Configuration Management

### ConfigMap Example

**configmap.yaml:**
```yaml
# Kubernetes ConfigMap for application configuration
# Stores non-sensitive configuration data as key-value pairs
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app: web-app
    environment: production
data:
  # Environment variables
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  
  # JSON configuration file
  config.json: |
    {
      "database": {
        "host": "db-service",
        "port": 5432,
        "timeout": 30000
      },
      "cache": {
        "enabled": true,
        "ttl": 3600,
        "maxSize": 1000
      },
      "server": {
        "port": 3000,
        "timeout": 30000,
        "bodyLimit": "10mb"
      }
    }
  
  # Nginx configuration snippet
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            proxy_pass http://web-service:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
        }
    }
```

### Secret Example

**secret.yaml:**
```yaml
# Kubernetes Secret for sensitive data
# Stores encrypted sensitive information like passwords and API keys
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  labels:
    app: web-app
    sensitive: "true"
type: Opaque
data:
  # Base64 encoded values - decode with: echo <value> | base64 -d
  database-password: c2VjcmV0LXBhc3N3b3JkCg==  # secret-password
  api-key: YXBpLWtleS1zZWNyZXQK                # api-key-secret
  jwt-secret: anN0LXNlY3JldC1rZXkK            # jst-secret-key
  ssl-certificate: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JS...  # SSL cert
stringData:
  # Non-base64 values (for development only)
  database-url: "postgresql://user:password@db-host:5432/mydb"
  external-api-url: "https://api.external-service.com/v1"
```

## Monitoring and Logging

### ServiceMonitor for Prometheus

**servicemonitor.yaml:**
```yaml
# Prometheus ServiceMonitor for application metrics
# Configures Prometheus to scrape metrics from the application
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app-monitor
  labels:
    app: web-app
    team: backend
    monitoring: "true"
spec:
  # Selector to match services
  selector:
    matchLabels:
      app: web-app
      metrics: "true"
  
  # Namespace selector
  namespaceSelector:
    matchNames:
    - my-app
    - monitoring
  
  # Endpoints to scrape
  endpoints:
  - port: http-metrics        # Port name defined in service
    interval: 30s             # Scrape interval
    scrapeTimeout: 10s        # Timeout for scraping
    path: /metrics            # Metrics endpoint path
    scheme: http              # HTTP scheme
    honorLabels: true         # Preserve original labels
    
    # TLS configuration (optional)
    tlsConfig:
      insecureSkipVerify: true
    
    # Relabeling rules
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod_name
      action: replace
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: namespace
      action: replace
  
  # Additional labels for the metrics
  targetLabels:
  - app
  - environment
  - tier
```

## Sample Applications with Full Code

### Complete Node.js Microservice

**package.json:**
```json
{
  "name": "ibm-container-app",
  "version": "1.0.0",
  "description": "Sample application for IBM Container training",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest --coverage",
    "lint": "eslint .",
    "build": "npm run lint && npm test"
  },
  "dependencies": {
    "express": "^4.18.0",
    "redis": "^4.0.0",
    "mongoose": "^6.0.0",
    "helmet": "^5.0.0",
    "cors": "^2.8.5",
    "express-rate-limit": "^6.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0",
    "jest": "^28.0.0",
    "eslint": "^8.0.0",
    "supertest": "^6.0.0"
  },
  "engines": {
    "node": ">=14.0.0",
    "npm": ">=6.0.0"
  }
}
```

**server.js - Complete Application:**
```javascript
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const rateLimit = require('express-rate-limit');
const redis = require('redis');

const app = express();
const PORT = process.env.PORT || 3000;

// Security middleware
app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '10kb' }));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});
app.use(limiter);

// Redis client with connection handling
const redisClient = redis.createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
  retry_strategy: (options) => {
    if (options.error && options.error.code === 'ECONNREFUSED') {
      return new Error('The server refused the connection');
    }
    if (options.total_retry_time > 1000 * 60 * 60) {
      return new Error('Retry time exhausted');
    }
    if (options.attempt > 10) {
      return undefined;
    }
    return Math.min(options.attempt * 100, 3000);
  }
});

redisClient.on('connect', () => {
  console.log('Connected to Redis');
});

redisClient.on('error', (err) => {
  console.error('Redis error:', err);
});

// Health check endpoint
app.get('/health', async (req, res) => {
  const healthcheck = {
    uptime: process.uptime(),
    message: 'OK',
    timestamp: Date.now(),
    redis: redisClient.connected ? 'connected' : 'disconnected'
  };
  
  try {
    res.status(200).json(healthcheck);
  } catch (error) {
    healthcheck.message = error;
    res.status(503).json(healthcheck);
  }
});

// Main endpoint
app.get('/', async (req, res) => {
  try {
    const visits = await new Promise((resolve, reject) => {
      redisClient.incr('visits', (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
    
    res.json({
      message: 'Welcome to IBM Container Application',
      visits: visits,
      server: process.env.HOSTNAME || 'local',
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(500).json({ error: 'Redis operation failed' });
  }
});

// API routes
app.get('/api/users', (req, res) => {
  res.json([{ id: 1, name: 'John Doe' }, { id: 2, name: 'Jane Smith' }]);
});

app.post('/api/users', (req, res) => {
  const { name, email } = req.body;
  // In a real application, you would save to database
  res.status(201).json({ id: Date.now(), name, email, created: new Date() });
});

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Endpoint not found' });
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  redisClient.quit();
  server.close(() => {
    console.log('Process terminated');
  });
});

// Start server
const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Health check available at http://localhost:${PORT}/health`);
});

module.exports = app; // For testing
```

## Best Practices with Implementation Examples

### Security Hardening

**Security Context in Deployment:**
```yaml
# Security-hardened Deployment configuration
# Implements security best practices and least privilege principles
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  labels:
    security: hardened
spec:
  template:
    spec:
      # Pod-level security context
      securityContext:
        runAsNonRoot: true          # Prevent running as root
        runAsUser: 1000             # Non-root user ID
        runAsGroup: 1000            # Non-root group ID
        fsGroup: 2000               # Filesystem group
        seccompProfile:             # Seccomp security profile
          type: RuntimeDefault
        supplementalGroups: [2000]  # Additional groups
        
        # Linux capabilities management
        sysctls:
        - name: net.ipv4.ip_forward
          value: "0"
        
      containers:
      - name: app
        # Container-level security context
        securityContext:
          allowPrivilegeEscalation: false  # No privilege escalation
          capabilities:
            drop: ["ALL"]                  # Drop all capabilities
          readOnlyRootFilesystem: true     # Read-only root filesystem
          runAsNonRoot: true               # Non-root user
          runAsUser: 1000                  # Specific user ID
          privileged: false                # No privileged container
          
          # AppArmor profile (if available)
          appArmorProfile: runtime/default
          
          # SELinux options
          seLinuxOptions:
            level: "s0:c123,c456"
          
          # Resource constraints
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

### Resource Management Best Practices

**Complete Resource Configuration:**
```yaml
# Comprehensive resource management configuration
# Ensures proper resource allocation and prevents resource exhaustion
resources:
  requests:
    memory: "128Mi"        # Guaranteed memory allocation
    cpu: "250m"            # Guaranteed CPU allocation (0.25 cores)
    ephemeral-storage: "1Gi" # Local storage request
    hugepages-2Mi: "64Mi"  # Huge pages allocation (if needed)
    
  limits:
    memory: "256Mi"        # Maximum memory usage
    cpu: "500m"            # Maximum CPU usage (0.5 cores)
    ephemeral-storage: "2Gi" # Maximum local storage
    hugepages-2Mi: "64Mi"  # Huge pages limit
    
  # Quality of Service class (automatically set based on requests/limits)
  # QoS Classes: Guaranteed (limits=requests), Burstable (limits>requests), BestEffort (no limits)
```

## Hands-on Labs with Step-by-Step Instructions

### Lab 1: Complete Docker Workflow

**Objective:** Containerize a Node.js application and deploy it using Docker

**Step 1: Create Project Structure**
```bash
mkdir my-docker-app && cd my-docker-app
npm init -y
npm install express redis helmet cors
```

**Step 2: Create Application Files**
Create `server.js` (as above) and `Dockerfile`:
```dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --only=production
COPY . .
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1
CMD ["node", "server.js"]
```

**Step 3: Build and Run**
```bash
# Build the image
docker build -t my-node-app:1.0.0 .

# Run with environment variables
docker run -d \
  --name my-app \
  -p 3000:3000 \
  -e REDIS_URL=redis://redis-server:6379 \
  -e NODE_ENV=production \
  my-node-app:1.0.0

# Check logs
docker logs my-app

# Test health endpoint
curl http://localhost:3000/health
```

### Lab 2: Kubernetes Deployment

**Objective:** Deploy application to Kubernetes cluster

**Step 1: Create Namespace**
```bash
kubectl create namespace my-app
```

**Step 2: Apply Configuration Files**
Create and apply `deployment.yaml`, `service.yaml`, and `ingress.yaml` as shown in previous sections.

**Step 3: Verify Deployment**
```bash
kubectl get all -n my-app
kubectl describe deployment web-app -n my-app
kubectl logs deployment/web-app -n my-app
```

## Troubleshooting Guide with Commands

### Common Issues and Solutions

**Pod Not Starting:**
```bash
# Check pod status
kubectl get pods -n my-app

# Describe pod for details
kubectl describe pod <pod-name> -n my-app

# Check events in namespace
kubectl get events -n my-app --sort-by='.lastTimestamp'

# Check container logs
kubectl logs <pod-name> -n my-app --previous
```

**Image Pull Errors:**
```bash
# Check image pull secrets
kubectl get secrets -n my-app

# Describe pod for pull errors
kubectl describe pod <pod-name> -n my-app | grep -i "pull"

# Test image pull manually
docker pull my-registry/my-image:tag
```

**Resource Issues:**
```bash
# Check resource usage
kubectl top pods -n my-app
kubectl top nodes

# Check resource quotas
kubectl describe resourcequota -n my-app

# Check limit ranges
kubectl describe limitrange -n my-app
```

**Network Connectivity:**
```bash
# Check service endpoints
kubectl get endpoints -n my-app

# Test service connectivity
kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm
curl http://web-service.my-app.svc.cluster.local

# Check network policies
kubectl get networkpolicies -n my-app
```

---
