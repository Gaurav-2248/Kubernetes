
## Table of Contents
1. [ConfigMaps - Configuration Management](#configmaps-configuration-management)
2. [Secrets - Sensitive Data](#secrets-sensitive-data)
3. [Volumes - Data Persistence](#volumes-data-persistence)
4. [Persistent Volumes & Claims](#persistent-volumes-claims)
5. [Storage Classes](#storage-classes)
6. [StatefulSets](#statefulsets)

---

## 1. ConfigMaps - Configuration Management

### What is a ConfigMap?

**ConfigMap** stores non-sensitive configuration data as key-value pairs.

**Why use ConfigMaps?**
- Separate configuration from application code
- Same image, different configs for dev/staging/prod
- Update configuration without rebuilding images
- Environment-specific settings

### The Problem ConfigMaps Solve

**Bad Approach: Hardcoded Configuration**
```dockerfile
# Dockerfile
ENV DATABASE_HOST=mysql.prod.com
ENV DATABASE_PORT=3306
ENV LOG_LEVEL=debug

# Need different settings for staging?
# → Build different image ❌
# → Can't reuse same image ❌
```

**Good Approach: ConfigMap**
```
Same Docker image → Different ConfigMaps per environment
   ↓                      ↓                    ↓
Development         Staging               Production
ConfigMap          ConfigMap             ConfigMap
```

### Creating ConfigMaps

#### Method 1: From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql.default.svc.cluster.local \
  --from-literal=DATABASE_PORT=3306 \
  --from-literal=LOG_LEVEL=info
```

#### Method 2: From File

**app.properties:**
```properties
DATABASE_HOST=mysql.default.svc.cluster.local
DATABASE_PORT=3306
LOG_LEVEL=info
MAX_CONNECTIONS=100
```

```bash
kubectl create configmap app-config --from-file=app.properties
```

#### Method 3: From Directory

```bash
# Creates ConfigMap with all files in directory
kubectl create configmap app-config --from-file=./config/
```

#### Method 4: Declarative YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  DATABASE_HOST: mysql.default.svc.cluster.local
  DATABASE_PORT: "3306"
  LOG_LEVEL: info
  
  # Multi-line configuration file
  app.properties: |
    database.host=mysql.default.svc.cluster.local
    database.port=3306
    log.level=info
    max.connections=100
  
  # Another config file
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
        }
    }
```

```bash
kubectl apply -f configmap.yaml
```

### Viewing ConfigMaps

```bash
# List ConfigMaps
kubectl get configmaps
kubectl get cm

# Describe ConfigMap
kubectl describe configmap app-config

# View as YAML
kubectl get configmap app-config -o yaml

# Edit ConfigMap
kubectl edit configmap app-config

# Delete ConfigMap
kubectl delete configmap app-config
```

### Using ConfigMaps in Pods

#### Option 1: Environment Variables (Individual Keys)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Single environment variable from ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
```

#### Option 2: All Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    # Load ALL keys from ConfigMap as environment variables
    - configMapRef:
        name: app-config
```

**Result inside container:**
```bash
# All ConfigMap keys become environment variables
echo $DATABASE_HOST
# Output: mysql.default.svc.cluster.local

echo $DATABASE_PORT
# Output: 3306

echo $LOG_LEVEL
# Output: info
```

#### Option 3: Mount as Volume (Files)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

**Result inside container:**
```bash
# ConfigMap keys become files
ls /etc/config/
# Output:
# DATABASE_HOST
# DATABASE_PORT
# LOG_LEVEL
# app.properties
# nginx.conf

cat /etc/config/DATABASE_HOST
# Output: mysql.default.svc.cluster.local

cat /etc/config/app.properties
# Output: database.host=mysql.default.svc.cluster.local
#         database.port=3306
#         ...
```

#### Option 4: Mount Specific Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf          # Mount only this key as a file
  
  volumes:
  - name: nginx-config
    configMap:
      name: app-config
      items:
      - key: nginx.conf            # Key from ConfigMap
        path: nginx.conf           # Filename in container
```

### Complete Example: Multi-Environment Setup

**dev-config.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  ENVIRONMENT: development
  DATABASE_HOST: mysql-dev.development.svc.cluster.local
  LOG_LEVEL: debug
  ENABLE_DEBUG: "true"
```

**prod-config.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  ENVIRONMENT: production
  DATABASE_HOST: mysql-prod.production.svc.cluster.local
  LOG_LEVEL: warning
  ENABLE_DEBUG: "false"
```

**deployment.yaml (same for both environments):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
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
      - name: app
        image: myapp:1.0          # Same image!
        envFrom:
        - configMapRef:
            name: app-config       # Different ConfigMap per namespace
```

**Deploy:**
```bash
# Development
kubectl apply -f dev-config.yaml
kubectl apply -f deployment.yaml -n development

# Production
kubectl apply -f prod-config.yaml
kubectl apply -f deployment.yaml -n production

# Same image, different configurations! ✓
```

### ConfigMap Update Behavior

**Important:** Changes to ConfigMap are NOT automatically reflected in running pods.

**Environment Variables:** NOT updated (need pod restart)
```bash
# Update ConfigMap
kubectl edit configmap app-config

# Pods won't see changes until restarted
kubectl rollout restart deployment webapp
```

**Mounted Volumes:** Updated automatically (eventually)
```yaml
# ConfigMap mounted as volume
volumeMounts:
- name: config
  mountPath: /etc/config

# Changes appear in /etc/config after ~60 seconds
# Application must watch for file changes
```

### ConfigMap Best Practices

1. **Use descriptive names**
   ```yaml
   ✓ app-config
   ✓ nginx-config
   ✓ database-config
   ✗ config
   ✗ cm1
   ```

2. **Version your ConfigMaps**
   ```yaml
   name: app-config-v1
   name: app-config-v2
   # Update deployment to use new version
   ```

3. **Use namespaces**
   ```
   development/app-config
   staging/app-config
   production/app-config
   ```

4. **Don't store secrets in ConfigMaps**
   ```yaml
   ✗ PASSWORD=mysecret
   ✓ Use Secrets instead
   ```

5. **Keep ConfigMaps small**
   ```
   Limit: 1MB per ConfigMap
   For large configs: Use volumes or external config servers
   ```

---

## 2. Secrets - Sensitive Data

### What is a Secret?

**Secret** stores sensitive information (passwords, tokens, keys).

**Similar to ConfigMap but:**
- Base64 encoded
- More access controls
- Can be encrypted at rest
- Limited to 1MB

### Why Use Secrets?

**Bad Approach:**
```yaml
env:
- name: DATABASE_PASSWORD
  value: "mypassword123"     # ❌ Visible in YAML, Git, logs
```

**Good Approach:**
```yaml
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password           # ✓ Stored securely
```

### Secret Types

```bash
# Opaque (default) - arbitrary key-value pairs
kubectl create secret generic my-secret --from-literal=key=value

# Docker registry - credentials for private registries
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass

# TLS - certificates and keys
kubectl create secret tls tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key

# Service Account Token
# (Automatically created by Kubernetes)

# SSH Auth
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa
```

### Creating Secrets

#### Method 1: From Literal Values

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret123
```

#### Method 2: From Files

**password.txt:**
```
supersecret123
```

```bash
kubectl create secret generic db-secret \
  --from-file=password=./password.txt
```

#### Method 3: Declarative YAML

**Manual Base64 Encoding:**
```bash
echo -n 'admin' | base64
# Output: YWRtaW4=

echo -n 'supersecret123' | base64
# Output: c3VwZXJzZWNyZXQxMjM=
```

**secret.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Base64 encoded values
  username: YWRtaW4=                    # admin
  password: c3VwZXJzZWNyZXQxMjM=        # supersecret123
```

**Alternative: Plain text (automatically encoded):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:              # Use stringData for plain text
  username: admin
  password: supersecret123
```

```bash
kubectl apply -f secret.yaml
```

### Viewing Secrets

```bash
# List secrets
kubectl get secrets

# Describe secret (doesn't show values)
kubectl describe secret db-secret

# View secret (shows base64 encoded values)
kubectl get secret db-secret -o yaml

# Decode secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode
# Output: supersecret123
```

### Using Secrets in Pods

#### Option 1: Environment Variables (Individual Keys)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

#### Option 2: All Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - secretRef:
        name: db-secret
```

#### Option 3: Mount as Volume (Files)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true           # Secrets should be read-only
  
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```

**Result inside container:**
```bash
ls /etc/secrets/
# Output:
# username
# password

cat /etc/secrets/username
# Output: admin

cat /etc/secrets/password
# Output: supersecret123
```

#### Option 4: Mount Specific Keys with Custom Paths

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/app
      readOnly: true
  
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      items:
      - key: username
        path: db/username.txt    # Custom path
        mode: 0400               # File permissions
      - key: password
        path: db/password.txt
        mode: 0400
```

### Docker Registry Secret

**For pulling images from private registries:**

```bash
# Create secret
kubectl create secret docker-registry myregistrykey \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# Use in pod
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: app
    image: myregistry.com/myapp:1.0    # Private image
  imagePullSecrets:
  - name: myregistrykey                 # Docker registry secret
```

### TLS Secret for Ingress

```bash
# Create TLS secret
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# Use in Ingress
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret        # TLS certificate
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### Complete Example: Database Connection

**db-secret.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: rootpass123
  database: myappdb
  username: appuser
  password: apppass123
```

**mysql-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
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
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
```

**app-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
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
      - name: app
        image: myapp:1.0
        env:
        - name: DB_HOST
          value: mysql.default.svc.cluster.local
        - name: DB_PORT
          value: "3306"
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: database
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
```

### Secret Best Practices

1. **Use RBAC to restrict access**
   ```yaml
   # Only specific ServiceAccounts can read secrets
   ```

2. **Enable encryption at rest**
   ```yaml
   # Configure API server to encrypt secrets in etcd
   ```

3. **Don't commit secrets to Git**
   ```bash
   # Use tools like:
   # - Sealed Secrets
   # - External Secrets Operator
   # - Vault
   ```

4. **Use separate secrets per environment**
   ```
   development/db-secret
   staging/db-secret
   production/db-secret
   ```

5. **Rotate secrets regularly**
   ```bash
   # Update secrets periodically
   # Restart pods to use new secrets
   ```

6. **Mount secrets as volumes (more secure)**
   ```yaml
   # Volumes are stored in tmpfs (RAM), not disk
   volumeMounts:
   - name: secret
     mountPath: /secrets
     readOnly: true
   ```

7. **Use external secret management**
   ```
   - HashiCorp Vault
   - AWS Secrets Manager
   - Azure Key Vault
   - Google Secret Manager
   ```

### Secret vs ConfigMap Comparison

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| **Purpose** | Non-sensitive configuration | Sensitive data |
| **Encoding** | Plain text | Base64 encoded |
| **Encryption** | No | Optional (at rest) |
| **Size Limit** | 1MB | 1MB |
| **RBAC** | Standard | Stricter |
| **Storage** | etcd | etcd (can be encrypted) |
| **Use Case** | App settings, config files | Passwords, tokens, keys |

---

## 3. Volumes - Data Persistence

### The Problem: Container Storage is Ephemeral

**Without Volumes:**
```
Container starts → Creates files → Container stops
                                          ↓
                                   Files are lost! ❌
```

**Example:**
```yaml
# Pod without volume
apiVersion: v1
kind: Pod
metadata:
  name: temp-pod
spec:
  containers:
  - name: app
    image: nginx
    # Any files written to /data are lost when pod restarts
```

### What is a Volume?

**Volume** = Directory accessible to containers in a pod

**Benefits:**
- Data persists across container restarts
- Shared data between containers in same pod
- Various storage backends supported

### Volume Lifecycle

```
Volume tied to Pod lifecycle:
Pod created → Volume mounted → Pod deleted → Volume deleted
                                                  ↓
                                     (Some volume types persist)
```

### Volume Types

#### 1. emptyDir - Temporary Storage

**Use case:** Scratch space, caching, temporary data sharing

**Characteristics:**
- Created when pod starts
- Deleted when pod is removed
- Shared between containers in pod
- Stored on node disk (or tmpfs for memory)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-data
spec:
  containers:
  # Writer container
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from writer" > /data/message.txt; sleep 3600']
    volumeMounts:
    - name: shared-storage
      mountPath: /data
  
  # Reader container
  - name: reader
    image: busybox
    command: ['sh', '-c', 'while true; do cat /data/message.txt; sleep 5; done']
    volumeMounts:
    - name: shared-storage
      mountPath: /data
  
  volumes:
  - name: shared-storage
    emptyDir: {}              # Empty directory
```

**emptyDir with tmpfs (RAM):**
```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory          # Store in RAM instead of disk
    sizeLimit: 1Gi          # Limit size
```

#### 2. hostPath - Node Storage

**Use case:** Access node filesystem (rare, use with caution)

**Characteristics:**
- Mounts directory/file from node
- Data persists beyond pod lifecycle
- Pod must run on same node to access data
- **Security risk** - gives access to node filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: node-logs
      mountPath: /host-logs
      readOnly: true
  
  volumes:
  - name: node-logs
    hostPath:
      path: /var/log           # Path on node
      type: Directory          # Must be a directory
```

**hostPath types:**
```yaml
type: Directory        # Must exist
type: DirectoryOrCreate # Create if doesn't exist
type: File             # Must be a file
type: FileOrCreate     # Create file if doesn't exist
```

**Warning:** Only use hostPath for:
- DaemonSets (monitoring/logging agents)
- Development/testing
- When you need node-level access

#### 3. configMap Volume

Already covered in ConfigMap section.

```yaml
volumes:
- name: config
  configMap:
    name: app-config
```

#### 4. secret Volume

Already covered in Secrets section.

```yaml
volumes:
- name: secrets
  secret:
    secretName: db-secret
```

#### 5. persistentVolumeClaim

**Most important for production!** Covered in next section.

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

#### 6. downwardAPI - Pod/Container Info

**Use case:** Expose pod metadata to container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do cat /etc/podinfo/*; sleep 5; done']
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "pod-name"
        fieldRef:
          fieldPath: metadata.name
      - path: "namespace"
        fieldRef:
          fieldPath: metadata.namespace
```

**Result inside container:**
```bash
cat /etc/podinfo/pod-name
# Output: downward-api

cat /etc/podinfo/namespace
# Output: default

cat /etc/podinfo/labels
# Output: app="myapp"
#         tier="frontend"
```

#### 7. projected - Combine Multiple Sources

```yaml
volumes:
- name: all-in-one
  projected:
    sources:
    - secret:
        name: db-secret
    - configMap:
        name: app-config
    - downwardAPI:
        items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
```

### Volume Mount Options

```yaml
volumeMounts:
- name: data
  mountPath: /app/data           # Where to mount
  subPath: app1                  # Mount only this subdirectory
  readOnly: true                 # Read-only mount
  mountPropagation: None         # Mount propagation (None, HostToContainer, Bidirectional)
```

**subPath example:**
```yaml
# One PVC, multiple apps using different subdirectories
volumes:
- name: shared-pvc
  persistentVolumeClaim:
    claimName: shared-storage

# App 1
volumeMounts:
- name: shared-pvc
  mountPath: /data
  subPath: app1                  # Mounts PVC/app1 to /data

# App 2
volumeMounts:
- name: shared-pvc
  mountPath: /data
  subPath: app2                  # Mounts PVC/app2 to /data
```

---

## 4. Persistent Volumes & Claims

### The Problem: Data Persistence Across Pods

**Scenario:**
```
Database pod stores data → Pod restarts → Data lost ❌
Database pod stores data → Pod moved to different node → Data lost ❌
```

**Solution: Persistent Volumes (PV) + Persistent Volume Claims (PVC)**

### Architecture

```
┌────────────────────────────────────────────────┐
│                 CLUSTER                        │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │  Storage (Admin creates)             │    │
│  │  - NFS Server                        │    │
│  │  - Cloud Storage (EBS, Azure Disk)   │    │
│  │  - Local Disks                       │    │
│  └───────────────┬──────────────────────┘    │
│                  │                             │
│  ┌───────────────▼──────────────────────┐    │
│  │  Persistent Volume (PV)              │    │
│  │  - Abstraction layer                 │    │
│  │  - Capacity: 10Gi                    │    │
│  │  - Access: ReadWriteOnce             │    │
│  └───────────────┬──────────────────────┘    │
│                  │ (binds to)                  │
│  ┌───────────────▼──────────────────────┐    │
│  │  Persistent Volume Claim (PVC)       │    │
│  │  - User requests storage             │    │
│  │  - Request: 5Gi                      │    │
│  └───────────────┬──────────────────────┘    │
│                  │                             │
│  ┌───────────────▼──────────────────────┐    │
│  │  Pod                                 │    │
│  │  - Uses PVC                          │    │
│  │  - Data persists                     │    │
│  └──────────────────────────────────────┘    │
└────────────────────────────────────────────────┘

PV = Actual storage (cluster resource)
PVC = Request for storage (namespaced resource)
```

### Persistent Volume (PV)

**PV** = Piece of storage in cluster (cluster-wide resource)

**Created by:** Cluster admin

**pv.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi                    # Size
  
  accessModes:
  - ReadWriteOnce                    # RWO, ROX, or RWX
  
  persistentVolumeReclaimPolicy: Retain  # Retain, Delete, or Recycle
  
  storageClassName: manual           # Storage class name
  
  # Storage backend (choose one)
  hostPath:
    path: /mnt/data                  # Local path (for testing)
  
  # Or NFS:
  # nfs:
  #   server: nfs-server.default.svc.cluster.local
  #   path: /exports
  
  # Or Cloud (AWS EBS example):
  # awsElasticBlockStore:
  #   volumeID: vol-0123456789
  #   fsType: ext4
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
```

### Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| **ReadWriteOnce** | RWO | Read-write by single node |
| **ReadOnlyMany** | ROX | Read-only by many nodes |
| **ReadWriteMany** | RWX | Read-write by many nodes |
| **ReadWriteOncePod** | RWOP | Read-write by single pod (K8s 1.22+) |

**Supported modes by storage type:**

| Storage Type | RWO | ROX | RWX |
|--------------|-----|-----|-----|
| AWS EBS | ✓ | - | - |
| Azure Disk | ✓ | - | - |
| GCE PD | ✓ | ✓ | - |
| NFS | ✓ | ✓ | ✓ |
| Local | ✓ | - | - |

### Reclaim Policy

**What happens to PV when PVC is deleted?**

1. **Retain** (default)
   - PV remains
   - Data preserved
   - Manual cleanup required
   - Use for production data

2. **Delete**
   - PV and storage deleted
   - Data lost
   - Use for temporary data

3. **Recycle** (deprecated)
   - Basic scrub (`rm -rf /volume/*`)
   - PV available for new claims

### Persistent Volume Claim (PVC)

**PVC** = Request for storage by user (namespaced resource)

**pvc.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce                    # Must match PV
  
  resources:
    requests:
      storage: 5Gi                   # Requested size
  
  storageClassName: manual           # Must match PV
  
  # Optional: Select specific PV
  # selector:
  #   matchLabels:
  #     type: fast
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

**PVC Status:**
```
Pending  → Bound → Terminating
   ↓         ↓          ↓
 Waiting  In Use   Being deleted
```

### Binding Process

```
1. User creates PVC requesting 5Gi

2. Kubernetes finds suitable PV:
   - Sufficient capacity (>= 5Gi)
   - Matching access mode
   - Matching storage class
   
3. PVC binds to PV (one-to-one)

4. PV status: Available → Bound
   PVC status: Pending → Bound
```

### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: password
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql      # MySQL data directory
  
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: pvc-example         # Reference PVC
```

**Deploy:**
```bash
# Create PV (admin)
kubectl apply -f pv.yaml

# Create PVC (user)
kubectl apply -f pvc.yaml

# Verify binding
kubectl get pv
kubectl get pvc

# Create pod using PVC
kubectl apply -f pod.yaml

# Verify
kubectl exec mysql-pod -- ls /var/lib/mysql
```

### Complete Example: MySQL with Persistent Storage

**1. pv.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    type: local
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/mysql-data
```

**2. pvc.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual
```

**3. deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
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
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

**Deploy:**
```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl create secret generic mysql-secret --from-literal=password=mypassword
kubectl apply -f deployment.yaml

# Test persistence
kubectl exec -it mysql-xxx -- mysql -u root -p
# Create database, insert data

# Delete pod
kubectl delete pod mysql-xxx

# New pod created automatically
# Data still there! ✓
```

### PV Lifecycle

```
┌─────────────────────────────────────────────┐
│              PV LIFECYCLE                   │
├─────────────────────────────────────────────┤
│                                             │
│  Available  →  Bound  →  Released  →  Failed│
│      ↓           ↓          ↓          ↓   │
│   Not used   In use   PVC deleted  Error   │
│                                             │
└─────────────────────────────────────────────┘

Available: Ready for binding
Bound: Bound to PVC
Released: PVC deleted, but PV not yet reclaimed
Failed: Automatic reclamation failed
```

### Commands

```bash
# PersistentVolumes (cluster-scoped)
kubectl get pv
kubectl describe pv <name>
kubectl delete pv <name>

# PersistentVolumeClaims (namespaced)
kubectl get pvc
kubectl describe pvc <name>
kubectl delete pvc <name>

# Check PVC usage
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumes[*].persistentVolumeClaim.claimName}{"\n"}{end}'

# Expand PVC (if StorageClass allows)
kubectl edit pvc <name>
# Increase spec.resources.requests.storage
```

---

## 5. Storage Classes

### What is a StorageClass?

**StorageClass** enables **dynamic provisioning** of PersistentVolumes.

**Problem with Static Provisioning:**
```
1. Admin creates PV manually
2. User creates PVC
3. PVC binds to PV
   ↓
Admin must create PV for every PVC request ❌
Not scalable ❌
```

**Solution: Dynamic Provisioning**
```
1. Admin creates StorageClass once
2. User creates PVC referencing StorageClass
3. Kubernetes automatically provisions PV
   ↓
Automatic! ✓
Scalable! ✓
```

### StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs     # Storage provider
parameters:
  type: gp3                            # Provider-specific params
  iopsPerGB: "10"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true             # Allow PVC resize
reclaimPolicy: Delete                  # Delete PV when PVC deleted
```

**Common Provisioners:**
- `kubernetes.io/aws-ebs` - AWS EBS
- `kubernetes.io/azure-disk` - Azure Disk
- `kubernetes.io/gce-pd` - Google Persistent Disk
- `kubernetes.io/no-provisioner` - Local volumes (no provisioning)
- External provisioners (NFS, Ceph, etc.)

### Using StorageClass

**storageclass.yaml:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
allowVolumeExpansion: true
```

**pvc-dynamic.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard         # Reference StorageClass
  resources:
    requests:
      storage: 10Gi
```

**Deploy:**
```bash
# No need to create PV manually!
kubectl apply -f storageclass.yaml
kubectl apply -f pvc-dynamic.yaml

# PV automatically created
kubectl get pv
kubectl get pvc
```

### Default StorageClass

```bash
# View StorageClasses
kubectl get storageclass
kubectl get sc

# Mark as default
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# PVC without storageClassName uses default
```

### Volume Binding Modes

**1. Immediate (default)**
```yaml
volumeBindingMode: Immediate
```
- PV provisioned as soon as PVC created
- May provision volume in wrong zone

**2. WaitForFirstConsumer**
```yaml
volumeBindingMode: WaitForFirstConsumer
```
- Waits for pod that uses PVC
- Provisions volume in same zone as pod
- **Recommended for topology-aware storage**

### Volume Expansion

```yaml
allowVolumeExpansion: true           # In StorageClass
```

**Expand PVC:**
```bash
kubectl edit pvc my-pvc
# Change:
# storage: 10Gi → 20Gi

# Restart pod to apply (some storage types)
kubectl rollout restart deployment myapp
```

### Multiple StorageClasses

```yaml
# Standard HDD storage
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2

# Fast SSD storage
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io2
  iopsPerGB: "50"

# Cheap slow storage
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: sc1
```

**Use different classes:**
```yaml
# Database - fast storage
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Gi

# Logs - slow storage
spec:
  storageClassName: slow
  resources:
    requests:
      storage: 1Ti
```

---

## 6. StatefulSets

### What is a StatefulSet?

**StatefulSet** manages deployment of stateful applications.

**Deployment vs StatefulSet:**

| Aspect | Deployment | StatefulSet |
|--------|------------|-------------|
| **Purpose** | Stateless apps | Stateful apps |
| **Pod Names** | Random (web-abc123) | Ordered (web-0, web-1) |
| **Pod Identity** | No stable identity | Stable identity |
| **Storage** | Shared or ephemeral | Dedicated per pod |
| **Scaling** | Any pod can be deleted | Ordered (last first) |
| **Startup** | Parallel | Sequential |
| **Network** | Load balanced | Stable DNS per pod |
| **Use Case** | Web apps, APIs | Databases, message queues |

### StatefulSet Features

1. **Stable Network Identity**
   ```
   Pod DNS: <pod-name>.<service-name>.<namespace>.svc.cluster.local
   
   mysql-0.mysql.default.svc.cluster.local
   mysql-1.mysql.default.svc.cluster.local
   mysql-2.mysql.default.svc.cluster.local
   ```

2. **Stable Storage**
   - Each pod gets its own PVC
   - PVC persists beyond pod lifecycle
   - Pod always reconnects to same PVC

3. **Ordered Operations**
   - **Startup**: 0, 1, 2... (sequential)
   - **Shutdown**: ...2, 1, 0 (reverse)
   - **Update**: One at a time, ordered

### StatefulSet YAML

**headless-service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None              # Headless service (no ClusterIP)
  selector:
    app: mysql
  ports:
  - port: 3306
```

**statefulset.yaml:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql           # Must match headless service
  replicas: 3
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
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  
  volumeClaimTemplates:        # PVC template
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

**Deploy:**
```bash
kubectl apply -f headless-service.yaml
kubectl apply -f statefulset.yaml

# Watch pods start sequentially
kubectl get pods -w
# mysql-0 → Running → mysql-1 starts → Running → mysql-2 starts

# View PVCs (one per pod)
kubectl get pvc
# data-mysql-0
# data-mysql-1
# data-mysql-2
```

### Accessing StatefulSet Pods

**Individual Pod DNS:**
```bash
# From inside cluster:
mysql -h mysql-0.mysql.default.svc.cluster.local -u root -p
mysql -h mysql-1.mysql.default.svc.cluster.local -u root -p
mysql -h mysql-2.mysql.default.svc.cluster.local -u root -p
```

**All Pods (via headless service):**
```bash
# Resolves to all pod IPs
nslookup mysql.default.svc.cluster.local
```

### StatefulSet Scaling

```bash
# Scale to 5 replicas
kubectl scale statefulset mysql --replicas=5

# Scale up: mysql-3, mysql-4 created sequentially
# Scale down: mysql-4, mysql-3 deleted sequentially

# Scale to 1
kubectl scale statefulset mysql --replicas=1
# mysql-2 deleted → mysql-1 deleted → Only mysql-0 remains
```

### Update Strategies

**1. RollingUpdate (default)**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0           # Update pods >= partition
```

- Updates pods in reverse order (largest ordinal first)
- One at a time

**2. OnDelete**
```yaml
spec:
  updateStrategy:
    type: OnDelete
```

- Pods updated only when manually deleted
- Manual control over updates

### Complete Example: MySQL Cluster

**mysql-configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  primary.cnf: |
    [mysqld]
    log-bin=mysql-bin
    server-id=1
  
  replica.cnf: |
    [mysqld]
    super-read-only
    server-id=2
```

**mysql-secret.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: rootpassword
  replication-password: replpassword
```

**mysql-service.yaml:**
```yaml
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306

---
# Regular service for reads
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
```

**mysql-statefulset.yaml:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      # Init container to determine if primary or replica
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate server-id from pod ordinal
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          
          # Copy config based on pod ordinal
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
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
        - name: conf
          mountPath: /etc/mysql/conf.d
        
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
      
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  
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

**Deploy:**
```bash
kubectl apply -f mysql-configmap.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml

# Watch deployment
kubectl get pods -w

# Connect to primary
kubectl exec -it mysql-0 -- mysql -u root -p

# Connect to replica
kubectl exec -it mysql-1 -- mysql -u root -p
```

---

## Summary - Part 3

**What You Learned:**

1. **ConfigMaps**
   - Store non-sensitive configuration
   - Use as environment variables or files
   - Separate config from code

2. **Secrets**
   - Store sensitive data securely
   - Base64 encoded
   - Types: Opaque, TLS, Docker registry
   - Mount as volumes or env vars

3. **Volumes**
   - Types: emptyDir, hostPath, configMap, secret, PVC
   - Share data between containers
   - Temporary vs persistent storage

4. **Persistent Volumes & Claims**
   - PV: Cluster storage resource
   - PVC: Storage request by user
   - Access modes: RWO, ROX, RWX
   - Reclaim policies: Retain, Delete

5. **Storage Classes**
   - Dynamic provisioning
   - Provider-specific parameters
   - Multiple classes for different tiers

6. **StatefulSets**
   - For stateful applications
   - Stable network identity
   - Ordered operations
   - Persistent storage per pod

**Next:**
- **Part 4**: Networking, Ingress, Network Policies, DNS
- **Part 5**: Security, RBAC, Best Practices, Interview Questions

---

**Continue to Part 4!** 🚀
