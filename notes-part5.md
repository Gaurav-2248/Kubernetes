# Kubernetes Zero to Hero - Part 5: Security, RBAC & Interview Prep

## Table of Contents
1. [Kubernetes Security Fundamentals](#kubernetes-security-fundamentals)
2. [Authentication & Authorization](#authentication-authorization)
3. [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
4. [Security Best Practices](#security-best-practices)
5. [Real-World Scenarios](#real-world-scenarios)
6. [Troubleshooting Guide](#troubleshooting-guide)
7. [Interview Questions & Answers](#interview-questions-answers)
8. [Hands-on Practice Labs](#hands-on-practice-labs)

---

## 1. Kubernetes Security Fundamentals

### The 4C's of Cloud Native Security

```
┌─────────────────────────────────────────┐
│           Code                          │
│  (Application vulnerabilities)          │
├─────────────────────────────────────────┤
│           Container                     │
│  (Image vulnerabilities, runtime)       │
├─────────────────────────────────────────┤
│           Cluster                       │
│  (K8s configuration, RBAC, policies)    │
├─────────────────────────────────────────┤
│           Cloud/Infrastructure          │
│  (Network, physical security)           │
└─────────────────────────────────────────┘
```

### Security Layers in Kubernetes

**1. API Server Security**
- All cluster operations go through API server
- First line of defense

**2. Authentication**
- Who are you?
- Methods: certificates, tokens, OIDC

**3. Authorization**
- What can you do?
- RBAC, ABAC, Webhook

**4. Admission Control**
- Can this be allowed?
- Validates/mutates requests

**5. Network Policies**
- Control pod-to-pod traffic

**6. Pod Security**
- What can pods do?
- SecurityContext, Pod Security Standards

### Attack Surface

**Common attack vectors:**
```
External:
- Exposed services without authentication
- Vulnerable container images
- Misconfigured Ingress

Internal:
- Overly permissive RBAC
- Pods running as root
- No network policies
- Secrets in environment variables
- Privileged containers
```

---

## 2. Authentication & Authorization

### Authentication Methods

**1. X.509 Client Certificates**
```bash
# Generate client certificate
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=john/O=developers"

# Sign with cluster CA
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out user.crt -days 365

# Create kubeconfig
kubectl config set-credentials john --client-certificate=user.crt --client-key=user.key
kubectl config set-context john-context --cluster=my-cluster --user=john
```

**2. Service Account Tokens**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
```

```bash
# Create service account
kubectl create serviceaccount my-service-account

# Get token
kubectl create token my-service-account

# Use in pod
kubectl run test --image=nginx --serviceaccount=my-service-account
```

**3. Static Token File** (Not recommended)
```bash
# /etc/kubernetes/pki/tokens.csv
token1,user1,uid1,"group1,group2"
token2,user2,uid2,"group3"
```

**4. Bootstrap Tokens**
```bash
# Used for node joining
kubeadm token create
```

**5. OpenID Connect (OIDC)**
```yaml
# API server configuration
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=kubernetes
--oidc-username-claim=email
```

### Service Accounts

**Every pod gets a service account:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: default  # Default if not specified
  containers:
  - name: app
    image: myapp:1.0
```

**Service account token automatically mounted:**
```bash
kubectl exec my-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt
# namespace
# token
```

**Create custom service account:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
automountServiceAccountToken: false  # Don't auto-mount token
```

**Use in pod:**
```yaml
spec:
  serviceAccountName: app-service-account
  automountServiceAccountToken: false  # Override if needed
```

### Authorization Modes

**1. AlwaysAllow** - Allow all (insecure, don't use)

**2. AlwaysDeny** - Deny all (not useful)

**3. ABAC (Attribute-Based Access Control)** - Legacy, complex

**4. RBAC (Role-Based Access Control)** - ✓ Recommended

**5. Node** - Special authorization for kubelets

**6. Webhook** - External authorization service

**API Server configuration:**
```bash
--authorization-mode=Node,RBAC
```

---

## 3. RBAC (Role-Based Access Control)

### RBAC Concepts

**Four main resource types:**

1. **Role** - Permissions within a namespace
2. **ClusterRole** - Cluster-wide permissions
3. **RoleBinding** - Grants Role to users/groups/SAs
4. **ClusterRoleBinding** - Grants ClusterRole cluster-wide

### Role vs ClusterRole

| Aspect | Role | ClusterRole |
|--------|------|-------------|
| **Scope** | Namespace | Cluster-wide |
| **Resources** | Namespaced resources | All resources including cluster-scoped |
| **Use Case** | App-specific access | Admin access, cluster resources |

### Creating a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]

- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**Common verbs:**
- `get` - Read individual resource
- `list` - List resources
- `watch` - Watch for changes
- `create` - Create new resource
- `update` - Update existing resource
- `patch` - Partially update resource
- `delete` - Delete resource
- `deletecollection` - Delete multiple resources

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# User
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io

# Group
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io

# Service Account
- kind: ServiceAccount
  name: app-service-account
  namespace: default

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
# Read all resources
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

# Access nodes
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

# Access PersistentVolumes (cluster-scoped)
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch"]
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-read-binding
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

### Built-in ClusterRoles

```bash
# View built-in ClusterRoles
kubectl get clusterroles

# Important default roles:
# cluster-admin - Super user (full access)
# admin - Admin within namespace
# edit - Read/write in namespace
# view - Read-only in namespace
```

**Using built-in roles:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-edit
  namespace: development
subjects:
- kind: User
  name: developer1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit              # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

### Real-World RBAC Examples

**1. Developer Role - Can manage apps, but not cluster resources**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
# Manage pods, deployments, services
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "replicasets", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# View logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]

# Execute into pods
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

**2. Read-Only User**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: viewer
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

**3. CI/CD Service Account**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer-role
  namespace: production
rules:
# Deploy applications
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Manage services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update"]

# Read ConfigMaps and Secrets
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: production
roleRef:
  kind: Role
  name: ci-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

**4. Namespace Admin**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-admin
  namespace: team-a
subjects:
- kind: User
  name: team-a-lead
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin              # Built-in admin role
  apiGroup: rbac.authorization.k8s.io
```

### Testing RBAC

**Check access:**
```bash
# Can I create deployments?
kubectl auth can-i create deployments

# Can user john create pods in default namespace?
kubectl auth can-i create pods --as=john --namespace=default

# Can service account ci-deployer update deployments?
kubectl auth can-i update deployments --as=system:serviceaccount:production:ci-deployer -n production

# What can I do?
kubectl auth can-i --list

# What can john do in production namespace?
kubectl auth can-i --list --as=john --namespace=production
```

### RBAC Commands

```bash
# Roles
kubectl get roles
kubectl get roles -A
kubectl describe role developer

# RoleBindings
kubectl get rolebindings
kubectl get rolebindings -A
kubectl describe rolebinding developer-binding

# ClusterRoles
kubectl get clusterroles
kubectl describe clusterrole admin

# ClusterRoleBindings
kubectl get clusterrolebindings
kubectl describe clusterrolebinding cluster-admin

# Create role
kubectl create role pod-reader --verb=get,list,watch --resource=pods

# Create rolebinding
kubectl create rolebinding read-pods --role=pod-reader --user=john

# Create clusterrole
kubectl create clusterrole cluster-reader --verb=get,list,watch --resource=*

# Create clusterrolebinding
kubectl create clusterrolebinding cluster-read --clusterrole=cluster-reader --user=jane
```

---

## 4. Security Best Practices

### 1. Pod Security

**SecurityContext - Pod Level:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true           # Don't allow root
    runAsUser: 1000              # Run as user 1000
    fsGroup: 2000                # File system group
    seccompProfile:              # Seccomp profile
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL                    # Drop all capabilities
        add:
        - NET_BIND_SERVICE       # Only add specific ones
```

**Key security settings:**

| Setting | Description | Recommended |
|---------|-------------|-------------|
| `runAsNonRoot` | Don't run as root | `true` |
| `readOnlyRootFilesystem` | Read-only root FS | `true` |
| `allowPrivilegeEscalation` | Prevent privilege escalation | `false` |
| `privileged` | Privileged container | `false` |
| `capabilities` | Linux capabilities | Drop ALL, add specific |

### 2. Pod Security Standards

**Three levels:**

1. **Privileged** - Unrestricted (default)
2. **Baseline** - Minimally restrictive
3. **Restricted** - Heavily restricted (recommended)

**Enable at namespace level:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 3. Image Security

**Use specific image tags:**
```yaml
# Bad
image: nginx:latest          # ❌ Unpredictable

# Good
image: nginx:1.21.6          # ✓ Specific version
image: nginx:1.21.6@sha256:abc123...  # ✓ Immutable digest
```

**Private registry:**
```yaml
spec:
  containers:
  - name: app
    image: myregistry.com/myapp:1.0
  imagePullSecrets:
  - name: registry-credentials
```

**Image pull policy:**
```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: Always    # Always pull (recommended for :latest)
    # imagePullPolicy: IfNotPresent  # Pull if not cached
    # imagePullPolicy: Never    # Never pull
```

**Scan images for vulnerabilities:**
```bash
# Using Trivy
trivy image nginx:1.21

# Using Clair, Anchore, etc.
```

### 4. Secrets Management

**Don't:**
```yaml
# ❌ Hardcoded
env:
- name: PASSWORD
  value: "mysecretpassword"

# ❌ In Git
apiVersion: v1
kind: Secret
data:
  password: bXlzZWNyZXQ=
```

**Do:**
```yaml
# ✓ From Secret
env:
- name: PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# ✓ Mount as volume
volumeMounts:
- name: secrets
  mountPath: /secrets
  readOnly: true

# ✓ Use external secret managers
# - HashiCorp Vault
# - AWS Secrets Manager
# - Azure Key Vault
```

### 5. Network Security

**Network Policies:**
```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Then allow specific traffic:**
```yaml
# Allow frontend → backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 6. RBAC Best Practices

**Principle of Least Privilege:**
```yaml
# Bad - Too permissive
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Good - Specific permissions
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

**Don't use cluster-admin:**
```bash
# Bad
kubectl create clusterrolebinding admin-binding \
  --clusterrole=cluster-admin \
  --user=developer

# Good - Use specific roles
kubectl create rolebinding developer-binding \
  --clusterrole=edit \
  --user=developer \
  --namespace=development
```

### 7. Resource Limits

**Always set resource limits:**
```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

**Enforce with LimitRange:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  - max:
      memory: 1Gi
      cpu: 1
    min:
      memory: 64Mi
      cpu: 100m
    default:
      memory: 256Mi
      cpu: 500m
    defaultRequest:
      memory: 128Mi
      cpu: 250m
    type: Container
```

### 8. Audit Logging

**Enable audit logs:**
```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  omitStages:
  - RequestReceived

- level: Request
  verbs: ["create", "update", "delete"]
  resources:
  - group: "apps"
    resources: ["deployments"]
```

**API server configuration:**
```bash
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

### 9. etcd Security

**Encrypt secrets at rest:**
```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

**API server configuration:**
```bash
--encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```

### 10. Admission Controllers

**Recommended admission controllers:**
```bash
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,PodSecurityPolicy,NodeRestriction
```

**PodSecurity Admission:**
```yaml
# Enforce security standards
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      audit: "restricted"
      warn: "restricted"
```

---

## 5. Real-World Scenarios

### Scenario 1: Application Deployment

**Problem:** Deploy a 3-tier web application (frontend, backend, database) with proper security.

**Solution:**

**1. Namespace:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

**2. Database (StatefulSet):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: webapp
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
      tier: database
  template:
    metadata:
      labels:
        app: mysql
        tier: database
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

**3. Backend (Deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      tier: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      serviceAccountName: backend-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: backend
        image: myapp/backend:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: mysql.webapp.svc.cluster.local
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
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
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**4. Frontend (Deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: frontend
        image: myapp/frontend:1.0
        ports:
        - containerPort: 80
        env:
        - name: API_URL
          value: http://backend.webapp.svc.cluster.local:8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

**5. Services:**
```yaml
# MySQL Service (Headless)
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: webapp
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
---
# Backend Service
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: webapp
spec:
  selector:
    app: backend
  ports:
  - port: 8080
---
# Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: webapp
spec:
  selector:
    app: frontend
  ports:
  - port: 80
```

**6. Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: webapp
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: webapp-tls
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

**7. Network Policies:**
```yaml
# Default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: webapp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Allow backend → database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-database
  namespace: webapp
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
          tier: backend
    ports:
    - protocol: TCP
      port: 3306
---
# Allow Ingress → Frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-frontend
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
```

### Scenario 2: Pod CrashLoopBackOff

**Problem:** Pod keeps restarting.

**Troubleshooting Steps:**

```bash
# 1. Check pod status
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# myapp-x   0/1     CrashLoopBackOff   5          3m

# 2. Describe pod (check events)
kubectl describe pod myapp-x

# Look for:
# - Image pull errors
# - Insufficient resources
# - Failed liveness probes
# - Exit code

# 3. Check logs
kubectl logs myapp-x
kubectl logs myapp-x --previous  # Logs from crashed container

# 4. Check resource usage
kubectl top pod myapp-x

# 5. Common causes and solutions:

# Cause 1: Wrong command/entrypoint
# Solution: Check container command
kubectl get pod myapp-x -o jsonpath='{.spec.containers[0].command}'

# Cause 2: Missing dependencies
# Solution: Check application logs

# Cause 3: Liveness probe failing too early
# Solution: Increase initialDelaySeconds
livenessProbe:
  initialDelaySeconds: 60  # Increase delay

# Cause 4: Out of memory
# Solution: Increase memory limit
resources:
  limits:
    memory: "512Mi"  # Increase

# Cause 5: Configuration error
# Solution: Check ConfigMap/Secret
kubectl get configmap myconfig -o yaml
```

### Scenario 3: Service Not Accessible

**Problem:** Cannot access service.

**Troubleshooting:**

```bash
# 1. Check service exists
kubectl get svc myservice

# 2. Check service endpoints
kubectl get endpoints myservice
# Should show pod IPs

# If no endpoints:
# - Check selector matches pod labels
kubectl get svc myservice -o yaml  # Check selector
kubectl get pods --show-labels     # Check pod labels

# 3. Check pods are running and ready
kubectl get pods -l app=myapp

# 4. Test from within cluster
kubectl run test --rm -it --image=busybox -- sh
wget -O- myservice:80

# 5. Check service type
kubectl get svc myservice
# For NodePort: curl http://<node-ip>:<node-port>
# For LoadBalancer: curl http://<external-ip>

# 6. Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy policy-name

# 7. Check Ingress (if using)
kubectl get ingress
kubectl describe ingress myingress

# 8. Check DNS
kubectl run test --rm -it --image=busybox -- nslookup myservice
```

### Scenario 4: High Resource Usage

**Problem:** Node running out of resources.

**Solution:**

```bash
# 1. Check node resources
kubectl top nodes

# 2. Check pod resources
kubectl top pods -A --sort-by=memory
kubectl top pods -A --sort-by=cpu

# 3. Identify resource hogs
kubectl describe node node-name
# Look for "Allocated resources" section

# 4. Set resource limits
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"

# 5. Implement HPA (Horizontal Pod Autoscaler)
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=80

# 6. Implement VPA (Vertical Pod Autoscaler)
# Automatically adjust resource requests

# 7. Add more nodes or upgrade node size
```

---

## 6. Troubleshooting Guide

### Common Issues & Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **ImagePullBackOff** | Can't pull image | Check image name, registry credentials |
| **CrashLoopBackOff** | Pod keeps restarting | Check logs, increase liveness delay |
| **Pending** | Pod not scheduled | Check resources, node selectors, taints |
| **ErrImagePull** | Invalid image | Verify image exists and is accessible |
| **OOMKilled** | Out of memory | Increase memory limit |
| **CreateContainerConfigError** | Config error | Check ConfigMap/Secret exists |
| **0/X nodes available** | No nodes match | Check node selectors, taints, resources |

### Debug Commands

```bash
# Pod debugging
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod> -f
kubectl logs <pod> --previous
kubectl exec -it <pod> -- /bin/bash
kubectl get events --sort-by='.lastTimestamp'

# Service debugging
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>

# Network debugging
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Resource usage
kubectl top nodes
kubectl top pods

# Check API server
kubectl cluster-info
kubectl get componentstatuses

# Check certificates
kubeadm certs check-expiration

# etcd health
ETCDCTL_API=3 etcdctl endpoint health

# View full config
kubectl config view
```

---

## 7. Interview Questions & Answers

### Beginner Level

**Q1: What is Kubernetes?**

**A:** Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It was originally developed by Google and is now maintained by the Cloud Native Computing Foundation (CNCF).

**Q2: What is a Pod?**

**A:** A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in a cluster and can contain one or more containers that share the same network namespace, IP address, and storage volumes. Containers within the same pod can communicate via localhost.

**Q3: What is the difference between Docker and Kubernetes?**

**A:** 
- **Docker** is a containerization platform that packages applications and their dependencies into containers.
- **Kubernetes** is a container orchestration platform that manages and orchestrates multiple containers across multiple hosts.
- Docker creates containers; Kubernetes manages them at scale.

**Q4: What is a Namespace in Kubernetes?**

**A:** A Namespace is a virtual cluster within a physical Kubernetes cluster. It provides a way to divide cluster resources between multiple users or teams. Namespaces are useful for:
- Multi-tenancy
- Environment separation (dev, staging, prod)
- Resource isolation and quotas
- RBAC boundaries

**Q5: What is kubectl?**

**A:** kubectl is the command-line tool used to interact with Kubernetes clusters. It communicates with the Kubernetes API server to:
- Deploy applications
- Inspect and manage cluster resources
- View logs
- Troubleshoot issues

**Q6: What are the main components of Kubernetes architecture?**

**A:**
**Control Plane (Master Node):**
- API Server - Entry point for all operations
- etcd - Key-value store for cluster data
- Scheduler - Assigns pods to nodes
- Controller Manager - Maintains desired state
- Cloud Controller Manager - Cloud provider integration

**Worker Nodes:**
- Kubelet - Manages pods on the node
- Kube-proxy - Network proxy
- Container Runtime - Runs containers (Docker, containerd)

**Q7: What is the difference between a Deployment and a ReplicaSet?**

**A:**
- **ReplicaSet** ensures a specified number of pod replicas are running at any time. It provides basic scaling and self-healing.
- **Deployment** is a higher-level abstraction that manages ReplicaSets. It provides additional features like:
  - Rolling updates
  - Rollback capability
  - Declarative updates
  - Version history
  
In practice, always use Deployments instead of directly using ReplicaSets.

**Q8: What types of Services are available in Kubernetes?**

**A:**
1. **ClusterIP** (default) - Exposes service on an internal IP, accessible only within the cluster
2. **NodePort** - Exposes service on each node's IP at a static port (30000-32767)
3. **LoadBalancer** - Creates an external load balancer in cloud environments
4. **ExternalName** - Maps service to a DNS name

### Intermediate Level

**Q9: Explain the difference between ConfigMap and Secret.**

**A:**

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| Purpose | Non-sensitive configuration data | Sensitive data (passwords, tokens, keys) |
| Encoding | Plain text | Base64 encoded |
| Encryption | Not encrypted | Can be encrypted at rest |
| Use Case | Application settings, config files | Credentials, certificates, API keys |
| Size Limit | 1MB | 1MB |

Both can be consumed as environment variables or mounted as volumes in pods.

**Q10: What are Liveness and Readiness probes?**

**A:**
- **Liveness Probe**: Determines if a container is running. If it fails, Kubernetes kills the container and restarts it according to the restart policy.
  - Use case: Detect deadlocks, infinite loops
  
- **Readiness Probe**: Determines if a container is ready to serve traffic. If it fails, the pod is removed from service endpoints, but the container is not restarted.
  - Use case: Wait for dependencies, warm-up period

**Startup Probe** (added later): Used for slow-starting containers. Disables liveness/readiness checks until it succeeds.

**Q11: What is an Ingress and how does it differ from a Service?**

**A:**
- **Service** provides L4 (TCP/UDP) load balancing and exposes pods within or outside the cluster.
- **Ingress** provides L7 (HTTP/HTTPS) load balancing with additional features:
  - Host-based routing (virtual hosting)
  - Path-based routing
  - SSL/TLS termination
  - Single entry point for multiple services
  
Ingress requires an Ingress Controller (nginx, traefik, etc.) to function.

**Q12: What is the purpose of a StatefulSet?**

**A:** StatefulSet is used for stateful applications that require:
- **Stable network identity**: Pods get predictable names (app-0, app-1, app-2)
- **Stable persistent storage**: Each pod gets its own PersistentVolume
- **Ordered deployment**: Pods are created/deleted in order (0→1→2, then 2→1→0)
- **Ordered updates**: Updates happen one at a time in order

Use cases: Databases, message queues, distributed systems (Cassandra, MongoDB, Kafka)

**Q13: Explain DaemonSet.**

**A:** DaemonSet ensures that a copy of a pod runs on all (or some) nodes in the cluster. When nodes are added/removed, pods are automatically added/removed.

**Use cases:**
- Log collection (Fluentd, Logstash)
- Monitoring agents (Prometheus Node Exporter, Datadog agent)
- Network proxies (kube-proxy)
- Storage daemons (Ceph, Gluster)

**Q14: What is a PersistentVolume (PV) and PersistentVolumeClaim (PVC)?**

**A:**
- **PersistentVolume (PV)**: A piece of storage in the cluster, provisioned by an admin or dynamically via StorageClasses. It's a cluster-wide resource.
  
- **PersistentVolumeClaim (PVC)**: A request for storage by a user. It's namespaced and binds to an available PV that matches the request.

**Relationship:**
```
Admin creates PV → User creates PVC → PVC binds to PV → Pod uses PVC
```

**Q15: What are Init Containers?**

**A:** Init Containers are specialized containers that run before application containers in a pod. They:
- Run to completion before app containers start
- Run sequentially (one after another)
- Must succeed for the pod to start

**Use cases:**
- Wait for dependencies (database, service)
- Populate data or configuration
- Security/compliance checks
- Clone git repositories

### Advanced Level

**Q16: Explain RBAC in Kubernetes.**

**A:** RBAC (Role-Based Access Control) is the primary authorization mechanism in Kubernetes.

**Four main resources:**
1. **Role**: Defines permissions within a namespace
2. **ClusterRole**: Defines permissions cluster-wide
3. **RoleBinding**: Grants Role to users/groups/service accounts
4. **ClusterRoleBinding**: Grants ClusterRole cluster-wide

**Example:**
```yaml
# Role: Can read pods in 'default' namespace
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# RoleBinding: Grant role to user 'john'
kind: RoleBinding
subjects:
- kind: User
  name: john
roleRef:
  kind: Role
  name: pod-reader
```

**Q17: How does Kubernetes networking work?**

**A:** Kubernetes follows a flat network model with these requirements:
1. Every pod gets its own unique IP address
2. All pods can communicate with each other without NAT
3. All nodes can communicate with all pods without NAT
4. The IP a pod sees itself as is the same IP others see it as

**Implementation:**
- **CNI (Container Network Interface)** plugins implement the network model (Calico, Flannel, Weave, Cilium)
- **kube-proxy** implements Services and load balancing using iptables or IPVS
- **CoreDNS** provides service discovery via DNS

**Q18: What are Network Policies?**

**A:** Network Policies are firewall rules for pods that control traffic at the IP address or port level.

**Default behavior**: All pods can communicate with all pods (no restrictions).

**With Network Policy**, you can:
- Allow/deny ingress (incoming) traffic
- Allow/deny egress (outgoing) traffic
- Select pods by labels, namespaces, or IP blocks

**Requirements**: CNI plugin must support Network Policies (Calico, Cilium, Weave - Flannel does NOT).

**Q19: Explain Rolling Updates and Rollbacks.**

**A:**

**Rolling Update:**
- Gradually replaces old pods with new pods
- Zero downtime deployment
- Controlled by `maxSurge` and `maxUnavailable`

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Max 1 extra pod during update
    maxUnavailable: 0    # No pods unavailable (zero downtime)
```

**Process:**
1. Create new ReplicaSet with new version
2. Scale up new ReplicaSet
3. Scale down old ReplicaSet
4. Repeat until complete

**Rollback:**
```bash
# Automatic rollback if update fails
# Manual rollback
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2
```

**Q20: What is a Service Mesh?**

**A:** A Service Mesh is an infrastructure layer that handles service-to-service communication. It uses sidecar proxies injected into each pod.

**Popular Service Meshes:**
- **Istio** - Feature-rich, complex
- **Linkerd** - Lightweight, simple
- **Consul Connect** - By HashiCorp

**Provides:**
- **Traffic Management**: Canary deployments, A/B testing, circuit breaking
- **Security**: Automatic mTLS, authentication, authorization
- **Observability**: Distributed tracing, metrics, logging
- **Resiliency**: Retries, timeouts, rate limiting

**Without Service Mesh**: Implement in application code
**With Service Mesh**: Configure via infrastructure (no code changes)

**Q21: How does HPA (Horizontal Pod Autoscaler) work?**

**A:** HPA automatically scales the number of pods based on observed metrics (CPU, memory, custom metrics).

**How it works:**
1. Metrics Server collects resource usage
2. HPA controller queries Metrics Server every 15s (default)
3. Calculates desired replicas: `desiredReplicas = ceil[currentReplicas * (currentMetricValue / targetMetricValue)]`
4. Scales deployment up/down

**Example:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80  # Scale when > 80% CPU
```

**Q22: What is etcd and why is it important?**

**A:** etcd is a distributed key-value store that serves as Kubernetes' backing store for all cluster data.

**Stores:**
- Cluster state
- Configuration data
- Secrets
- Service discovery information

**Critical characteristics:**
- Highly available (typically 3 or 5 nodes)
- Consistent (uses Raft consensus algorithm)
- Source of truth for cluster state

**If etcd fails:**
- Cannot create/update/delete resources
- Existing workloads continue running
- Cluster is effectively frozen

**Best practices:**
- Regular backups
- Run on dedicated nodes
- Enable encryption at rest
- Limit access (only API server should access)

**Q23: Explain Taints and Tolerations.**

**A:**

**Taints**: Applied to nodes to repel pods
**Tolerations**: Applied to pods to allow scheduling on tainted nodes

**Use cases:**
- Dedicate nodes to specific workloads
- Isolate problematic workloads
- Handle special hardware (GPU nodes)

**Example:**
```bash
# Taint node
kubectl taint nodes node1 key=value:NoSchedule

# Pod tolerates taint
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

**Taint Effects:**
- **NoSchedule**: Don't schedule new pods
- **PreferNoSchedule**: Try to avoid scheduling
- **NoExecute**: Evict existing pods + don't schedule new ones

**Q24: What is the difference between Node Affinity and Pod Affinity?**

**A:**

**Node Affinity**: Schedule pods to specific nodes based on node labels
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

**Pod Affinity**: Schedule pods near other pods
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - cache
      topologyKey: kubernetes.io/hostname
```

**Pod Anti-Affinity**: Schedule pods away from other pods (e.g., spread replicas across nodes)

**Q25: What are Operators in Kubernetes?**

**A:** Operators are software extensions that use custom resources to manage applications and their components.

**Concept**: Encode operational knowledge into software
- **CRD (Custom Resource Definition)**: Extend Kubernetes API with custom resources
- **Controller**: Watches CRDs and takes action

**Examples:**
- **Prometheus Operator**: Manages Prometheus deployments
- **MySQL Operator**: Manages MySQL clusters
- **Elasticsearch Operator**: Manages Elasticsearch clusters

**Operator Pattern:**
1. Define custom resource
2. User creates custom resource instance
3. Operator controller watches for changes
4. Operator takes action to reconcile desired state

---

## 8. Hands-on Practice Labs

### Lab 1: Deploy Multi-Tier Application

**Objective**: Deploy WordPress with MySQL

**Steps:**
```bash
# 1. Create namespace
kubectl create namespace wordpress

# 2. Create MySQL secret
kubectl create secret generic mysql-secret \
  --from-literal=password=mypassword \
  -n wordpress

# 3. Deploy MySQL
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
EOF

# 4. Deploy WordPress
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: LoadBalancer
  selector:
    app: wordpress
  ports:
  - port: 80
    targetPort: 80
EOF

# 5. Get service
kubectl get svc -n wordpress

# 6. Access WordPress
# Use LoadBalancer IP or port-forward
kubectl port-forward svc/wordpress 8080:80 -n wordpress
# Open: http://localhost:8080
```

### Lab 2: Implement RBAC

**Objective**: Create developer role with limited permissions

```bash
# 1. Create namespace
kubectl create namespace development

# 2. Create user certificate (simulate)
# In reality, integrate with OIDC or use certificates

# 3. Create Role
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
EOF

# 4. Create ServiceAccount (simulate user)
kubectl create serviceaccount developer -n development

# 5. Create RoleBinding
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: developer
  namespace: development
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
EOF

# 6. Test permissions
# Get service account token
TOKEN=$(kubectl create token developer -n development)

# Test with token
kubectl --token=$TOKEN get pods -n development  # ✓ Allowed
kubectl --token=$TOKEN get secrets -n development  # ✗ Forbidden
kubectl --token=$TOKEN get nodes  # ✗ Forbidden

# 7. Check what developer can do
kubectl auth can-i --list --as=system:serviceaccount:development:developer -n development
```

### Lab 3: Configure Network Policies

**Objective**: Implement network isolation

```bash
# 1. Create namespaces
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# 2. Label namespaces
kubectl label namespace frontend tier=frontend
kubectl label namespace backend tier=backend
kubectl label namespace database tier=database

# 3. Deploy applications
# Frontend
kubectl run frontend --image=nginx -n frontend --labels="tier=frontend"
kubectl expose pod frontend --port=80 -n frontend

# Backend
kubectl run backend --image=nginx -n backend --labels="tier=backend"
kubectl expose pod backend --port=80 -n backend

# Database
kubectl run database --image=redis -n database --labels="tier=database"
kubectl expose pod database --port=6379 -n database

# 4. Default deny all
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# 5. Allow frontend → backend
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# 6. Allow backend → database
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-backend
  namespace: database
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 6379
EOF

# 7. Test connectivity
# Frontend → Backend (should work)
kubectl exec -n frontend frontend -- wget -O- backend.backend

# Frontend → Database (should fail)
kubectl exec -n frontend frontend -- nc -zv database.database 6379

# Backend → Database (should work)
kubectl exec -n backend backend -- nc -zv database.database 6379
```

---

## Summary - Complete Kubernetes Journey

### What You've Learned

**Part 1: Fundamentals**
- Containerization basics
- Kubernetes architecture
- Setting up local cluster
- Basic kubectl commands

**Part 2: Core Objects**
- Pods, ReplicaSets, Deployments
- Services and service discovery
- Labels and selectors
- Namespaces

**Part 3: Configuration & Storage**
- ConfigMaps and Secrets
- Volumes and persistent storage
- StatefulSets

**Part 4: Networking**
- Kubernetes networking model
- DNS and service discovery
- Ingress and Ingress controllers
- Network Policies

**Part 5: Security & Production**
- Authentication and authorization
- RBAC implementation
- Security best practices
- Real-world troubleshooting
- Interview preparation

### Next Steps

**1. Hands-on Practice**
- Set up local cluster (Minikube/Kind)
- Deploy sample applications
- Break things and fix them

**2. Certifications**
- **CKA** (Certified Kubernetes Administrator)
- **CKAD** (Certified Kubernetes Application Developer)
- **CKS** (Certified Kubernetes Security Specialist)

**3. Advanced Topics**
- Service Mesh (Istio, Linkerd)
- GitOps (ArgoCD, Flux)
- Observability (Prometheus, Grafana)
- CI/CD with Kubernetes

**4. Resources**
- kubernetes.io documentation
- KodeKloud, A Cloud Guru courses
- Kubernetes Slack community
- CNCF projects

---

## Quick Reference Card

### Essential Commands
```bash
# Cluster
kubectl cluster-info
kubectl get nodes

# Pods
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod> -f
kubectl exec -it <pod> -- sh

# Deployments
kubectl get deployments
kubectl create deployment <name> --image=<image>
kubectl scale deployment <name> --replicas=<n>
kubectl set image deployment/<name> <container>=<new-image>
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>

# Services
kubectl get svc
kubectl expose deployment <name> --port=<port> --type=<type>

# ConfigMaps & Secrets
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value

# Apply/Delete
kubectl apply -f <file>
kubectl delete -f <file>

# Troubleshooting
kubectl describe <resource> <name>
kubectl get events
kubectl top nodes
kubectl top pods
```

### YAML Template
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        env:
        - name: ENV_VAR
          value: "value"
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
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

---



**End of Part 5**
