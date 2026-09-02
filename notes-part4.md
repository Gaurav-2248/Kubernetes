# Kubernetes Zero to Hero - Part 4: Networking & Ingress

## Table of Contents
1. [Kubernetes Networking Fundamentals](#kubernetes-networking-fundamentals)
2. [DNS in Kubernetes](#dns-in-kubernetes)
3. [Ingress Controllers](#ingress-controllers)
4. [Ingress Resources](#ingress-resources)
5. [Network Policies](#network-policies)
6. [Service Mesh Introduction](#service-mesh-introduction)

---

## 1. Kubernetes Networking Fundamentals

### The Kubernetes Network Model

Kubernetes networking follows these rules:

1. **Every Pod gets its own IP address**
   - No need for NAT between pods
   - Containers in a pod share the same IP

2. **All Pods can communicate with each other**
   - Without NAT
   - Across nodes

3. **All Nodes can communicate with all Pods**
   - Without NAT

4. **Pod sees same IP as others see it**
   - No port mapping confusion

### Network Architecture

```
┌─────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                 │
│                                                     │
│  ┌──────────────────┐      ┌──────────────────┐  │
│  │   Node 1         │      │   Node 2         │  │
│  │                  │      │                  │  │
│  │  ┌────────────┐  │      │  ┌────────────┐  │  │
│  │  │ Pod A      │  │      │  │ Pod C      │  │  │
│  │  │ 10.1.1.2   │  │      │  │ 10.1.2.2   │  │  │
│  │  └────────────┘  │      │  └────────────┘  │  │
│  │                  │      │                  │  │
│  │  ┌────────────┐  │      │  ┌────────────┐  │  │
│  │  │ Pod B      │  │      │  │ Pod D      │  │  │
│  │  │ 10.1.1.3   │  │      │  │ 10.1.2.3   │  │  │
│  │  └────────────┘  │      │  └────────────┘  │  │
│  │                  │      │                  │  │
│  └──────────────────┘      └──────────────────┘  │
│           │                         │             │
│           └─────────┬───────────────┘             │
│                     │                             │
│              ┌──────▼──────┐                      │
│              │   CNI       │                      │
│              │  (Network   │                      │
│              │   Plugin)   │                      │
│              └─────────────┘                      │
└─────────────────────────────────────────────────────┘

Pod A can directly reach Pod C at 10.1.2.2 (no NAT)
```

### CNI (Container Network Interface)

**CNI** plugins implement the Kubernetes network model.

**Popular CNI Plugins:**

| Plugin | Description | Features |
|--------|-------------|----------|
| **Calico** | High performance, network policies | BGP routing, Network policies, Encryption |
| **Flannel** | Simple overlay network | Easy setup, UDP/VXLAN |
| **Weave** | Mesh network | Automatic discovery, Encryption |
| **Cilium** | eBPF-based | High performance, Advanced policies, Observability |
| **Canal** | Calico + Flannel | Policy + Simple overlay |

**Installing CNI (Example: Calico):**
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Pod-to-Pod Communication

**Within Same Node:**
```
Pod A (10.1.1.2) → Virtual Bridge → Pod B (10.1.1.3)
```

**Across Nodes:**
```
Pod A (10.1.1.2) → Node 1 → CNI Plugin → Node 2 → Pod C (10.1.2.2)
```

### Service Communication

**ClusterIP Service:**
```
┌──────────────────────────────────────┐
│  Client Pod                          │
│  Calls: backend-service:8080         │
└────────────┬─────────────────────────┘
             │
             ↓
┌────────────────────────────────────────┐
│  kube-proxy (on each node)            │
│  - Maintains iptables/IPVS rules      │
│  - Load balances to backend pods      │
└────────────┬───────────────────────────┘
             │
      ┌──────┼──────┐
      ↓      ↓      ↓
   Pod1   Pod2   Pod3
  :8080  :8080  :8080
```

### kube-proxy Modes

**1. iptables Mode (default)**
- Uses iptables rules for load balancing
- Random selection
- Lower overhead

**2. IPVS Mode**
- Uses IPVS (IP Virtual Server)
- Better performance at scale
- Multiple load balancing algorithms
- Requires IPVS kernel modules

**Enable IPVS:**
```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: ipvs
    ipvs:
      scheduler: rr  # Round-robin
```

### Network Namespaces

Kubernetes uses Linux network namespaces for isolation.

**Each Pod gets:**
- Its own network namespace
- Its own network interface
- Its own routing table
- Its own iptables rules

**Containers in same Pod share:**
- Network namespace
- Can communicate via localhost
- See same IP address

### Troubleshooting Network Issues

**Debug Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ['sh', '-c', 'sleep 3600']
```

```bash
kubectl apply -f netshoot.yaml
kubectl exec -it netshoot -- bash

# Inside pod:
# Test DNS
nslookup kubernetes.default

# Test connectivity
curl http://backend-service:8080

# Check routing
ip route

# Check IP address
ip addr

# Ping another pod
ping 10.1.2.3

# Trace route
traceroute backend-service

# Check DNS resolution
dig backend-service.default.svc.cluster.local

# Network tools available:
# curl, wget, netcat, nmap, tcpdump, etc.
```

---

## 2. DNS in Kubernetes

### CoreDNS

**CoreDNS** is the default DNS server in Kubernetes.

**What it does:**
- Provides DNS service discovery
- Maps service names to IP addresses
- Enables service-to-service communication

**CoreDNS Deployment:**
```bash
# View CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml
```

### DNS Naming Convention

**Format:**
```
<service-name>.<namespace>.svc.<cluster-domain>
```

**Default cluster domain:** `cluster.local`

### Service DNS Records

**ClusterIP Service Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  selector:
    app: backend
  ports:
  - port: 8080
```

**DNS Names:**
```bash
# Full FQDN
backend.production.svc.cluster.local

# Short forms (from within same namespace)
backend                              # Same namespace
backend.production                   # Any namespace
backend.production.svc              # Full without domain
```

### Pod DNS Records

**Pod DNS Format:**
```
<pod-ip-with-dashes>.<namespace>.pod.<cluster-domain>
```

**Example:**
```
Pod IP: 10.1.2.3
Namespace: default

DNS: 10-1-2-3.default.pod.cluster.local
```

**Enable Pod DNS (optional):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None  # Headless service
  selector:
    app: nginx
  publishNotReadyAddresses: true
```

### StatefulSet DNS

**StatefulSet pods get stable DNS names:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx  # Headless service
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
        image: nginx
```

**Pod DNS Names:**
```
web-0.nginx.default.svc.cluster.local
web-1.nginx.default.svc.cluster.local
web-2.nginx.default.svc.cluster.local
```

### Custom DNS Configuration

**Pod-level DNS:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns
spec:
  containers:
  - name: app
    image: nginx
  dnsPolicy: "None"  # Use custom DNS
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - default.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:
    - name: ndots
      value: "2"
```

**DNS Policies:**

| Policy | Description |
|--------|-------------|
| **Default** | Inherit DNS from node |
| **ClusterFirst** | Use cluster DNS (default for pods) |
| **ClusterFirstWithHostNet** | For pods with hostNetwork=true |
| **None** | Use custom dnsConfig |

### DNS Testing

```bash
# Create test pod
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# Inside pod:
# Test service DNS
nslookup backend.default.svc.cluster.local

# Test short name
nslookup backend

# Check /etc/resolv.conf
cat /etc/resolv.conf
# Output:
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

### DNS Performance Tuning

**ndots parameter:**
```yaml
dnsConfig:
  options:
  - name: ndots
    value: "2"  # Default is 5
```

**How ndots works:**
```
ndots: 5 (default)

Query: backend
Tries:
1. backend.default.svc.cluster.local.
2. backend.svc.cluster.local.
3. backend.cluster.local.
4. backend. (absolute)
   ↓
5 DNS queries for short name!

ndots: 2 (optimized)
Tries:
1. backend.default.svc.cluster.local.
2. backend. (absolute)
   ↓
2 DNS queries
```

**Best Practice:** Use full DNS names in production to reduce queries.

```yaml
# Instead of:
http://backend

# Use:
http://backend.default.svc.cluster.local
```

---

## 3. Ingress Controllers

### What is Ingress?

**Problem: Exposing Multiple Services**

**Without Ingress:**
```
service1 → LoadBalancer 1 (External IP 1)
service2 → LoadBalancer 2 (External IP 2)
service3 → LoadBalancer 3 (External IP 3)
   ↓
Expensive! Multiple IPs! ❌
```

**With Ingress:**
```
                    Single LoadBalancer
                            ↓
                    Ingress Controller
                     /      |      \
              service1  service2  service3
   ↓
One IP, route by hostname/path! ✓
```

**Ingress provides:**
- HTTP/HTTPS routing
- Virtual hosting (multiple domains)
- Path-based routing
- SSL/TLS termination
- Load balancing

### Ingress Architecture

```
┌──────────────────────────────────────────┐
│          Internet/External               │
└────────────────┬─────────────────────────┘
                 │
                 ↓
         ┌──────────────┐
         │ LoadBalancer │
         └──────┬───────┘
                │
                ↓
    ┌───────────────────────┐
    │  Ingress Controller   │  ← Nginx, Traefik, etc.
    │  (reads Ingress rules)│
    └───────────┬───────────┘
                │
        ┌───────┼───────┐
        ↓       ↓       ↓
    Service1 Service2 Service3
        ↓       ↓       ↓
     Pods    Pods    Pods
```

### Popular Ingress Controllers

| Controller | Description | Features |
|------------|-------------|----------|
| **Nginx Ingress** | Most popular | Full-featured, battle-tested |
| **Traefik** | Modern, dynamic | Auto-discovery, Let's Encrypt |
| **HAProxy** | High performance | TCP/HTTP load balancing |
| **Contour** | Envoy-based | gRPC support, modern |
| **Kong** | API Gateway | Plugins, rate limiting |
| **AWS ALB** | AWS native | Integration with AWS services |
| **GCE** | Google Cloud native | Integration with GCP |

### Installing Ingress Controller

**Nginx Ingress (most common):**
```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Or using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Get external IP
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

---

## 4. Ingress Resources

### Basic Ingress

**services:**
```yaml
# Frontend service
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80

---
# Backend service
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 8080
```

**ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080
```

**Deploy:**
```bash
kubectl apply -f services.yaml
kubectl apply -f ingress.yaml

# Check Ingress
kubectl get ingress
kubectl describe ingress simple-ingress

# Access (add to /etc/hosts first)
# <EXTERNAL-IP> myapp.example.com
curl http://myapp.example.com/
curl http://myapp.example.com/api
```

### Path Types

**1. Exact**
```yaml
pathType: Exact
path: /api
# Matches: /api only
# Not: /api/, /api/users
```

**2. Prefix**
```yaml
pathType: Prefix
path: /api
# Matches: /api, /api/, /api/users, /api/v1/users
```

**3. ImplementationSpecific**
```yaml
pathType: ImplementationSpecific
# Depends on Ingress Controller
```

### Multiple Hosts

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
  # First host
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: website
            port:
              number: 80
  
  # Second host
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  
  # Third host
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-panel
            port:
              number: 80
```

**Access:**
```
http://www.example.com → website service
http://api.example.com → api-service
http://admin.example.com → admin-panel
```

### TLS/HTTPS Ingress

**Create TLS secret:**
```bash
# Generate self-signed cert (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com"

# Create secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

**ingress-tls.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

**Access:**
```bash
curl https://myapp.example.com
```

### Default Backend

**For unmatched requests:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  ingressClassName: nginx
  
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
  
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

### Ingress Annotations

**Nginx Ingress Controller annotations:**

```yaml
metadata:
  annotations:
    # Rewrite target
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # Backend protocol
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
    # Custom timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    
    # Client body size
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"
    
    # Whitelist source IP
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,172.16.0.0/12"
    
    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```

### Complete Example: Multi-Service Application

**1. deployments.yaml:**
```yaml
# Frontend
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
    spec:
      containers:
      - name: frontend
        image: nginx
        ports:
        - containerPort: 80

---
# Backend API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myapp/api:1.0
        ports:
        - containerPort: 8080

---
# Admin Panel
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: myapp/admin:1.0
        ports:
        - containerPort: 3000
```

**2. services.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80

---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: admin
spec:
  selector:
    app: admin
  ports:
  - port: 3000
```

**3. ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  
  tls:
  - hosts:
    - myapp.example.com
    - api.myapp.example.com
    - admin.myapp.example.com
    secretName: myapp-tls
  
  rules:
  # Frontend
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  
  # API
  - host: api.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080
  
  # Admin (with authentication)
  - host: admin.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin
            port:
              number: 3000
```

**Deploy:**
```bash
kubectl apply -f deployments.yaml
kubectl apply -f services.yaml
kubectl apply -f ingress.yaml

# Verify
kubectl get ingress
kubectl get svc
kubectl get pods
```

### Ingress Commands

```bash
# Create Ingress
kubectl apply -f ingress.yaml

# Get Ingress
kubectl get ingress
kubectl get ing

# Describe Ingress
kubectl describe ingress app-ingress

# Edit Ingress
kubectl edit ingress app-ingress

# Delete Ingress
kubectl delete ingress app-ingress

# View Ingress YAML
kubectl get ingress app-ingress -o yaml

# Get Ingress in all namespaces
kubectl get ingress -A
```

---

## 5. Network Policies

### What are Network Policies?

**By default:** All pods can communicate with all pods (no restrictions).

**Network Policy:** Firewall rules for pods.

**Use cases:**
- Restrict traffic between namespaces
- Allow only specific pods to access database
- Block egress traffic
- Implement zero-trust networking

### Requirements

**Network Policies require CNI plugin support:**
- ✓ Calico
- ✓ Cilium
- ✓ Weave Net
- ✗ Flannel (no policy support)

### Network Policy Example

**Scenario:** Allow only frontend pods to access backend pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend          # Apply to backend pods
  
  policyTypes:
  - Ingress                 # Control incoming traffic
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # Allow only from frontend pods
    ports:
    - protocol: TCP
      port: 8080
```

**Result:**
```
✓ frontend pods → backend pods (port 8080)
✗ other pods → backend pods
✗ direct access from outside
```

### Default Deny All

**Block all ingress traffic:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}          # Select all pods
  policyTypes:
  - Ingress
  # No ingress rules = deny all
```

**Block all egress traffic:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all
```

**Block all ingress and egress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}                     # Empty rule = allow all
```

### Namespace Selector

**Allow traffic from specific namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-namespace
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend    # Allow from 'frontend' namespace
    ports:
    - protocol: TCP
      port: 8080
```

**Label namespace:**
```bash
kubectl label namespace frontend name=frontend
```

### IP Block (CIDR)

**Allow traffic from specific IP range:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-cidr
spec:
  podSelector:
    matchLabels:
      app: web
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - ipBlock:
        cidr: 172.16.0.0/16
        except:
        - 172.16.1.0/24     # Exclude this subnet
    ports:
    - protocol: TCP
      port: 80
```

### Egress Policy

**Allow outbound traffic only to specific destinations:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-db
spec:
  podSelector:
    matchLabels:
      app: backend
  
  policyTypes:
  - Egress
  
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  
  # Allow database
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
```

### Complete Example: Three-Tier Application

**Application:**
```
Frontend → Backend → Database
```

**1. Database Network Policy:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend      # Only backend can access DB
    ports:
    - protocol: TCP
      port: 3306
```

**2. Backend Network Policy:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend     # Only frontend can access backend
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  
  # Allow database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 3306
```

**3. Frontend Network Policy:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      tier: frontend
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: ingress-nginx  # Allow from Ingress
    ports:
    - protocol: TCP
      port: 80
  
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  
  # Allow backend
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
```

**Traffic Flow:**
```
Internet → Ingress → Frontend → Backend → Database
           ✓         ✓          ✓         ✓

Direct access:
Internet → Backend → ✗ Blocked
Frontend → Database → ✗ Blocked
```

### Testing Network Policies

```bash
# Deploy policies
kubectl apply -f db-policy.yaml
kubectl apply -f backend-policy.yaml
kubectl apply -f frontend-policy.yaml

# Test from frontend to backend (should work)
kubectl exec frontend-pod -- curl backend-service:8080

# Test from random pod to backend (should fail)
kubectl run test --rm -it --image=busybox -- wget -O- backend-service:8080

# View policies
kubectl get networkpolicies
kubectl describe networkpolicy backend-policy
```

### Network Policy Commands

```bash
# Create
kubectl apply -f policy.yaml

# Get
kubectl get networkpolicies
kubectl get netpol

# Describe
kubectl describe networkpolicy policy-name

# Delete
kubectl delete networkpolicy policy-name

# View YAML
kubectl get networkpolicy policy-name -o yaml
```

---

## 6. Service Mesh Introduction

### What is a Service Mesh?

**Service Mesh** = Infrastructure layer for service-to-service communication.

**Provides:**
- Traffic management
- Security (mTLS)
- Observability
- Resiliency

**Popular Service Meshes:**
- **Istio** - Feature-rich, complex
- **Linkerd** - Lightweight, simple
- **Consul** - By HashiCorp
- **Kuma** - By Kong

### Why Service Mesh?

**Without Service Mesh:**
- Implement retry logic in each service
- Add circuit breakers in code
- Manually configure TLS
- Limited traffic visibility

**With Service Mesh:**
- Traffic management via configuration
- Automatic mTLS between services
- Built-in observability
- No code changes needed

### Istio Architecture (Example)

```
┌────────────────────────────────────────┐
│           Control Plane                │
│  - Pilot (traffic management)          │
│  - Citadel (security)                  │
│  - Galley (configuration)              │
└─────────────────┬──────────────────────┘
                  │
      ┌───────────┼───────────┐
      │           │           │
┌─────▼─────┐ ┌──▼────────┐ ┌▼──────────┐
│   Pod A   │ │   Pod B   │ │  Pod C    │
│ ┌───────┐ │ │ ┌───────┐ │ │ ┌───────┐ │
│ │ App   │ │ │ │ App   │ │ │ │ App   │ │
│ └───────┘ │ │ └───────┘ │ │ └───────┘ │
│ ┌───────┐ │ │ ┌───────┐ │ │ ┌───────┐ │
│ │Envoy  │ │ │ │Envoy  │ │ │ │Envoy  │ │
│ │Proxy  │◄┼─┼►│Proxy  │◄┼─┼►│Proxy  │ │
│ └───────┘ │ │ └───────┘ │ │ └───────┘ │
└───────────┘ └───────────┘ └───────────┘
```

**Sidecar Pattern:**
- Envoy proxy injected into each pod
- All traffic goes through proxy
- Proxy handles retries, timeouts, TLS, etc.

### Basic Service Mesh Features

**Traffic Management:**
```yaml
# Canary deployment: 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

**Automatic mTLS:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # All traffic encrypted
```

---

## Summary - Part 4

**What You Learned:**

1. **Kubernetes Networking**
   - Network model (flat network, no NAT)
   - CNI plugins
   - kube-proxy modes
   - Pod-to-pod and service communication

2. **DNS**
   - CoreDNS
   - Service DNS naming
   - Pod DNS records
   - DNS configuration and tuning

3. **Ingress Controllers**
   - What Ingress is and why use it
   - Popular controllers (Nginx, Traefik, etc.)
   - Installing Ingress controller

4. **Ingress Resources**
   - Basic routing
   - Multiple hosts
   - TLS/HTTPS
   - Path types
   - Annotations

5. **Network Policies**
   - Default deny/allow
   - Ingress and egress rules
   - Namespace and pod selectors
   - IP blocks

6. **Service Mesh**
   - Introduction to service mesh
   - Why use it
   - Basic concepts

**Next:**
- **Part 5**: Security, RBAC, Best Practices, Real-World Scenarios, Interview Questions

---

**Continue to Part 5 for the final section!** 🚀
