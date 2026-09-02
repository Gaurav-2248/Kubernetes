--

## Table of Contents
1. [Understanding Kubernetes Objects](#understanding-kubernetes-objects)
2. [Pods - The Foundation](#pods-the-foundation)
3. [ReplicaSets - High Availability](#replicasets-high-availability)
4. [Deployments - Production Ready](#deployments-production-ready)
5. [Services - Networking & Discovery](#services-networking-discovery)
6. [Labels & Selectors](#labels-selectors)
7. [Namespaces - Isolation](#namespaces-isolation)

---

## 1. Understanding Kubernetes Objects

### What are Kubernetes Objects?

**Definition:** Kubernetes objects are persistent entities in the Kubernetes system that represent the state of your cluster.

Think of objects as **records of intent** - you tell Kubernetes what you want (desired state), and Kubernetes works to maintain that state.

### Object Anatomy

Every Kubernetes object has the same basic structure:

```yaml
apiVersion: v1              # Which version of K8s API
kind: Pod                   # Type of object
metadata:                   # Data about the object
  name: my-app
  labels:
    app: web
spec:                       # Desired state specification
  containers:
  - name: nginx
    image: nginx:1.21
```

**Four Key Fields:**

1. **apiVersion** - Which API version to use
   - `v1` - Core API (Pod, Service, ConfigMap)
   - `apps/v1` - Apps API (Deployment, StatefulSet, DaemonSet)
   - `batch/v1` - Batch API (Job, CronJob)
   - `networking.k8s.io/v1` - Networking (Ingress, NetworkPolicy)

2. **kind** - Type of object
   - Pod, Deployment, Service, ConfigMap, Secret, etc.

3. **metadata** - Identifying information
   - name (required)
   - labels (key-value pairs for organization)
   - annotations (non-identifying metadata)
   - namespace (which namespace the object belongs to)

4. **spec** - Desired state
   - Different for each object type
   - Describes what you want

### Declarative vs Imperative

**Imperative (Commands):**
```bash
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx
kubectl scale deployment web --replicas=3
```
- Quick and easy
- Good for learning/testing
- Hard to track changes
- Not version controlled

**Declarative (YAML files):**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f .  # Apply all YAML files in directory
```
- Recommended for production
- Version controlled (Git)
- Reproducible
- Can see history of changes

**Best Practice:** Use declarative approach with YAML files stored in Git.

### Working with YAML Files

```bash
# Apply configuration
kubectl apply -f myapp.yaml

# View what would be applied (dry run)
kubectl apply -f myapp.yaml --dry-run=client

# Generate YAML from imperative command
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deployment.yaml

# View existing object as YAML
kubectl get pod my-pod -o yaml

# Edit object directly
kubectl edit pod my-pod

# Delete using YAML
kubectl delete -f myapp.yaml

# Apply all files in directory
kubectl apply -f ./configs/
```

---

## 2. Pods - The Foundation

### What is a Pod?

**Pod** = The smallest and simplest unit in Kubernetes.

**Key Concepts:**
- A pod wraps one or more containers
- Containers in a pod share:
  - Network namespace (same IP address)
  - Storage volumes
  - Lifecycle (created/destroyed together)
- Pods are **ephemeral** (temporary, disposable)
- Each pod gets a unique IP address

### Analogy

Think of a Pod as a **house**:
- The house (pod) has one address
- Multiple people (containers) can live in the house
- They share the same address and utilities
- If the house is demolished, everyone moves out

### Single-Container Pod (Most Common)

**YAML Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
    environment: production
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

**Create the Pod:**
```bash
# From YAML file
kubectl apply -f nginx-pod.yaml

# Imperative command
kubectl run nginx-pod --image=nginx:1.21

# Check pod status
kubectl get pods

# Output:
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          30s
```

### Pod Lifecycle

```
┌─────────────────────────────────────────────────┐
│           POD LIFECYCLE PHASES                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Pending  →  Running  →  Succeeded/Failed      │
│     ↓                          ↓               │
│  ContainerCreating         Terminating          │
│                                ↓               │
│                            Completed            │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Pod Status:**

1. **Pending** - Accepted but not yet running
   - Downloading image
   - Waiting for resources
   - Scheduling

2. **Running** - Bound to node, containers created
   - At least one container is running

3. **Succeeded** - All containers terminated successfully
   - Won't restart

4. **Failed** - All containers terminated, at least one failed
   - Exit code != 0

5. **Unknown** - Cannot determine pod state
   - Communication error with node

### Multi-Container Pod

**When to use:**
- Sidecar pattern (logging, monitoring)
- Ambassador pattern (proxy)
- Adapter pattern (normalize output)

**Example: Web Server + Log Collector**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
  # Main container
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  # Sidecar container
  - name: log-collector
    image: fluent/fluentd
    volumeMounts:
    - name: shared-logs
      mountPath: /logs
  
  # Shared volume
  volumes:
  - name: shared-logs
    emptyDir: {}
```

**How it works:**
```
┌─────────────────────────────────┐
│         Pod: web-with-logging   │
│                                 │
│  ┌──────────┐    ┌───────────┐ │
│  │  Nginx   │    │  Fluentd  │ │
│  │Container │    │ Container │ │
│  └────┬─────┘    └─────┬─────┘ │
│       │                │       │
│       └────────┬────────┘       │
│                │                │
│         ┌──────▼──────┐        │
│         │Shared Volume│        │
│         │   (Logs)    │        │
│         └─────────────┘        │
└─────────────────────────────────┘

Both containers share the same IP and can communicate via localhost
```

### Pod Commands

```bash
# Create pod
kubectl run mypod --image=nginx

# Get pods
kubectl get pods
kubectl get pods -o wide  # Shows more details (Node, IP)
kubectl get pods -A       # All namespaces

# Describe pod (detailed info)
kubectl describe pod mypod

# View logs
kubectl logs mypod
kubectl logs mypod -f     # Follow/stream logs
kubectl logs mypod -c container-name  # Multi-container pod

# Execute command in pod
kubectl exec mypod -- ls /
kubectl exec -it mypod -- /bin/bash  # Interactive shell

# Port forwarding (access pod from localhost)
kubectl port-forward mypod 8080:80
# Now access: http://localhost:8080

# Copy files to/from pod
kubectl cp mypod:/path/to/file ./local-file
kubectl cp ./local-file mypod:/path/to/file

# Delete pod
kubectl delete pod mypod

# Delete with YAML
kubectl delete -f pod.yaml
```

### Pod with Resource Limits

**Why set limits?**
- Prevent one pod from consuming all resources
- Ensure fair resource distribution
- Enable proper scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:          # Minimum guaranteed resources
        memory: "64Mi"
        cpu: "250m"      # 250 millicores = 0.25 CPU
      limits:            # Maximum allowed resources
        memory: "128Mi"
        cpu: "500m"
```

**Resource Units:**
- **CPU**: `1` = 1 vCPU/Core, `500m` = 0.5 CPU, `100m` = 0.1 CPU
- **Memory**: `Mi` (Mebibyte), `Gi` (Gibibyte)
  - `128Mi` = 128 Mebibytes
  - `1Gi` = 1 Gibibyte

### Pod with Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DATABASE_HOST
      value: "mysql.default.svc.cluster.local"
    - name: DATABASE_PORT
      value: "3306"
    - name: ENVIRONMENT
      value: "production"
```

### Init Containers

**Purpose:** Run before main containers start. Used for setup tasks.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  # Init containers (run first, in order)
  initContainers:
  - name: init-database
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for db; sleep 2; done;']
  
  # Main containers (run after init containers complete)
  containers:
  - name: app
    image: myapp:1.0
```

**Flow:**
```
Init Container 1 → Init Container 2 → Main Containers start
  (must succeed)      (must succeed)      (run in parallel)
```

### Pod Health Checks

**Three types of probes:**

1. **Liveness Probe** - Is the container running?
   - Fails → Container restarts
   
2. **Readiness Probe** - Is the container ready to serve traffic?
   - Fails → Removed from Service endpoints

3. **Startup Probe** - Has the container started?
   - For slow-starting containers

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-check-demo
spec:
  containers:
  - name: app
    image: myapp:1.0
    
    # Liveness Probe
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15  # Wait 15s before first check
      periodSeconds: 10         # Check every 10s
      timeoutSeconds: 1         # Timeout after 1s
      failureThreshold: 3       # Restart after 3 failures
    
    # Readiness Probe
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    
    # Startup Probe (for slow startup)
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

**Probe Types:**
```yaml
# HTTP GET
httpGet:
  path: /healthz
  port: 8080

# TCP Socket
tcpSocket:
  port: 3306

# Command Execution
exec:
  command:
  - cat
  - /tmp/healthy
```

### Why Not Use Pods Directly?

**Problem:** Pods are ephemeral (temporary)

```
Pod crashes → Pod dies → Manual recreation needed ❌
Node fails → All pods on that node lost ❌
Need scaling → Manually create more pods ❌
```

**Solution:** Use higher-level controllers (ReplicaSet, Deployment)

---

## 3. ReplicaSets - High Availability

### What is a ReplicaSet?

**ReplicaSet** ensures a specified number of pod replicas are running at all times.

**Key Features:**
- Maintains desired number of pods
- Self-healing (recreates failed pods)
- Horizontal scaling
- Uses label selectors to identify pods

### Analogy

Think of ReplicaSet as a **security guard** counting people:
- You tell the guard: "Keep exactly 3 people in the room"
- Someone leaves → Guard brings someone in
- Someone extra enters → Guard asks them to leave
- Always maintains the exact count

### ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3                    # Number of pods to maintain
  selector:                      # How to find pods to manage
    matchLabels:
      app: nginx
  template:                      # Pod template
    metadata:
      labels:
        app: nginx              # Must match selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**Structure:**
```
ReplicaSet
├── replicas: 3
├── selector: app=nginx
└── template:
    └── Pod specification
```

### How ReplicaSet Works

```
┌────────────────────────────────────────┐
│         ReplicaSet Controller          │
│    "I need 3 pods with app=nginx"     │
└──────────────┬─────────────────────────┘
               │
               ↓
    ┌──────────────────────┐
    │  Current State: 2    │
    │  Desired State: 3    │
    │  Action: Create 1    │
    └──────────────────────┘
               │
               ↓
┌──────────────┴──────────────────┐
│  Pod1   │   Pod2   │   Pod3     │
│(running)│ (running)│ (creating) │
└─────────────────────────────────┘
```

### ReplicaSet Commands

```bash
# Create ReplicaSet
kubectl apply -f replicaset.yaml

# Get ReplicaSets
kubectl get rs
kubectl get replicaset

# Output:
# NAME               DESIRED   CURRENT   READY   AGE
# nginx-replicaset   3         3         3       2m

# Describe ReplicaSet
kubectl describe rs nginx-replicaset

# Scale ReplicaSet
kubectl scale rs nginx-replicaset --replicas=5

# Delete ReplicaSet (also deletes pods)
kubectl delete rs nginx-replicaset

# Delete ReplicaSet but keep pods
kubectl delete rs nginx-replicaset --cascade=orphan
```

### Self-Healing Demo

```bash
# Create ReplicaSet with 3 replicas
kubectl apply -f replicaset.yaml

# Watch pods
kubectl get pods -w

# In another terminal, delete a pod
kubectl delete pod <pod-name>

# Watch the output - a new pod is immediately created!
# ReplicaSet ensures 3 pods always running
```

### Scaling Example

```bash
# Initial: 3 replicas
kubectl get rs

# Scale up to 5
kubectl scale rs nginx-replicaset --replicas=5

# Check pods
kubectl get pods
# Now you have 5 pods

# Scale down to 2
kubectl scale rs nginx-replicaset --replicas=2

# Check again
kubectl get pods
# 3 pods are terminating, 2 remain
```

### Why Not Use ReplicaSet Directly?

**Problem:** No built-in update/rollback mechanism

```
Update image → Delete old pods → Create new pods
                    ↓
              Downtime! ❌
              
Rollback? → Manual process ❌
```

**Solution:** Use Deployments (which manage ReplicaSets)

---

## 4. Deployments - Production Ready

### What is a Deployment?

**Deployment** is a higher-level controller that manages ReplicaSets and provides:
- Declarative updates
- Rolling updates (zero-downtime)
- Rollback capability
- Version history

**Hierarchy:**
```
Deployment
    ↓ (creates and manages)
ReplicaSet
    ↓ (creates and manages)
Pods
```

### Why Use Deployments?

**Scenario: Update Application Version**

**With ReplicaSet:**
```
1. Delete old ReplicaSet
2. Create new ReplicaSet with new image
   ↓
Downtime during recreation ❌
```

**With Deployment:**
```
1. Update Deployment with new image
2. Deployment creates new ReplicaSet
3. Gradually scales new ReplicaSet up
4. Gradually scales old ReplicaSet down
   ↓
Zero downtime! ✓
Automatic rollback if issues! ✓
```

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

**Create Deployment:**
```bash
# From YAML
kubectl apply -f deployment.yaml

# Imperative command
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# Verify
kubectl get deployments
kubectl get rs
kubectl get pods
```

### Deployment Strategies

#### 1. Rolling Update (Default)

Gradually replaces old pods with new pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # Max 2 extra pods during update
      maxUnavailable: 1   # Max 1 pod can be unavailable
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0
```

**Process:**
```
Initial State: 10 pods (v1.0)

Step 1: Create 2 new pods (v2.0) [maxSurge=2]
├─ Old: 10 pods (v1.0)
└─ New: 2 pods (v2.0)
Total: 12 pods

Step 2: Terminate 1 old pod [maxUnavailable=1]
├─ Old: 9 pods (v1.0)
└─ New: 2 pods (v2.0)
Total: 11 pods

Step 3: Repeat until all pods are v2.0
```

#### 2. Recreate Strategy

Delete all old pods before creating new ones.

```yaml
spec:
  strategy:
    type: Recreate
```

**Process:**
```
1. Delete all old pods
2. Wait for termination
3. Create new pods
   ↓
Downtime during transition! ⚠️
```

**When to use:**
- Application can't handle multiple versions running
- Database migrations needed
- Downtime is acceptable

### Updating Deployments

**Method 1: Update YAML and Apply**
```bash
# Edit deployment.yaml (change image version)
# image: nginx:1.21 → image: nginx:1.22

kubectl apply -f deployment.yaml
```

**Method 2: kubectl set image**
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
```

**Method 3: kubectl edit**
```bash
kubectl edit deployment nginx-deployment
# Opens editor, make changes, save
```

### Monitoring Rollout

```bash
# Trigger update
kubectl set image deployment/nginx nginx=nginx:1.22

# Watch rollout status
kubectl rollout status deployment/nginx

# Output:
# Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas updated...
# Waiting for deployment "nginx" rollout to finish: 2 of 3 updated replicas available...
# deployment "nginx" successfully rolled out

# View rollout history
kubectl rollout history deployment/nginx

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/nginx nginx=nginx:1.22
```

### Rollback

```bash
# View rollout history
kubectl rollout history deployment/nginx

# Rollback to previous version
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=1

# Check status
kubectl rollout status deployment/nginx
```

### Pause and Resume Rollouts

**Use case:** Make multiple changes before rolling out

```bash
# Pause rollout
kubectl rollout pause deployment/nginx

# Make multiple changes
kubectl set image deployment/nginx nginx=nginx:1.23
kubectl set resources deployment/nginx -c=nginx --limits=cpu=200m,memory=512Mi

# Resume rollout (applies all changes at once)
kubectl rollout resume deployment/nginx
```

### Scaling Deployments

```bash
# Scale to 5 replicas
kubectl scale deployment nginx --replicas=5

# Auto-scale based on CPU (HPA - Horizontal Pod Autoscaler)
kubectl autoscale deployment nginx --min=3 --max=10 --cpu-percent=80

# View HPA
kubectl get hpa
```

### Complete Deployment Example

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 10      # Keep 10 old ReplicaSets
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0           # Zero downtime
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1.0
    spec:
      containers:
      - name: webapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Resources
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Environment variables
        env:
        - name: ENV
          value: "production"
        - name: PORT
          value: "8080"
```

**Deploy:**
```bash
# Create deployment
kubectl apply -f deployment.yaml

# Verify
kubectl get deployments
kubectl get rs
kubectl get pods

# Update to v1.1
kubectl set image deployment/web-app webapp=myapp:1.1

# Monitor rollout
kubectl rollout status deployment/web-app

# If issues, rollback
kubectl rollout undo deployment/web-app
```

### Deployment Commands Cheat Sheet

```bash
# Create
kubectl create deployment <name> --image=<image>
kubectl apply -f deployment.yaml

# View
kubectl get deployments
kubectl describe deployment <name>

# Scale
kubectl scale deployment <name> --replicas=5

# Update
kubectl set image deployment/<name> <container>=<new-image>
kubectl edit deployment <name>
kubectl apply -f deployment.yaml  # After editing YAML

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=<number>

# Delete
kubectl delete deployment <name>
```

---

## 5. Services - Networking & Discovery

### What is a Service?

**Problem:** Pods have dynamic IPs
```
Pod created → Gets IP: 10.1.2.3
Pod dies → New pod → Gets new IP: 10.1.2.4
How do other pods find it? ❌
```

**Solution:** Service provides stable endpoint

**Service** is an abstract way to expose pods as a network service.

**Key Features:**
- Stable IP address (ClusterIP)
- DNS name
- Load balancing across pods
- Service discovery

### Analogy

Think of a Service as a **company receptionist**:
- Company has many employees (pods) who come and go
- Receptionist (service) has a fixed phone number
- Customers call the receptionist
- Receptionist forwards calls to available employees
- Employees change, but phone number stays same

### Service Types

1. **ClusterIP** (Default) - Internal cluster access only
2. **NodePort** - Exposes on each node's IP at a static port
3. **LoadBalancer** - External load balancer (cloud provider)
4. **ExternalName** - Maps to DNS name

### 1. ClusterIP Service

**Use case:** Internal communication between services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP              # Default, can be omitted
  selector:
    app: backend               # Target pods with this label
  ports:
  - protocol: TCP
    port: 80                   # Service port
    targetPort: 8080           # Container port
```

**How it works:**
```
┌────────────────────────────────────┐
│          Service                    │
│  Name: backend-service             │
│  ClusterIP: 10.96.0.10             │
│  Port: 80                          │
└──────────────┬─────────────────────┘
               │ (Load balances to)
      ┌────────┼────────┐
      │        │        │
  ┌───▼───┐ ┌─▼────┐ ┌─▼────┐
  │ Pod 1 │ │ Pod 2│ │ Pod 3│
  │ :8080 │ │ :8080│ │ :8080│
  └───────┘ └──────┘ └──────┘
```

**Access:**
```bash
# From inside cluster:
curl http://backend-service:80
curl http://backend-service.default.svc.cluster.local:80

# DNS format: <service-name>.<namespace>.svc.cluster.local
```

### 2. NodePort Service

**Use case:** Access service from outside cluster using node IP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80            # Service port (cluster internal)
    targetPort: 8080    # Container port
    nodePort: 30080     # Port on each node (30000-32767)
```

**How it works:**
```
External Client
      │
      ↓
Node IP: 192.168.1.100:30080
      │
      ↓
Service: web-service:80
      │
      ↓ (Load balances)
  ┌───┼───┐
Pod1  Pod2  Pod3
:8080 :8080 :8080
```

**Access:**
```bash
# From outside cluster:
curl http://<node-ip>:30080

# From inside cluster:
curl http://web-service:80
```

### 3. LoadBalancer Service

**Use case:** Cloud environments (AWS, GCP, Azure)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

**How it works:**
```
Internet
    │
    ↓
Cloud Load Balancer (External IP)
    │
    ↓
Service: public-web
    │
    ↓ (Distributes to)
Pods across multiple nodes
```

**Access:**
```bash
# Get external IP
kubectl get service public-web

# Output:
# NAME         TYPE           EXTERNAL-IP      PORT(S)
# public-web   LoadBalancer   35.123.45.67     80:31234/TCP

# Access from anywhere:
curl http://35.123.45.67
```

### 4. ExternalName Service

**Use case:** Access external service with cluster DNS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.example.com
```

**Access:**
```bash
# Instead of:
mysql -h mydb.example.com

# Use:
mysql -h external-db
```

### Service Discovery

**DNS-based (Automatic):**

Every service gets a DNS name:
```
<service-name>.<namespace>.svc.cluster.local
```

**Example:**
```yaml
# Service in 'default' namespace
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: default
spec:
  selector:
    app: backend
  ports:
  - port: 80
```

**DNS Names:**
```bash
# Same namespace:
curl http://backend

# Different namespace:
curl http://backend.default.svc.cluster.local

# Short form (different namespace):
curl http://backend.default
```

### Service + Deployment Example

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx              # Must match deployment labels
  ports:
  - protocol: TCP
    port: 80                # Service port
    targetPort: 80          # Container port
```

**Deploy:**
```bash
# Apply both
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Or combine in one file (separate with ---)
kubectl apply -f app.yaml

# Verify
kubectl get deployments
kubectl get services
kubectl get endpoints nginx-service  # Shows pod IPs
```

### Endpoints

**Endpoints** = List of pod IPs that match service selector

```bash
# View endpoints
kubectl get endpoints nginx-service

# Output:
# NAME            ENDPOINTS
# nginx-service   10.1.2.3:80,10.1.2.4:80,10.1.2.5:80

# Describe service to see endpoints
kubectl describe service nginx-service
```

### Service Commands

```bash
# Create service (expose deployment)
kubectl expose deployment nginx --port=80 --type=NodePort

# Get services
kubectl get services
kubectl get svc

# Describe service
kubectl describe service nginx

# View endpoints
kubectl get endpoints
kubectl get ep

# Edit service
kubectl edit service nginx

# Delete service
kubectl delete service nginx
```

### Session Affinity

**Default:** Requests distributed randomly

**Session Affinity:** Route requests from same client to same pod

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
  sessionAffinity: ClientIP  # Sticky sessions based on client IP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

---

## 6. Labels & Selectors

### What are Labels?

**Labels** are key-value pairs attached to objects (pods, services, deployments).

**Purpose:**
- Organize and select groups of objects
- Enable loose coupling
- Filter and query resources

**Examples:**
```yaml
labels:
  app: frontend
  environment: production
  tier: web
  version: v1.2
  team: platform
```

### Label Syntax

**Rules:**
- Key: up to 63 characters
- Value: up to 63 characters
- Format: `key: value` or `prefix/key: value`
- Alphanumeric, `-`, `_`, `.`

**Examples:**
```yaml
# Simple labels
app: nginx
env: prod

# With prefix
kubernetes.io/name: nginx
app.kubernetes.io/component: frontend
```

### Adding Labels

**In YAML:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: web
    environment: production
    tier: frontend
```

**Imperative:**
```bash
# Add label to existing pod
kubectl label pod my-pod environment=production

# Add label to multiple pods
kubectl label pods --all tier=frontend

# Update existing label (overwrite)
kubectl label pod my-pod environment=staging --overwrite

# Remove label
kubectl label pod my-pod environment-
```

### Selectors

**Selectors** use labels to filter and select objects.

#### Equality-based Selector

```bash
# Get pods with specific label
kubectl get pods -l app=nginx
kubectl get pods -l environment=production

# Multiple conditions (AND)
kubectl get pods -l app=nginx,environment=production

# Not equal
kubectl get pods -l environment!=development
```

#### Set-based Selector

```bash
# IN
kubectl get pods -l 'environment in (production, staging)'

# NOT IN
kubectl get pods -l 'environment notin (development, testing)'

# EXISTS
kubectl get pods -l app

# NOT EXISTS
kubectl get pods -l '!app'
```

### Selectors in Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend              # Targets pods with app=backend label
    tier: api
  ports:
  - port: 80
```

**How it works:**
```
Service selector: app=backend, tier=api
    ↓
Matches pods with BOTH labels
    ↓
Service endpoints: [Pod1, Pod2, Pod3]
```

### Selectors in Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:              # Simple equality
      app: web
      tier: frontend
  template:
    metadata:
      labels:                 # Pod labels (must match selector)
        app: web
        tier: frontend
        version: v1.0
```

**Advanced selector:**
```yaml
spec:
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - web
      - api
    - key: environment
      operator: NotIn
      values:
      - development
```

### Common Label Patterns

**Application labels:**
```yaml
labels:
  app.kubernetes.io/name: mysql
  app.kubernetes.io/instance: mysql-prod
  app.kubernetes.io/version: "5.7.21"
  app.kubernetes.io/component: database
  app.kubernetes.io/part-of: wordpress
  app.kubernetes.io/managed-by: helm
```

**Environment labels:**
```yaml
labels:
  environment: production
  tier: backend
  release: stable
```

**Team/ownership labels:**
```yaml
labels:
  team: platform
  owner: devops
  cost-center: engineering
```

### Annotations vs Labels

| Aspect | Labels | Annotations |
|--------|--------|-------------|
| **Purpose** | Identify and select objects | Store arbitrary metadata |
| **Used by** | Kubernetes (selectors) | Humans and tools |
| **Size limit** | 63 characters | 256 KB |
| **Queryable** | Yes (`-l` selector) | No |
| **Example** | `app: nginx` | `description: "Nginx web server"` |

**Annotations example:**
```yaml
metadata:
  annotations:
    description: "Production web server"
    contact: "devops@company.com"
    version: "1.2.3"
    build-date: "2026-09-02"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

---

## 7. Namespaces - Isolation

### What is a Namespace?

**Namespace** = Virtual cluster within a physical cluster

**Purpose:**
- Organize resources
- Isolate environments (dev, staging, prod)
- Multi-tenancy (different teams/projects)
- Resource quotas per namespace

### Analogy

Think of namespaces as **departments in a company**:
- Sales department, Engineering department, HR department
- Each has its own resources and people
- Some company resources are shared (cafeteria, parking)
- Each department has its own budget (resource quota)

### Default Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns
```

**Output:**
```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
kube-node-lease   Active   10d
```

**Namespaces:**

1. **default** - Default namespace for resources without specified namespace

2. **kube-system** - Kubernetes system components (DNS, dashboard, etc.)

3. **kube-public** - Publicly accessible (even without authentication)

4. **kube-node-lease** - Node heartbeat/lease objects

### Creating Namespaces

**Method 1: Imperative**
```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

**Method 2: Declarative**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
```

```bash
kubectl apply -f namespace.yaml
```

### Working with Namespaces

```bash
# Create resource in specific namespace
kubectl run nginx --image=nginx -n development

# Get resources in namespace
kubectl get pods -n development
kubectl get all -n development

# Get resources in all namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# Describe namespace
kubectl describe namespace development

# Delete namespace (deletes all resources in it!)
kubectl delete namespace development
```

### Set Default Namespace

```bash
# View current context
kubectl config current-context

# Set default namespace for current context
kubectl config set-context --current --namespace=development

# Verify
kubectl config view --minify | grep namespace:

# Now all commands use 'development' namespace by default
kubectl get pods  # Same as: kubectl get pods -n development
```

### Resources in Namespace

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production      # Specify namespace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: myapp:1.0
```

### Cross-Namespace Communication

**Service DNS:**
```
<service-name>.<namespace>.svc.cluster.local
```

**Example:**
```yaml
# Service in 'backend' namespace
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: backend
spec:
  selector:
    app: api
  ports:
  - port: 8080
```

**Access from different namespace:**
```bash
# From 'frontend' namespace pod:
curl http://api-service.backend.svc.cluster.local:8080

# Short form:
curl http://api-service.backend:8080
```

### Resource Quotas

**Limit resources per namespace:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"           # Total CPU requests
    requests.memory: 20Gi        # Total memory requests
    limits.cpu: "20"             # Total CPU limits
    limits.memory: 40Gi          # Total memory limits
    pods: "50"                   # Max number of pods
    services: "10"               # Max number of services
    persistentvolumeclaims: "5"  # Max PVCs
```

```bash
# Apply quota
kubectl apply -f quota.yaml

# View quotas
kubectl get resourcequota -n development
kubectl describe resourcequota dev-quota -n development
```

### LimitRange

**Set default and maximum resource limits:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: development
spec:
  limits:
  - default:           # Default limits
      memory: 512Mi
      cpu: 500m
    defaultRequest:    # Default requests
      memory: 256Mi
      cpu: 250m
    max:               # Maximum allowed
      memory: 1Gi
      cpu: 1
    min:               # Minimum required
      memory: 128Mi
      cpu: 100m
    type: Container
```

### Namespace Best Practices

**1. Use namespaces for environments:**
```
- development
- staging
- production
```

**2. Use namespaces for teams/projects:**
```
- team-frontend
- team-backend
- team-data
```

**3. Use namespaces for isolation:**
```
- customer-a
- customer-b (multi-tenancy)
```

**4. Don't use namespaces:**
- For resources that must be cluster-wide (nodes, namespaces, PersistentVolumes)
- When single environment is sufficient
- For very small projects

### Namespace-scoped vs Cluster-scoped Resources

**Namespace-scoped:**
```bash
# Resources that belong to a namespace
kubectl api-resources --namespaced=true

# Examples:
- pods
- services
- deployments
- configmaps
- secrets
```

**Cluster-scoped:**
```bash
# Resources that are cluster-wide
kubectl api-resources --namespaced=false

# Examples:
- nodes
- namespaces
- persistentvolumes
- storageclasses
- clusterroles
```

---

## Summary - Part 2

**What You Learned:**

1. **Kubernetes Objects**
   - YAML structure (apiVersion, kind, metadata, spec)
   - Declarative vs Imperative
   - Working with YAML files

2. **Pods**
   - Smallest unit in Kubernetes
   - Single and multi-container pods
   - Pod lifecycle and status
   - Health checks (liveness, readiness, startup probes)
   - Init containers
   - Resource limits

3. **ReplicaSets**
   - Ensures desired number of pods
   - Self-healing
   - Scaling

4. **Deployments**
   - Manages ReplicaSets
   - Rolling updates
   - Rollback capability
   - Deployment strategies
   - Production-ready

5. **Services**
   - Stable network endpoint
   - Service types (ClusterIP, NodePort, LoadBalancer, ExternalName)
   - Service discovery
   - Load balancing

6. **Labels & Selectors**
   - Organizing resources
   - Selecting resources
   - Service and deployment selectors

7. **Namespaces**
   - Virtual clusters
   - Resource isolation
   - Resource quotas
   - Cross-namespace communication

**Next:**
- **Part 3**: ConfigMaps, Secrets, Volumes, Persistent Storage
- **Part 4**: Networking, Ingress, Network Policies
- **Part 5**: Security, RBAC, Best Practices, Interview Prep

---

**Continue to Part 3!** 🚀
