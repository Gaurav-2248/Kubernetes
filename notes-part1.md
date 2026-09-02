# Kubernetes Zero to Hero - Part 1: Fundamentals

## Table of Contents
1. [Introduction to Containerization](#introduction-to-containerization)
2. [What is Kubernetes?](#what-is-kubernetes)
3. [Why Kubernetes?](#why-kubernetes)
4. [Kubernetes Architecture](#kubernetes-architecture)
5. [Setting Up Your First Cluster](#setting-up-your-first-cluster)

---

## 1. Introduction to Containerization

### What is a Container?

Before understanding Kubernetes, you need to understand containers.

**Traditional Application Deployment:**
```
[Application Code] → [OS Dependencies] → [Server Hardware]
```

**Problems with Traditional Approach:**
- "It works on my machine" syndrome
- Dependency conflicts
- Resource wastage
- Slow deployment
- Environment inconsistencies

**Container Approach:**
```
[Application + Dependencies + Runtime] → [Container Engine] → [OS] → [Hardware]
```

**What is a Container?**
- A lightweight, standalone package that includes everything needed to run an application
- Includes: code, runtime, system tools, libraries, settings
- Isolated from other containers
- Shares the host OS kernel (unlike VMs)

### Containers vs Virtual Machines

```
VIRTUAL MACHINES                    CONTAINERS
┌─────────────────────┐            ┌─────────────────────┐
│   App A   │  App B  │            │ App A │ App B │App C│
│ Bins/Libs │Bins/Libs│            │B/L│B/L│B/L          │
├───────────┴─────────┤            ├─────────────────────┤
│    Guest OS 1       │            │  Container Engine   │
├─────────────────────┤            │    (Docker/etc)     │
│   App C   │  App D  │            ├─────────────────────┤
│ Bins/Libs │Bins/Libs│            │     Host OS         │
├───────────┴─────────┤            ├─────────────────────┤
│    Guest OS 2       │            │   Infrastructure    │
├─────────────────────┤            └─────────────────────┘
│    Hypervisor       │
├─────────────────────┤            BENEFITS:
│     Host OS         │            ✓ Lightweight (MBs)
├─────────────────────┤            ✓ Fast startup (seconds)
│   Infrastructure    │            ✓ Efficient resource use
└─────────────────────┘            ✓ Portable
                                   ✓ Consistent environments
VMs are Heavy:
- Each VM needs full OS (GBs)
- Slow boot (minutes)
- More resource overhead
```

### Docker Basics (Quick Overview)

**Docker** is the most popular container platform.

**Key Docker Concepts:**
```bash
# Docker Image - Blueprint/template for containers
docker pull nginx:latest

# Docker Container - Running instance of an image
docker run -d -p 80:80 nginx

# Dockerfile - Instructions to build an image
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y python3
COPY app.py /app/
CMD ["python3", "/app/app.py"]
```

**Docker Lifecycle:**
```
Write Dockerfile → Build Image → Push to Registry → Pull Image → Run Container
```

### The Problem: Managing Containers at Scale

**Scenario:** You have a web application

**Development:**
- 1 container for frontend
- 1 container for backend
- 1 container for database
✓ Easy to manage manually

**Production:**
- 10 frontend containers (for load balancing)
- 5 backend containers
- 3 database containers
- Spread across multiple servers
- Need automatic restart on failure
- Need load balancing
- Need scaling up/down based on traffic
- Need zero-downtime deployments
- Need health monitoring

**Manual Management = NIGHTMARE** ❌

**This is where Kubernetes comes in!** ✓

---

## 2. What is Kubernetes?

### Definition

**Kubernetes (K8s)** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

**Etymology:**
- Created by Google (based on their internal Borg system)
- Open-sourced in 2014
- Name comes from Greek: "κυβερνήτης" (helmsman/pilot)
- K8s = K + 8 letters + s

### What Kubernetes Does

Think of Kubernetes as an **intelligent orchestrator** for your containers:

```
YOU SAY:                          KUBERNETES DOES:
"Run 3 copies of my app"    →     Creates 3 containers across servers
"Keep them running"         →     Monitors health, restarts failures
"Balance traffic"           →     Distributes requests evenly
"Update to v2.0"           →     Rolling update without downtime
"Scale to 10 copies"       →     Adds 7 more containers automatically
"Use this much CPU/RAM"    →     Allocates resources efficiently
```

### Kubernetes as a Cluster Manager

```
                    ┌────────────────────────┐
                    │    KUBERNETES          │
                    │    (Control Plane)     │
                    └───────────┬────────────┘
                                │
                    ┌───────────┴────────────┐
                    │                        │
            ┌───────▼──────┐        ┌───────▼──────┐
            │   SERVER 1   │        │   SERVER 2   │
            │  (Worker)    │        │  (Worker)    │
            │              │        │              │
            │ [C1][C2][C3] │        │ [C4][C5][C6] │
            └──────────────┘        └──────────────┘
            
C1-C6 = Containers managed by Kubernetes
```

---

## 3. Why Kubernetes?

### Problems Kubernetes Solves

#### Problem 1: Manual Deployment is Tedious
**Without K8s:**
```bash
# SSH into each server
ssh server1
docker run myapp
ssh server2
docker run myapp
# Repeat for 100 servers... 😫
```

**With K8s:**
```bash
kubectl create deployment myapp --image=myapp --replicas=100
# Done! All 100 containers deployed ✓
```

#### Problem 2: No Automatic Recovery
**Without K8s:**
- Container crashes → stays down
- Server fails → all containers on it are lost
- Manual intervention required

**With K8s:**
- Container crashes → automatically restarted
- Server fails → containers rescheduled to healthy servers
- Self-healing system

#### Problem 3: Scaling is Manual
**Without K8s:**
```
Traffic spike → Server overload → Manual SSH → Start more containers → 
Configure load balancer → Hope you did it fast enough
```

**With K8s:**
```
Traffic spike → K8s detects high CPU → Automatically scales containers → 
Rebalances load → When traffic drops, scales down
```

#### Problem 4: Updates Cause Downtime
**Without K8s:**
```
Stop all containers → Update → Start containers → Downtime = Lost revenue
```

**With K8s:**
```
Rolling update: Update 1 container → Verify → Update next → Zero downtime
```

#### Problem 5: Resource Waste
**Without K8s:**
- Server 1: 20% CPU usage
- Server 2: 80% CPU usage
- Inefficient resource distribution

**With K8s:**
- Intelligently schedules containers across servers
- Optimizes resource utilization
- Reduces infrastructure costs

### Key Features of Kubernetes

| Feature | Description | Benefit |
|---------|-------------|---------|
| **Auto-scaling** | Automatically scales containers based on CPU/memory/custom metrics | Handle traffic spikes, optimize costs |
| **Self-healing** | Restarts failed containers, replaces and reschedules containers | High availability without manual intervention |
| **Load Balancing** | Distributes network traffic across containers | Better performance, no single point of failure |
| **Automated Rollouts & Rollbacks** | Gradually deploys new versions, can rollback if issues detected | Zero-downtime deployments, safe updates |
| **Service Discovery** | Containers can find and communicate with each other automatically | Simplified networking |
| **Storage Orchestration** | Automatically mounts storage systems (local, cloud, network) | Persistent data for containers |
| **Secret & Configuration Management** | Manages sensitive information securely | Secure credential storage |
| **Batch Execution** | Run batch jobs and scheduled tasks | Automated background processing |

### When to Use Kubernetes

**Use Kubernetes When:**
- ✓ Running microservices architecture
- ✓ Need high availability (99.9%+ uptime)
- ✓ Application has variable load (need auto-scaling)
- ✓ Running multiple environments (dev, staging, prod)
- ✓ Team is comfortable with containerization
- ✓ Need zero-downtime deployments
- ✓ Managing 5+ services or containers

**Don't Use Kubernetes When:**
- ✗ Simple application (single server is enough)
- ✗ Team has no container experience
- ✗ Very small team with no DevOps resources
- ✗ Budget/resource constraints (K8s has learning curve)
- ✗ Application is a monolith with no plans to modernize

---

## 4. Kubernetes Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────┐
│                 KUBERNETES CLUSTER                      │
│                                                         │
│  ┌──────────────────────────────────────────────┐     │
│  │           CONTROL PLANE (Master)             │     │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────┐ │     │
│  │  │   API    │  │Scheduler │  │Controller │ │     │
│  │  │  Server  │  │          │  │  Manager  │ │     │
│  │  └──────────┘  └──────────┘  └───────────┘ │     │
│  │  ┌──────────────────────────────────────┐   │     │
│  │  │           etcd (Database)            │   │     │
│  │  └──────────────────────────────────────┘   │     │
│  └──────────────────┬───────────────────────────┘     │
│                     │ (manages)                        │
│  ┌──────────────────┴───────────────────────────┐     │
│  │              WORKER NODES                     │     │
│  │  ┌─────────────┐  ┌─────────────┐           │     │
│  │  │   Node 1    │  │   Node 2    │  ← More   │     │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │   Nodes   │     │
│  │  │ │Pod│Pod  │ │  │ │Pod│Pod  │ │           │     │
│  │  │ └─────────┘ │  │ └─────────┘ │           │     │
│  │  │  Kubelet    │  │  Kubelet    │           │     │
│  │  │  Kube-proxy │  │  Kube-proxy │           │     │
│  │  └─────────────┘  └─────────────┘           │     │
│  └───────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Components Explained (Simple Analogies)

Think of Kubernetes as a **Company**:

#### Control Plane = Management/Head Office

**1. API Server = Reception Desk**
- Entry point for all communications
- Receives all requests (kubectl commands, other components)
- Validates and processes requests
- Front-end to the cluster

**Example:**
```bash
kubectl get pods
# ↓
# Request goes to API Server first
# ↓
# API Server validates your credentials
# ↓
# Fetches data from etcd
# ↓
# Returns pod list to you
```

**2. etcd = Company Database/Filing System**
- Stores all cluster data
- Key-value store
- Source of truth for cluster state
- Highly consistent and distributed

**What's stored:**
- Node information
- Pod specifications
- ConfigMaps and Secrets
- Service configurations
- Current state vs desired state

**3. Scheduler = HR Department (Assigns Work)**
- Watches for new pods that have no node assigned
- Selects the best node for the pod to run on
- Considers: resource requirements, hardware constraints, affinity rules

**Scheduling Decision Process:**
```
New Pod Created → Scheduler Sees It → Checks:
  - Which nodes have enough CPU/RAM?
  - Any specific node requirements?
  - Is there affinity/anti-affinity?
  - Resource constraints?
→ Selects Best Node → Assigns Pod to Node
```

**4. Controller Manager = Operations Manager**
- Runs various controller processes
- Ensures desired state = actual state
- Continuously monitors cluster

**Controllers Include:**
- **Node Controller**: Monitors node health
- **Replication Controller**: Maintains correct number of pods
- **Endpoints Controller**: Populates endpoints (connects Services & Pods)
- **Service Account Controller**: Creates default accounts for namespaces

**Example:**
```
Desired State: 3 replicas of nginx
Actual State: 2 replicas (1 crashed)
↓
Replication Controller detects mismatch
↓
Creates 1 new pod
↓
State is now balanced
```

**5. Cloud Controller Manager** (Optional)
- Interacts with cloud provider APIs
- Manages cloud-specific resources (load balancers, storage, networking)

#### Worker Nodes = Employees/Workers

**1. Kubelet = Site Supervisor**
- Primary agent running on each node
- Ensures containers are running in pods
- Communicates with API server
- Reports node and pod status

**What Kubelet Does:**
```
1. Receives pod specifications from API server
2. Ensures containers in pod are running and healthy
3. Reports pod status back to API server
4. Manages pod lifecycle (start, stop, restart)
```

**2. Kube-proxy = Network Manager**
- Network proxy running on each node
- Maintains network rules
- Enables communication to pods
- Implements Kubernetes Service concept

**How it Works:**
```
Request to Service → Kube-proxy intercepts → 
Load balances to one of the pod replicas
```

**3. Container Runtime = The Worker Who Actually Does the Job**
- Software responsible for running containers
- Examples: Docker, containerd, CRI-O

**Workflow:**
```
Kubelet receives instruction → 
Tells Container Runtime to start container → 
Container Runtime pulls image and runs it
```

### Complete Request Flow Example

**Scenario:** You create a deployment with 3 replicas

```
1. YOU:
   kubectl create deployment nginx --image=nginx --replicas=3

2. kubectl → API SERVER:
   "Create deployment with 3 nginx pods"

3. API SERVER:
   - Authenticates and authorizes request
   - Validates the request
   - Writes deployment spec to etcd

4. DEPLOYMENT CONTROLLER (part of Controller Manager):
   - Sees new deployment in etcd
   - Creates a ReplicaSet with 3 pod specifications

5. REPLICASET CONTROLLER:
   - Sees ReplicaSet needs 3 pods
   - Creates 3 pod objects (no node assigned yet)
   - Writes to etcd

6. SCHEDULER:
   - Sees 3 unscheduled pods
   - Evaluates all worker nodes
   - Decides: Pod1→Node1, Pod2→Node2, Pod3→Node1
   - Updates pod specs with node assignment

7. KUBELET (on Node1):
   - Watches API server for pods assigned to its node
   - Sees 2 pods assigned to Node1
   - Tells Container Runtime to pull nginx image and start containers

8. KUBELET (on Node2):
   - Sees 1 pod assigned to Node2
   - Starts the container

9. KUBE-PROXY (on all nodes):
   - Updates network rules to route traffic to new pods

10. ALL KUBELETS:
    - Continuously report pod status to API server
    - API server updates etcd
    - Controller Manager monitors: Actual state (3 pods) = Desired state (3 pods) ✓
```

### Master Node vs Worker Node Comparison

| Aspect | Master Node (Control Plane) | Worker Node |
|--------|----------------------------|-------------|
| **Purpose** | Manages the cluster | Runs application workloads |
| **Components** | API Server, Scheduler, Controllers, etcd | Kubelet, Kube-proxy, Container Runtime |
| **Workloads** | Typically no user workloads | Runs user application pods |
| **Count** | 1 (dev) or 3+ (production) for HA | Many (1 to 1000s) |
| **Resource** | Less CPU/RAM needed | More resources for applications |
| **Critical** | Cluster stops working if all fail | Workloads affected, but cluster survives |

### High Availability Architecture

**Production Setup:**

```
                  ┌─── Load Balancer ───┐
                  │                     │
         ┌────────▼────────┐   ┌───────▼────────┐
         │  Master Node 1  │   │ Master Node 2  │
         │  - API Server   │   │ - API Server   │
         │  - Scheduler    │   │ - Scheduler    │
         │  - Controllers  │   │ - Controllers  │
         └────────┬────────┘   └────────┬───────┘
                  │                     │
         ┌────────┴─────────────────────┴────────┐
         │         etcd Cluster (3+ nodes)       │
         │         (Distributed Database)        │
         └────────┬─────────────────────┬────────┘
                  │                     │
    ┌─────────────┴──────┬──────────────┴──────────┐
    │                    │                          │
┌───▼────┐         ┌────▼─────┐            ┌──────▼───┐
│Worker 1│         │Worker 2  │    ...     │Worker N  │
└────────┘         └──────────┘            └──────────┘

Benefits:
- Master failure → Other masters take over
- etcd uses Raft consensus (majority vote)
- No single point of failure
```

---

## 5. Setting Up Your First Cluster

### Local Kubernetes Options

For learning and development, you can run Kubernetes locally:

#### Option 1: Minikube (Most Popular)

**What is it?**
- Single-node Kubernetes cluster
- Runs in a VM or container
- Perfect for learning

**Installation:**

**Windows:**
```bash
# Install via Chocolatey
choco install minikube

# Or download installer from:
# https://minikube.sigs.k8s.io/docs/start/
```

**macOS:**
```bash
brew install minikube
```

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Starting Minikube:**
```bash
# Start cluster
minikube start

# Check status
minikube status

# Open dashboard (web UI)
minikube dashboard

# Stop cluster
minikube stop

# Delete cluster
minikube delete
```

#### Option 2: Kind (Kubernetes in Docker)

**What is it?**
- Kubernetes cluster runs in Docker containers
- Very fast startup
- Used for testing Kubernetes itself

**Installation:**
```bash
# Windows (with Chocolatey)
choco install kind

# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Starting Kind:**
```bash
# Create cluster
kind create cluster --name my-cluster

# List clusters
kind get clusters

# Delete cluster
kind delete cluster --name my-cluster
```

#### Option 3: Docker Desktop

**Built-in Kubernetes:**
- Docker Desktop includes Kubernetes
- Enable in Settings → Kubernetes → Enable Kubernetes
- Easy for beginners

#### Option 4: K3s (Lightweight Kubernetes)

**What is it?**
- Lightweight Kubernetes by Rancher
- Perfect for edge/IoT devices
- Uses <512MB RAM

**Installation (Linux):**
```bash
curl -sfL https://get.k3s.io | sh -
```

### Installing kubectl (Kubernetes CLI)

kubectl is your command-line tool to interact with Kubernetes.

**Installation:**

**Windows:**
```bash
# Via Chocolatey
choco install kubernetes-cli

# Or download from:
# https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
```

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Verify Installation:**
```bash
kubectl version --client
```

### Your First Kubernetes Commands

```bash
# Start Minikube
minikube start

# Check cluster info
kubectl cluster-info

# View nodes in cluster
kubectl get nodes

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.27.0

# Get all resources
kubectl get all

# View cluster details
kubectl get nodes -o wide
```

### Understanding kubectl Context

**Context** = Which cluster you're talking to

```bash
# View available contexts
kubectl config get-contexts

# Current context
kubectl config current-context

# Switch context
kubectl config use-context minikube

# View full config
kubectl config view
```

### Creating Your First Pod

```bash
# Run a simple nginx container
kubectl run my-nginx --image=nginx

# Check if it's running
kubectl get pods

# Detailed pod info
kubectl describe pod my-nginx

# View logs
kubectl logs my-nginx

# Execute command in pod
kubectl exec -it my-nginx -- /bin/bash
# You're now inside the container!
# exit to get out

# Delete pod
kubectl delete pod my-nginx
```

### Quick Deployment Example

```bash
# Create a deployment (managed pods)
kubectl create deployment hello-k8s --image=nginx --replicas=3

# View deployment
kubectl get deployment

# View pods created by deployment
kubectl get pods

# Expose deployment as a service
kubectl expose deployment hello-k8s --port=80 --type=NodePort

# Get service details
kubectl get service hello-k8s

# Access the service (Minikube)
minikube service hello-k8s

# Scale deployment
kubectl scale deployment hello-k8s --replicas=5

# Watch pods being created
kubectl get pods -w

# Clean up
kubectl delete deployment hello-k8s
kubectl delete service hello-k8s
```

### Kubernetes Dashboard

Visual way to manage cluster:

```bash
# Minikube - open dashboard
minikube dashboard

# General Kubernetes (deploy dashboard)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Access dashboard
kubectl proxy
# Then open: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

---

## Summary - Part 1

**What You Learned:**

1. **Containers Basics**
   - What containers are
   - Containers vs VMs
   - Why containerization matters

2. **Kubernetes Introduction**
   - What Kubernetes is
   - Problems it solves
   - Key features

3. **Architecture**
   - Control Plane components (API Server, etcd, Scheduler, Controllers)
   - Worker Node components (Kubelet, Kube-proxy, Container Runtime)
   - How they work together

4. **Hands-on Setup**
   - Local Kubernetes options
   - Installing kubectl
   - First commands
   - Running your first pod

**Next Steps:**
- **Part 2** will cover: Pods, ReplicaSets, Deployments, Services in depth
- **Part 3** will cover: ConfigMaps, Secrets, Volumes, Persistent Storage
- **Part 4** will cover: Networking, Ingress, Network Policies
- **Part 5** will cover: Security, RBAC, Best Practices, Interview Prep

---

## Quick Reference

### Essential Commands for Beginners
```bash
# Cluster
kubectl cluster-info
kubectl get nodes

# Pods
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod> -- /bin/bash

# Deployments
kubectl get deployments
kubectl create deployment <name> --image=<image>
kubectl scale deployment <name> --replicas=<number>

# Services
kubectl get services
kubectl expose deployment <name> --port=<port>

# General
kubectl get all
kubectl delete <resource> <name>
```

### Common Kubectl Shortcuts
```bash
kubectl get po    # pods
kubectl get svc   # services
kubectl get deploy # deployments
kubectl get no    # nodes
kubectl get ns    # namespaces
```

---

**Continue to Part 2 for deep dive into Kubernetes Objects!** 🚀
