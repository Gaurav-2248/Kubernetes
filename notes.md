# Kubernetes (K8s) Interview Notes

## Table of Contents
1. [What is Kubernetes?](#what-is-kubernetes)
2. [Core Architecture](#core-architecture)
3. [Key Components](#key-components)
4. [Kubernetes Objects](#kubernetes-objects)
5. [Networking](#networking)
6. [Storage](#storage)
7. [Security](#security)
8. [Commands Cheat Sheet](#commands-cheat-sheet)
9. [Common Interview Questions](#common-interview-questions)

---

## What is Kubernetes?

**Definition**: Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

**Key Features**:
- Auto-scaling
- Self-healing
- Load balancing
- Rolling updates & rollbacks
- Service discovery
- Storage orchestration

**Why K8s?**
- Manages containers at scale
- Platform independent
- Cloud agnostic
- High availability & fault tolerance

---

## Core Architecture

### Master Node (Control Plane)
Manages the cluster and makes decisions about scheduling, scaling, etc.

**Components**:
1. **API Server** - Entry point for all REST commands; front-end for control plane
2. **etcd** - Key-value store for all cluster data (brain of K8s)
3. **Scheduler** - Assigns pods to nodes based on resource requirements
4. **Controller Manager** - Runs controller processes (node, replication, endpoints, service account)
5. **Cloud Controller Manager** - Interacts with cloud provider APIs

### Worker Node
Runs application workloads in containers.

**Components**:
1. **Kubelet** - Agent that ensures containers are running in pods
2. **Kube-proxy** - Network proxy for service networking
3. **Container Runtime** - Software to run containers (Docker, containerd, CRI-O)

```
┌─────────────────────────────────────┐
│         MASTER NODE                 │
│  ┌──────────┐  ┌──────────────┐   │
│  │API Server│  │  Scheduler   │   │
│  └──────────┘  └──────────────┘   │
│  ┌──────────┐  ┌──────────────┐   │
│  │   etcd   │  │ Controller   │   │
│  └──────────┘  └──────────────┘   │
└─────────────────────────────────────┘
           │
           │ (manages)
           ▼
┌─────────────────────────────────────┐
│         WORKER NODES                │
│  ┌─────────────────────────────┐   │
│  │ Pod │ Pod │ Pod │ Pod │ Pod │   │
│  └─────────────────────────────┘   │
│  ┌──────────┐  ┌──────────────┐   │
│  │ Kubelet  │  │  Kube-proxy  │   │
│  └──────────┘  └──────────────┘   │
└─────────────────────────────────────┘
```

---

## Key Components

### 1. Pod
- **Smallest deployable unit** in K8s
- Wraps one or more containers
- Shares network namespace and storage
- Ephemeral (can be destroyed and recreated)

**Types**:
- Single container pod (most common)
- Multi-container pod (sidecar pattern)

### 2. ReplicaSet
- Ensures specified number of pod replicas are running
- Self-healing mechanism
- Usually managed by Deployment

### 3. Deployment
- Manages ReplicaSets
- Provides declarative updates
- Supports rolling updates and rollbacks
- **Most commonly used** for stateless applications

### 4. Service
- Provides stable network endpoint for pods
- Load balances traffic across pods
- Types:
  - **ClusterIP** (default) - Internal cluster access only
  - **NodePort** - Exposes on each node's IP at a static port
  - **LoadBalancer** - External load balancer (cloud provider)
  - **ExternalName** - Maps to DNS name

### 5. ConfigMap
- Stores non-sensitive configuration data
- Key-value pairs
- Can be consumed as environment variables or files

### 6. Secret
- Stores sensitive data (passwords, tokens, keys)
- Base64 encoded
- Types: Opaque, TLS, Docker registry, etc.

### 7. Namespace
- Virtual clusters within a physical cluster
- Isolates resources
- Default namespaces: `default`, `kube-system`, `kube-public`

### 8. Ingress
- Manages external HTTP/HTTPS access to services
- Provides load balancing, SSL termination, name-based virtual hosting
- Requires Ingress Controller (nginx, traefik, etc.)

### 9. StatefulSet
- For stateful applications
- Provides stable network identity and persistent storage
- Ordered deployment and scaling
- Use cases: Databases, message queues

### 10. DaemonSet
- Ensures all (or some) nodes run a copy of a pod
- Use cases: Log collectors, monitoring agents, network proxies

### 11. Job & CronJob
- **Job**: Runs pods to completion (batch processing)
- **CronJob**: Scheduled jobs (like cron in Linux)

---

## Kubernetes Objects

### YAML Structure
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: my-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

**Key Fields**:
- `apiVersion` - K8s API version
- `kind` - Resource type
- `metadata` - Name, labels, annotations
- `spec` - Desired state specification

---

## Networking

### Pod Networking
- Each pod gets unique IP address
- Containers in same pod communicate via localhost
- Pods communicate without NAT

### Service Networking
- Services get ClusterIP (virtual IP)
- kube-proxy implements service networking
- Modes: iptables, IPVS, userspace

### Network Policies
- Firewall rules for pods
- Control ingress/egress traffic
- Requires CNI plugin support (Calico, Weave, Cilium)

### CNI (Container Network Interface)
Popular plugins:
- **Flannel** - Simple overlay network
- **Calico** - Network policies + routing
- **Weave** - Easy setup, encryption
- **Cilium** - eBPF-based, high performance

---

## Storage

### Volumes
Provides persistent storage to pods.

**Types**:
1. **emptyDir** - Temporary, deleted with pod
2. **hostPath** - Mounts from node filesystem
3. **PersistentVolume (PV)** - Cluster-level storage resource
4. **PersistentVolumeClaim (PVC)** - Request for storage by user

### Storage Classes
- Dynamic provisioning of PVs
- Different classes for different QoS levels
- Examples: SSD, HDD, NFS

**PV Access Modes**:
- `ReadWriteOnce (RWO)` - Single node read-write
- `ReadOnlyMany (ROX)` - Many nodes read-only
- `ReadWriteMany (RWX)` - Many nodes read-write

---

## Security

### 1. Authentication
- Service Accounts (for pods)
- User Accounts (for humans)
- Methods: Certificates, tokens, OIDC, webhook

### 2. Authorization
- **RBAC** (Role-Based Access Control) - Most common
- ABAC (Attribute-Based)
- Node authorization
- Webhook mode

**RBAC Components**:
- **Role** - Permissions within a namespace
- **ClusterRole** - Cluster-wide permissions
- **RoleBinding** - Grants Role to users/groups
- **ClusterRoleBinding** - Grants ClusterRole

### 3. Admission Controllers
- Intercept requests before persistence
- Examples: PodSecurityPolicy, ResourceQuota, LimitRanger

### 4. Pod Security
- **Security Context** - Define privilege and access control
- **Pod Security Standards**: Privileged, Baseline, Restricted
- **Network Policies** - Control pod-to-pod communication

### 5. Secrets Management
- Base64 encoded (not encrypted by default)
- Can integrate with external secret managers (Vault, AWS Secrets Manager)
- Encrypted at rest with KMS

---

## Commands Cheat Sheet

### Cluster Info
```bash
kubectl cluster-info                 # Cluster information
kubectl get nodes                    # List all nodes
kubectl describe node <node-name>    # Node details
kubectl top nodes                    # Node resource usage
```

### Pods
```bash
kubectl get pods                     # List pods in default namespace
kubectl get pods -A                  # List all pods in all namespaces
kubectl get pods -o wide             # Detailed pod info with node/IP
kubectl describe pod <pod-name>      # Detailed pod information
kubectl logs <pod-name>              # View pod logs
kubectl logs -f <pod-name>           # Stream logs
kubectl exec -it <pod-name> -- /bin/bash  # SSH into pod
kubectl delete pod <pod-name>        # Delete pod
```

### Deployments
```bash
kubectl create deployment <name> --image=<image>  # Create deployment
kubectl get deployments              # List deployments
kubectl describe deployment <name>   # Deployment details
kubectl scale deployment <name> --replicas=5      # Scale deployment
kubectl set image deployment/<name> <container>=<new-image>  # Update image
kubectl rollout status deployment/<name>          # Check rollout status
kubectl rollout undo deployment/<name>            # Rollback deployment
kubectl rollout history deployment/<name>         # View rollout history
kubectl delete deployment <name>     # Delete deployment
```

### Services
```bash
kubectl get services                 # List services
kubectl expose deployment <name> --port=80 --type=NodePort  # Expose deployment
kubectl describe service <name>      # Service details
kubectl delete service <name>        # Delete service
```

### ConfigMaps & Secrets
```bash
kubectl create configmap <name> --from-literal=key=value
kubectl get configmaps
kubectl describe configmap <name>

kubectl create secret generic <name> --from-literal=password=secret
kubectl get secrets
kubectl describe secret <name>
```

### Namespaces
```bash
kubectl get namespaces               # List namespaces
kubectl create namespace <name>      # Create namespace
kubectl get pods -n <namespace>      # Get pods in namespace
kubectl config set-context --current --namespace=<name>  # Switch namespace
```

### Apply & Delete Resources
```bash
kubectl apply -f <file.yaml>         # Create/update resources from file
kubectl apply -f <directory>         # Apply all YAML files in directory
kubectl delete -f <file.yaml>        # Delete resources from file
kubectl delete pod <name>            # Delete specific resource
kubectl delete all --all             # Delete all resources in namespace
```

### Debugging
```bash
kubectl describe <resource> <name>   # Detailed info
kubectl logs <pod-name>              # Container logs
kubectl logs <pod-name> -c <container>  # Specific container logs
kubectl exec -it <pod> -- /bin/bash  # Interactive shell
kubectl port-forward <pod> 8080:80   # Port forwarding
kubectl top pod <pod-name>           # Resource usage
kubectl get events                   # Cluster events
kubectl get all                      # Get all resources
```

### YAML Operations
```bash
kubectl get deployment <name> -o yaml  # Get resource in YAML
kubectl explain pods                 # Documentation for resource
kubectl explain pods.spec            # Spec documentation
kubectl dry-run=client -o yaml       # Generate YAML without creating
```

### Context & Config
```bash
kubectl config view                  # View kubeconfig
kubectl config get-contexts          # List contexts
kubectl config use-context <name>    # Switch context
kubectl config current-context       # Current context
```

---

## Common Interview Questions

### Basic Level

**Q1: What is Kubernetes?**
A: K8s is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

**Q2: What is a Pod?**
A: A Pod is the smallest deployable unit in K8s that can contain one or more containers sharing network and storage.

**Q3: Difference between Docker and Kubernetes?**
A: Docker is a containerization platform that packages applications, while Kubernetes orchestrates and manages these containers at scale.

**Q4: What is kubectl?**
A: kubectl is the command-line tool to interact with Kubernetes clusters.

**Q5: What is a Namespace?**
A: A virtual cluster within a physical cluster that provides isolation for resources.

### Intermediate Level

**Q6: Difference between ReplicaSet and Deployment?**
A: Deployment is higher-level abstraction that manages ReplicaSets and provides rolling updates, rollbacks, and declarative updates.

**Q7: What are the types of Services?**
A: ClusterIP (internal), NodePort (exposes on node), LoadBalancer (external LB), ExternalName (DNS mapping).

**Q8: What is the difference between ConfigMap and Secret?**
A: ConfigMap stores non-sensitive configuration, Secret stores sensitive data (base64 encoded).

**Q9: Explain Liveness vs Readiness probes?**
- **Liveness**: Checks if container is running; restarts if fails
- **Readiness**: Checks if container is ready to serve traffic; removes from service if fails

**Q10: What is an Ingress?**
A: Ingress manages external HTTP/HTTPS access to services, providing load balancing, SSL termination, and name-based virtual hosting.

### Advanced Level

**Q11: How does K8s achieve high availability?**
A: Multiple master nodes, etcd clustering, pod replicas, self-healing, rolling updates, and health checks.

**Q12: Explain K8s networking model?**
A: All pods can communicate without NAT, all nodes can communicate with pods without NAT, pod sees same IP as others see it.

**Q13: What is RBAC in Kubernetes?**
A: Role-Based Access Control manages authorization using Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings.

**Q14: How does Kubernetes handle secrets securely?**
A: Secrets are base64 encoded, can be encrypted at rest using KMS, mounted as volumes or env vars, and can integrate with external secret managers.

**Q15: Explain StatefulSet vs Deployment?**
- **Deployment**: Stateless apps, pods are interchangeable, no persistent identity
- **StatefulSet**: Stateful apps, stable network identity, ordered deployment, persistent storage

**Q16: What is DaemonSet used for?**
A: Ensures a pod runs on all (or specific) nodes. Used for logging agents, monitoring, network proxies.

**Q17: How do you troubleshoot a pod that won't start?**
```bash
kubectl describe pod <name>          # Check events
kubectl logs <name>                  # Check logs
kubectl get events                   # Cluster events
kubectl get pod <name> -o yaml       # Full pod config
```

**Q18: What is a Helm Chart?**
A: Package manager for Kubernetes that bundles K8s resources into reusable templates.

**Q19: Explain Blue-Green vs Canary deployment?**
- **Blue-Green**: Two identical environments; switch traffic completely
- **Canary**: Gradual rollout to small subset, then full deployment

**Q20: What is a Service Mesh? (Istio, Linkerd)**
A: Infrastructure layer for service-to-service communication providing traffic management, security, and observability.

---

## Real-World Scenarios

### Scenario 1: Pod CrashLoopBackOff
**Problem**: Pod keeps restarting
**Debugging**:
1. `kubectl describe pod <name>` - Check events
2. `kubectl logs <name> --previous` - Check previous container logs
3. Check resource limits, liveness/readiness probes
4. Verify image exists and is correct

### Scenario 2: Service Not Accessible
**Debugging**:
1. Check service endpoints: `kubectl get endpoints <service>`
2. Verify selector matches pod labels
3. Check service type and port configuration
4. Test from within cluster: `kubectl run test --rm -it --image=busybox -- wget -O- <service-ip>`

### Scenario 3: High Memory/CPU Usage
**Solution**:
1. Set resource requests and limits
2. Use HPA (Horizontal Pod Autoscaler)
3. Monitor with Prometheus/Grafana
4. Optimize application code

### Scenario 4: Storage Issues
**Debugging**:
1. Check PVC status: `kubectl get pvc`
2. Verify StorageClass exists
3. Check PV bound status
4. Review access modes compatibility

---

## Best Practices

1. **Resource Management**
   - Always set resource requests and limits
   - Use namespace quotas

2. **Security**
   - Use RBAC
   - Don't run containers as root
   - Use Pod Security Standards
   - Scan images for vulnerabilities

3. **Configuration**
   - Use ConfigMaps and Secrets
   - Don't hardcode values
   - Version your YAML files

4. **Monitoring & Logging**
   - Implement health checks
   - Centralized logging (ELK, Loki)
   - Monitoring (Prometheus, Grafana)

5. **Deployment**
   - Use rolling updates
   - Test in staging first
   - Have rollback plan
   - Use Helm for complex apps

6. **Naming & Labels**
   - Use consistent naming conventions
   - Label everything for filtering
   - Use annotations for metadata

---

## Quick Reference - YAML Examples

### Deployment
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
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgresql://localhost:5432"
  log_level: "info"
```

### Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=      # base64 encoded
  password: cGFzc3dvcmQ=  # base64 encoded
```

---

## Interview Tips

1. **Understand the basics thoroughly** - Pod, Service, Deployment
2. **Know common commands** - Practice kubectl commands
3. **Explain architecture clearly** - Master vs Worker nodes
4. **Real-world scenarios** - Be ready to troubleshoot
5. **Hands-on experience** - Use Minikube/Kind for practice
6. **Security awareness** - RBAC, secrets, network policies
7. **Stay updated** - K8s evolves rapidly

---

## Practice Resources

- **Local Setup**: Minikube, Kind, K3s
- **Cloud**: GKE (Google), EKS (AWS), AKS (Azure)
- **Learning**: Kubernetes.io documentation
- **Practice**: KodeKloud, A Cloud Guru
- **Certification**: CKA, CKAD, CKS

---

**Good luck with your interviews!** 🚀
