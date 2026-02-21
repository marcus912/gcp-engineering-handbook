# Chapter 20: Kubernetes Security & Access Control

**Prerequisites:** Chapter 1 (IAM & Identity), Chapter 12 (Containers & Kubernetes Fundamentals)

Kubernetes security is a multi-layered discipline that requires defense in depth across cloud infrastructure, cluster configuration, container runtime, and application code. This chapter covers the essential security controls, best practices, and tools for securing production Kubernetes environments on GKE.

---

## 19.1 Kubernetes Security Model

### The 4 C's of Cloud Native Security

Kubernetes security follows a layered defense-in-depth model known as the "4 C's":

```
┌─────────────────────────────────────────────────────────────┐
│                           CODE                              │
│  Application vulnerabilities, secrets in code, dependencies │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                    CONTAINER                          │ │
│  │  Image vulnerabilities, container config, registries  │ │
│  │                                                       │ │
│  │  ┌─────────────────────────────────────────────────┐ │ │
│  │  │                CLUSTER                          │ │ │
│  │  │  RBAC, network policies, pod security, secrets  │ │ │
│  │  │                                                 │ │ │
│  │  │  ┌───────────────────────────────────────────┐ │ │ │
│  │  │  │            CLOUD                          │ │ │ │
│  │  │  │  Network security, IAM, infrastructure    │ │ │ │
│  │  │  │                                           │ │ │ │
│  │  │  └───────────────────────────────────────────┘ │ │ │
│  │  │                                                 │ │ │
│  │  └─────────────────────────────────────────────────┘ │ │
│  │                                                       │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Each layer provides security controls that complement the others.
If one layer is compromised, other layers provide defense.
```

### Kubernetes Attack Surface

**Control Plane Attack Vectors:**
- API Server exposure (authentication bypass, authorization flaws)
- etcd access (contains all cluster secrets unencrypted by default)
- Controller manager and scheduler compromise
- Admission webhook manipulation

**Data Plane Attack Vectors:**
- Container escapes (kernel exploits, misconfigurations)
- Privileged containers with host access
- Stolen service account tokens
- Lateral movement via network policies gaps
- Supply chain attacks (compromised images)

**GKE-Specific Considerations:**
- Metadata server access from pods (169.254.169.254)
- Workload Identity misconfiguration
- GKE node SSH access
- Cloud IAM integration points

### Defense in Depth Strategy

| Layer | Security Controls | GKE Features |
|-------|------------------|--------------|
| **Cloud** | VPC isolation, firewall rules, private clusters, IAM | Private GKE, VPC-native clusters, Binary Authorization |
| **Cluster** | RBAC, network policies, admission control, audit logs | GKE RBAC, Dataplane V2, Policy Controller |
| **Container** | Image scanning, signing, runtime policies, read-only FS | Container Analysis, Binary Authorization, Shielded GKE Nodes |
| **Code** | Dependency scanning, secrets management, least privilege | Artifact Analysis, Secret Manager integration, Workload Identity |

### Threat Modeling Example

```
Threat: Compromised application container
├── Initial Access: Exploit in web application dependency
├── Privilege Escalation: Attempt to access node resources
│   └── Mitigation: Pod Security Standards (Restricted)
├── Credential Access: Read service account token
│   └── Mitigation: Bound service account tokens, short TTL
├── Lateral Movement: Scan internal network
│   └── Mitigation: Network policies block unauthorized egress
├── Collection: Access Kubernetes Secrets
│   └── Mitigation: RBAC limits secret access, encryption at rest
└── Impact: Cryptomining or data exfiltration
    └── Mitigation: Runtime security (Falco), resource limits
```

---

## 19.2 Authentication & Authorization

### Authentication Methods

**1. User Authentication (Human Access)**

GKE uses Google Cloud IAM for user authentication:

```bash
# Configure kubectl to use GKE cluster
gcloud container clusters get-credentials prod-cluster \
  --region us-central1 \
  --project my-project

# This configures kubectl to use gcloud as authentication provider
kubectl config view --minify

# Authentication flow:
# kubectl → gcloud auth → Google OAuth → GKE API Server
```

**2. Service Account Authentication (Pod Access)**

Kubernetes service accounts provide identity for pods:

```yaml
# Traditional service account token (legacy, no expiration)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
automountServiceAccountToken: true
```

**3. Projected Service Account Tokens (Recommended)**

Bound tokens with expiration and audience binding:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  serviceAccountName: app-service-account
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: sa-token
  volumes:
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600  # 1 hour
          audience: https://my-api.example.com
```

**4. OIDC Authentication**

For integrating external identity providers:

```bash
# Configure API server for OIDC (managed by GKE)
# User side: configure kubectl with OIDC plugin
kubectl config set-credentials oidc-user \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://accounts.google.com \
  --exec-arg=--oidc-client-id=XXXXX \
  --exec-arg=--oidc-client-secret=YYYYY
```

### RBAC (Role-Based Access Control)

**RBAC Components:**
- **Role/ClusterRole**: Define permissions (verbs on resources)
- **RoleBinding/ClusterRoleBinding**: Grant permissions to subjects
- **Subjects**: Users, groups, or service accounts

**Role Example (Namespace-scoped):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]  # "" indicates core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

**ClusterRole Example (Cluster-wide):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  # Optionally restrict to specific secret names
  resourceNames: ["db-password", "api-key"]
```

**RoleBinding Example:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
- kind: User
  name: jane@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding Example:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-all-namespaces
subjects:
- kind: Group
  name: sre-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view  # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Best Practices: Least Privilege

**1. Namespace-specific Service Accounts:**

```yaml
# Bad: Using default service account
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  # Uses default service account (has no explicit permissions, but still a token)
  containers:
  - name: app
    image: gcr.io/my-project/app:v1

---
# Good: Dedicated service account with minimal permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
automountServiceAccountToken: false  # Disable if not needed

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["app-config"]  # Only specific ConfigMap

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: true  # Enable only when needed
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
```

**2. Avoid Wildcard Permissions:**

```yaml
# Bad: Wildcard permissions
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Good: Specific permissions
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get"]
```

**3. Dangerous Permissions to Avoid:**

```yaml
# DANGEROUS: Can escalate privileges
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings"]
  verbs: ["create", "update", "patch", "delete", "bind", "escalate"]

# DANGEROUS: Can read all secrets
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["*"]

# DANGEROUS: Can create privileged pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create"]
  # Without Pod Security Standards enforcement!

# DANGEROUS: Can exec into any pod
rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

### RBAC Auditing and Testing

```bash
# Check if a user/SA can perform an action (dry-run)
kubectl auth can-i get pods \
  --namespace production \
  --as system:serviceaccount:production:app-sa

# Check all permissions for a service account
kubectl auth can-i --list \
  --namespace production \
  --as system:serviceaccount:production:app-sa

# Audit who has access to secrets in a namespace
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces \
  -o json | jq -r '
    .items[] |
    select(.roleRef.kind == "Role" or .roleRef.kind == "ClusterRole") |
    select(.metadata.namespace == "production" or .metadata.namespace == null) |
    "\(.kind): \(.metadata.name) -> \(.roleRef.name)"
  '

# Find overly permissive roles
kubectl get clusterroles -o json | jq -r '
  .items[] |
  select(.rules[]? | select(.verbs[]? == "*" and .resources[]? == "*")) |
  .metadata.name
'
```

### Impersonation for Testing

```bash
# Test as another user (requires impersonation permissions)
kubectl get pods --as=jane@example.com

# Test as a service account
kubectl get secrets \
  --as=system:serviceaccount:production:app-sa \
  --namespace=production

# Test as a group
kubectl get nodes --as=user --as-group=developers
```

**Impersonation Role:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
```

---

## 19.3 Pod Security Standards & Admission

### Pod Security Standards Overview

Kubernetes defines three standard security profiles:

| Profile | Use Case | Key Restrictions |
|---------|----------|------------------|
| **Privileged** | System workloads, CNI plugins | No restrictions, full host access |
| **Baseline** | General workloads, defense in depth | Prevents known privilege escalations |
| **Restricted** | Security-critical applications | Heavily restricted, follows hardening best practices |

### Profile Comparison

```yaml
# PRIVILEGED Profile (no restrictions)
---
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  hostNetwork: true          # Allowed
  hostPID: true              # Allowed
  hostIPC: true              # Allowed
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      privileged: true       # Allowed
      capabilities:
        add: ["ALL"]         # Allowed
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /                # Allowed

---
# BASELINE Profile (prevents known escalations)
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
spec:
  # hostNetwork: true        # BLOCKED
  # hostPID: true            # BLOCKED
  # hostIPC: true            # BLOCKED
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      # privileged: true     # BLOCKED
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]  # Limited set allowed
      allowPrivilegeEscalation: false
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: app-config
  # hostPath volumes         # BLOCKED (most types)

---
# RESTRICTED Profile (hardened, production-ready)
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true        # REQUIRED
    runAsUser: 1000           # REQUIRED (non-root UID)
    fsGroup: 2000             # Set for volume permissions
    seccompProfile:
      type: RuntimeDefault    # REQUIRED
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      runAsNonRoot: true      # REQUIRED
      runAsUser: 1000         # REQUIRED
      allowPrivilegeEscalation: false  # REQUIRED
      readOnlyRootFilesystem: true     # Best practice
      capabilities:
        drop: ["ALL"]         # REQUIRED
      seccompProfile:
        type: RuntimeDefault  # REQUIRED (or pod-level)
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}              # Only approved volume types
  - name: cache
    emptyDir: {}
```

### Pod Security Admission

Pod Security Admission (PSA) replaced PodSecurityPolicy in Kubernetes 1.25. It enforces Pod Security Standards at the namespace level.

**PSA Modes:**
- `enforce`: Reject non-compliant pods
- `audit`: Allow but log violations to audit log
- `warn`: Allow but return warning to user

**Namespace Labels:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # Audit baseline violations
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: latest

    # Warn on baseline violations
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest

---
# Development namespace with baseline enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest

---
# System namespace with privileged access
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
```

**Apply PSA to existing namespace:**

```bash
# Enforce restricted standard
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Add audit mode
kubectl label namespace production \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest

# Check current PSA labels
kubectl get namespace production -o yaml | grep pod-security
```

### Common SecurityContext Fields

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  # Pod-level security context
  securityContext:
    runAsUser: 1000           # UID to run as
    runAsGroup: 3000          # Primary GID
    fsGroup: 2000             # Volume ownership GID
    runAsNonRoot: true        # Enforce non-root
    supplementalGroups: [4000, 5000]  # Additional GIDs
    seccompProfile:
      type: RuntimeDefault    # Seccomp profile
    seLinuxOptions:           # SELinux (if enabled)
      level: "s0:c123,c456"

  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    # Container-level security context (overrides pod-level)
    securityContext:
      runAsUser: 1001         # Override pod UID
      runAsNonRoot: true
      readOnlyRootFilesystem: true  # Immutable root FS
      allowPrivilegeEscalation: false
      privileged: false       # Not privileged
      capabilities:
        drop:
        - ALL                 # Drop all capabilities
        add:
        - NET_BIND_SERVICE    # Add specific capability
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/audit.json

    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: app-data
      mountPath: /data

    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
      requests:
        memory: "256Mi"
        cpu: "250m"

  volumes:
  - name: tmp
    emptyDir: {}
  - name: app-data
    emptyDir: {}
```

### Common PSA Violations and Fixes

**1. Running as root:**

```yaml
# ❌ Violation
spec:
  containers:
  - name: app
    image: nginx:latest  # Runs as root by default

# ✅ Fix
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:latest  # Non-root image
    securityContext:
      runAsNonRoot: true
      runAsUser: 101  # nginx user
```

**2. Missing seccomp profile:**

```yaml
# ❌ Violation
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    # No seccomp profile

# ✅ Fix
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
```

**3. Privilege escalation allowed:**

```yaml
# ❌ Violation
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      allowPrivilegeEscalation: true  # Or not set

# ✅ Fix
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      allowPrivilegeEscalation: false
```

**4. Capabilities not dropped:**

```yaml
# ❌ Violation
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    # No capabilities dropped

# ✅ Fix
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      capabilities:
        drop: ["ALL"]
```

**5. Writable root filesystem:**

```yaml
# ❌ Not enforced by restricted, but best practice
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    # Root FS is writable

# ✅ Best practice
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

---

## 19.4 Secrets Management

### Kubernetes Secrets Basics

Kubernetes Secrets are base64-encoded (NOT encrypted) by default:

```bash
# Create a secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --namespace production

# View secret (base64 encoded)
kubectl get secret db-credentials -o yaml

# Decode secret
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

**Secret YAML:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YWRtaW4=           # base64("admin")
  password: c3VwZXJzZWNyZXQ=   # base64("supersecret")
---
# Or use stringData for plain text (automatically encoded)
apiVersion: v1
kind: Secret
metadata:
  name: api-key
  namespace: production
type: Opaque
stringData:
  key: "my-secret-api-key"  # Stored as base64 automatically
```

**Using Secrets in Pods:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    env:
    # Environment variable from secret
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    # Mount secret as file
    - name: api-key-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: api-key-volume
    secret:
      secretName: api-key
      defaultMode: 0400  # Read-only for owner
```

### Encryption at Rest

By default, Secrets are stored **unencrypted** in etcd. Enable encryption:

```yaml
# EncryptionConfiguration (managed by GKE when enabled)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    - configmaps  # Optional
    providers:
    # First provider is used for encryption
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
    - identity: {}  # Fallback for reading old unencrypted data
```

**Enable in GKE:**

```bash
# GKE encrypts secrets at rest with Google-managed keys by default
# For customer-managed encryption keys (CMEK):
gcloud container clusters create secure-cluster \
  --region us-central1 \
  --database-encryption-key projects/PROJECT/locations/REGION/keyRings/RING/cryptoKeys/KEY \
  --enable-application-layer-secrets-encryption

# Or update existing cluster
gcloud container clusters update prod-cluster \
  --region us-central1 \
  --database-encryption-key projects/PROJECT/locations/REGION/keyRings/RING/cryptoKeys/KEY
```

### External Secrets Integration

**1. GCP Secret Manager with Workload Identity:**

```yaml
# Create Secret Manager secret
apiVersion: v1
kind: Secret
metadata:
  name: gcp-secret-ref
  namespace: production
  annotations:
    # Reference to GCP Secret Manager
    cloud.google.com/secret: "db-password"
type: Opaque
```

**Using GCP Secret Manager CSI Driver:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: app-sa@PROJECT.iam.gserviceaccount.com

---
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    volumeMounts:
    - name: secrets
      mountPath: /secrets
      readOnly: true
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: gcp-secrets
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
  namespace: production
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT/secrets/db-password/versions/latest"
        path: "db-password"
  secretObjects:
  - secretName: db-credentials
    type: Opaque
    data:
    - objectName: "db-password"
      key: "password"
```

**2. External Secrets Operator:**

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace
```

```yaml
# SecretStore for GCP Secret Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secret-store
  namespace: production
spec:
  provider:
    gcpsm:
      projectID: "my-project"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: prod-cluster
          serviceAccountRef:
            name: external-secrets-sa

---
# ExternalSecret that syncs from GCP Secret Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: SecretStore
  target:
    name: db-credentials  # K8s Secret to create
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: db-username  # GCP Secret Manager secret name
  - secretKey: password
    remoteRef:
      key: db-password
      version: latest  # Or specific version
```

**3. HashiCorp Vault Integration:**

```yaml
# Vault Agent Injector annotations
apiVersion: v1
kind: Pod
metadata:
  name: app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "app-role"
    vault.hashicorp.com/agent-inject-secret-database: "secret/data/database/config"
    vault.hashicorp.com/agent-inject-template-database: |
      {{- with secret "secret/data/database/config" -}}
      export DB_USERNAME="{{ .Data.data.username }}"
      export DB_PASSWORD="{{ .Data.data.password }}"
      {{- end }}
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    command: ["/bin/sh"]
    args:
    - -c
    - source /vault/secrets/database && ./app
```

**4. Sealed Secrets (GitOps-friendly):**

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal

# Create sealed secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Safe to commit to git
git add sealed-secret.yaml
```

```yaml
# SealedSecret (encrypted, safe for git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    username: AgBhY2D...  # Encrypted with controller's public key
    password: AgCKj3m...
  template:
    metadata:
      name: db-credentials
    type: Opaque
# Controller decrypts and creates regular Secret
```

### Secrets Management Comparison

| Solution | Pros | Cons | Best For |
|----------|------|------|----------|
| **K8s Secrets (encrypted at rest)** | Native, simple, no external deps | Limited features, rotation manual | Small deployments, dev environments |
| **GCP Secret Manager + CSI** | GCP native, automatic rotation, audit logs | GCP-specific, requires CSI driver | GKE deployments, GCP-centric orgs |
| **External Secrets Operator** | Multi-cloud, GitOps-friendly, auto-sync | Extra component to manage | Multi-cloud, hybrid environments |
| **HashiCorp Vault** | Enterprise features, dynamic secrets, rich ACLs | Complex setup, operational overhead | Large orgs, advanced secret workflows |
| **Sealed Secrets** | GitOps-friendly, simple | One-way encryption, no dynamic rotation | GitOps workflows, declarative deploys |

---

## 19.5 Image Security & Supply Chain

### Container Image Scanning

**1. Trivy Scanning:**

```bash
# Install Trivy
brew install aquasecurity/trivy/trivy

# Scan local image
trivy image gcr.io/my-project/app:v1

# Scan with severity filter
trivy image --severity HIGH,CRITICAL gcr.io/my-project/app:v1

# Scan and exit on vulnerabilities
trivy image --exit-code 1 --severity CRITICAL gcr.io/my-project/app:v1

# Scan Dockerfile for misconfigurations
trivy config Dockerfile

# Generate JSON report
trivy image -f json -o report.json gcr.io/my-project/app:v1

# Scan Kubernetes manifests
trivy k8s --report summary deployment.yaml
```

**Trivy in CI/CD:**

```yaml
# GitHub Actions example
name: Container Scan
on: [push]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Build image
      run: docker build -t gcr.io/my-project/app:${{ github.sha }} .
    - name: Run Trivy scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: gcr.io/my-project/app:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'  # Fail on vulnerabilities
    - name: Upload results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

**2. GCP Container Analysis:**

```bash
# Enable Container Scanning API
gcloud services enable containerscanning.googleapis.com

# Container Analysis automatically scans images pushed to GCR/Artifact Registry
docker push gcr.io/my-project/app:v1

# View vulnerabilities
gcloud container images list-tags gcr.io/my-project/app --format=json
gcloud container images describe gcr.io/my-project/app:v1 \
  --format="value(image_summary.vulnerability_summary)"

# Get detailed vulnerability report
gcloud container images describe gcr.io/my-project/app:v1 \
  --format=json | jq '.image_summary.vulnerability_counts'
```

### Image Signing and Verification

**Sigstore Cosign:**

```bash
# Install Cosign
brew install cosign

# Generate key pair (for testing; use KMS in production)
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key gcr.io/my-project/app:v1

# Sign with keyless mode (OIDC-based, no key management)
cosign sign gcr.io/my-project/app:v1

# Verify signature
cosign verify --key cosign.pub gcr.io/my-project/app:v1

# Verify with keyless mode
cosign verify \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com \
  gcr.io/my-project/app:v1

# Sign with attestations (SBOM, vulnerability scan)
cosign attest --key cosign.key \
  --predicate sbom.json \
  --type spdxjson \
  gcr.io/my-project/app:v1

# Verify attestation
cosign verify-attestation \
  --key cosign.pub \
  --type spdxjson \
  gcr.io/my-project/app:v1
```

**GKE Binary Authorization:**

```bash
# Enable Binary Authorization API
gcloud services enable binaryauthorization.googleapis.com

# Create attestor (verifies signatures)
gcloud container binauthz attestors create prod-attestor \
  --attestation-authority-note=projects/PROJECT/notes/prod-note \
  --attestation-authority-note-project=PROJECT

# Add public key to attestor
gcloud container binauthz attestors public-keys add \
  --attestor=prod-attestor \
  --keyversion=1 \
  --keyversion-key=cosign-key \
  --keyversion-keyring=cosign-keyring \
  --keyversion-location=us-central1 \
  --keyversion-project=PROJECT

# Create policy
cat > policy.yaml <<EOF
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
  - projects/PROJECT/attestors/prod-attestor
clusterAdmissionRules:
  us-central1.prod-cluster:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
    - projects/PROJECT/attestors/prod-attestor
EOF

# Apply policy
gcloud container binauthz policy import policy.yaml

# Create attestation
gcloud container binauthz attestations sign-and-create \
  --artifact-url=gcr.io/my-project/app@sha256:abc123... \
  --attestor=prod-attestor \
  --attestor-project=PROJECT \
  --keyversion=1 \
  --keyversion-key=cosign-key \
  --keyversion-keyring=cosign-keyring \
  --keyversion-location=us-central1 \
  --keyversion-project=PROJECT
```

**Policy Controller (Gatekeeper) for Image Policies:**

```yaml
# ConstraintTemplate for image verification
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsignature
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSignature
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedSigners:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredsignature

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not signed_by_allowed_signer(container.image)
        msg := sprintf("Image %v is not signed by allowed signer", [container.image])
      }

      signed_by_allowed_signer(image) {
        # Integration with cosign verification
        # Simplified for example
        allowed_signer := input.parameters.allowedSigners[_]
        contains(image, allowed_signer)
      }

---
# Constraint requiring signed images
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSignature
metadata:
  name: require-signed-images
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    allowedSigners:
    - "gcr.io/my-project"
```

### Base Image Selection

```dockerfile
# ❌ Avoid: Large, vulnerable base images
FROM ubuntu:latest
RUN apt-get update && apt-get install -y python3 python3-pip
COPY . /app
WORKDIR /app
RUN pip3 install -r requirements.txt
CMD ["python3", "app.py"]

# ✅ Better: Official slim images
FROM python:3.11-slim
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app
WORKDIR /app
CMD ["python", "app.py"]

# ✅ Best: Distroless images (minimal attack surface)
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

FROM gcr.io/distroless/python3
COPY --from=builder /app/deps /app/deps
COPY . /app
WORKDIR /app
ENV PYTHONPATH=/app/deps
CMD ["app.py"]

# ✅ Alternative: Alpine (small, but musl libc compatibility issues)
FROM python:3.11-alpine
RUN apk add --no-cache gcc musl-dev
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app
WORKDIR /app
CMD ["python", "app.py"]
```

### Image Digest Pinning

```yaml
# ❌ Avoid: Mutable tags
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: gcr.io/my-project/app:latest  # Tag can change!

# ✅ Good: Version tags
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: gcr.io/my-project/app:v1.2.3

# ✅ Best: Digest pinning (immutable)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: gcr.io/my-project/app@sha256:abc123def456...
        # Even if tag changes, this specific image is used
```

```bash
# Get image digest
docker inspect gcr.io/my-project/app:v1.2.3 \
  --format='{{index .RepoDigests 0}}'

# Or use crane
crane digest gcr.io/my-project/app:v1.2.3

# Pull by digest
docker pull gcr.io/my-project/app@sha256:abc123...
```

---

## 19.6 Runtime Security

### Seccomp Profiles

Seccomp (Secure Computing Mode) restricts system calls a container can make.

**Using RuntimeDefault profile:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-demo
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # Recommended default
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
```

**Custom Seccomp profile:**

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "accept4",
        "access",
        "arch_prctl",
        "bind",
        "brk",
        "clone",
        "close",
        "connect",
        "dup",
        "dup2",
        "epoll_create1",
        "epoll_ctl",
        "epoll_wait",
        "exit",
        "exit_group",
        "fcntl",
        "fstat",
        "futex",
        "getcwd",
        "getpid",
        "getrlimit",
        "getsockname",
        "getsockopt",
        "listen",
        "mmap",
        "mprotect",
        "munmap",
        "open",
        "openat",
        "read",
        "readlink",
        "rt_sigaction",
        "rt_sigprocmask",
        "sched_getaffinity",
        "set_robust_list",
        "setsockopt",
        "socket",
        "stat",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-seccomp
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
```

**Generate seccomp profile from audit logs:**

```bash
# Run with audit profile to log syscalls
# Use tools like oci-seccomp-bpf-hook or inspektor-gadget
# Then create allowlist from logs
```

### AppArmor Profiles

AppArmor provides mandatory access control (MAC) for files, capabilities, and network.

```yaml
# Load AppArmor profile on nodes first
# /etc/apparmor.d/k8s-deny-write
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  # Deny write to specific paths
  deny /etc/** w,
  deny /usr/** w,
  deny /bin/** w,
  deny /sbin/** w,

  # Allow writes to specific directories
  /tmp/** rw,
  /var/tmp/** rw,
}
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-demo
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
  - name: app
    image: gcr.io/my-project/app:v1
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### Falco Runtime Security

Falco detects anomalous behavior using eBPF or kernel modules.

**Install Falco:**

```bash
# Install via Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco-system \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falco.grpc.enabled=true \
  --set falco.grpcOutput.enabled=true

# Install falcosidekick for alerting
helm install falcosidekick falcosecurity/falcosidekick \
  --namespace falco-system \
  --set config.slack.webhookurl=https://hooks.slack.com/...
```

**Falco Rules:**

```yaml
# Custom Falco rules ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
  namespace: falco-system
data:
  custom-rules.yaml: |
    - rule: Unauthorized Process in Container
      desc: Detect execution of unauthorized processes
      condition: >
        spawned_process and container and
        not proc.name in (node, nginx, python, java)
      output: >
        Unauthorized process started
        (user=%user.name command=%proc.cmdline container=%container.name image=%container.image.repository)
      priority: WARNING
      tags: [process, container]

    - rule: Read Sensitive File
      desc: Detect reads of sensitive files
      condition: >
        open_read and container and
        fd.name in (/etc/shadow, /etc/passwd, /etc/sudoers)
      output: >
        Sensitive file opened for reading
        (user=%user.name file=%fd.name container=%container.name)
      priority: WARNING
      tags: [filesystem, container]

    - rule: Outbound Connection to Suspicious IP
      desc: Detect connections to known malicious IPs
      condition: >
        outbound and container and
        fd.sip in (suspicious_ips)
      output: >
        Suspicious outbound connection
        (user=%user.name ip=%fd.sip port=%fd.sport container=%container.name)
      priority: CRITICAL
      tags: [network, container]

    - rule: Container Drift Detection
      desc: Detect binary execution not in original container image
      condition: >
        spawned_process and container and
        proc.is_exe_from_memfd=true
      output: >
        Container drift detected - executable not in image
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: ERROR
      tags: [process, container]

    - rule: Kubernetes Client Tool in Container
      desc: Detect kubectl/kubelet usage in containers
      condition: >
        spawned_process and container and
        proc.name in (kubectl, kubelet)
      output: >
        Kubernetes client tool executed in container
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: WARNING
      tags: [process, kubernetes]

    - macro: suspicious_ips
      items: [192.0.2.1, 198.51.100.1]
```

**View Falco Alerts:**

```bash
# View Falco logs
kubectl logs -n falco-system -l app.kubernetes.io/name=falco

# Stream alerts
kubectl logs -n falco-system -l app.kubernetes.io/name=falco -f

# Example alert:
# {
#   "output": "Unauthorized process started (user=root command=nc -l 8080 container=app image=gcr.io/my-project/app)",
#   "priority": "Warning",
#   "rule": "Unauthorized Process in Container",
#   "time": "2026-02-18T10:15:30Z",
#   "output_fields": {
#     "container.name": "app",
#     "proc.cmdline": "nc -l 8080",
#     "user.name": "root"
#   }
# }
```

### Cilium Tetragon

Tetragon provides eBPF-based security observability and enforcement.

```bash
# Install Tetragon
helm repo add cilium https://helm.cilium.io
helm install tetragon cilium/tetragon \
  --namespace kube-system \
  --set tetragon.enabled=true
```

**Tetragon TracingPolicy:**

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: file-access-monitoring
spec:
  kprobes:
  - call: "security_file_open"
    return: true
    syscall: false
    args:
    - index: 0
      type: "file"
    returnArg:
      index: 0
      type: "int"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Prefix"
        values:
        - "/etc/shadow"
        - "/etc/passwd"
      matchActions:
      - action: Post
        rateLimit: 1m
      - action: Alert

---
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: network-monitoring
spec:
  kprobes:
  - call: "tcp_connect"
    syscall: false
    args:
    - index: 0
      type: "sock"
    selectors:
    - matchArgs:
      - index: 0
        operator: "DAddr"
        values:
        - "192.0.2.0/24"  # Suspicious network
      matchActions:
      - action: Alert
```

### Runtime Security Tools Comparison

| Tool | Detection Method | Strengths | Limitations | Use Case |
|------|-----------------|-----------|-------------|----------|
| **Seccomp** | Syscall filtering | Low overhead, kernel-level | Static allow/deny lists | Restrict container syscalls |
| **AppArmor** | MAC for files/caps | File-level controls | Requires profiles on nodes | Enforce file access policies |
| **Falco** | eBPF/kernel module | Rich rule language, real-time | Learning curve, alert fatigue | Runtime threat detection |
| **Tetragon** | eBPF | Low overhead, kernel visibility | Newer, smaller ecosystem | Advanced observability & enforcement |
| **SELinux** | MAC (labels) | Fine-grained, proven | Complex policy language | High-security environments |

---

## 19.7 Audit Logging

### Kubernetes Audit Policy

Audit logging records API server requests for security and compliance.

**Audit Levels:**
- `None`: Don't log
- `Metadata`: Log request metadata (user, timestamp, resource) but not request/response bodies
- `Request`: Log metadata and request body (not response)
- `RequestResponse`: Log metadata, request, and response bodies (verbose)

**Audit Policy Example:**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Don't log read-only requests to system resources
- level: None
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["events"]

# Don't log requests to certain non-resource paths
- level: None
  nonResourceURLs:
  - /healthz*
  - /version
  - /swagger*

# Log metadata for read operations on secrets
- level: Metadata
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["secrets"]

# Log request body for secret modifications
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]
  omitStages:
  - RequestReceived

# Log full request/response for RBAC changes
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

# Log exec/attach/portforward (interactive sessions)
- level: Metadata
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]

# Log authentication failures
- level: Metadata
  omitStages:
  - RequestReceived
  userGroups:
  - system:unauthenticated

# Catch-all: log metadata for everything else
- level: Metadata
  omitStages:
  - RequestReceived
```

**GKE Audit Logging:**

```bash
# Enable audit logs in GKE (via Cloud Logging)
gcloud container clusters update prod-cluster \
  --region us-central1 \
  --enable-cloud-logging \
  --logging=SYSTEM,WORKLOAD

# Audit logs are automatically sent to Cloud Logging
# View in console: Logging > Logs Explorer
# Filter: resource.type="k8s_cluster"
```

### Key Events to Audit

**Security-Critical Events:**

```yaml
# 1. Secret access
- level: RequestResponse
  verbs: ["get", "list", "create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]

# 2. RBAC modifications
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"

# 3. Privileged pod creation
- level: RequestResponse
  verbs: ["create", "update", "patch"]
  resources:
  - group: ""
    resources: ["pods"]
  # Filter for privileged in admission controller

# 4. ConfigMap changes (may contain sensitive data)
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["configmaps"]

# 5. Service account token requests
- level: Metadata
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["serviceaccounts/token"]

# 6. Node modifications
- level: RequestResponse
  resources:
  - group: ""
    resources: ["nodes"]

# 7. Admission webhook configurations
- level: RequestResponse
  resources:
  - group: "admissionregistration.k8s.io"

# 8. Custom resource definitions
- level: RequestResponse
  resources:
  - group: "apiextensions.k8s.io"
    resources: ["customresourcedefinitions"]
```

### Incident Investigation Patterns

**1. Unauthorized Secret Access:**

```bash
# Query Cloud Logging for secret access
gcloud logging read '
  resource.type="k8s_cluster"
  AND protoPayload.resourceName=~"secrets"
  AND protoPayload.authenticationInfo.principalEmail!="system:serviceaccount:kube-system:*"
  AND timestamp>="2026-02-18T00:00:00Z"
' --limit 50 --format json

# Look for:
# - Unexpected users accessing secrets
# - Service accounts from wrong namespaces
# - Failed authentication attempts
```

**2. RBAC Privilege Escalation:**

```bash
# Find RBAC changes
gcloud logging read '
  resource.type="k8s_cluster"
  AND protoPayload.resourceName=~"clusterrolebindings"
  AND protoPayload.methodName=~"create|update|patch"
  AND timestamp>="2026-02-18T00:00:00Z"
' --format json

# Check for:
# - New cluster-admin bindings
# - Wildcard permissions added
# - Service accounts granted admin rights
```

**3. Suspicious Pod Execution:**

```bash
# Find exec/attach events
gcloud logging read '
  resource.type="k8s_cluster"
  AND protoPayload.resourceName=~"pods/exec"
  AND timestamp>="2026-02-18T00:00:00Z"
' --format json

# Correlate with:
# - User identity
# - Source IP
# - Time of day (off-hours?)
# - Pod security context
```

**4. Failed Authentication Attempts:**

```bash
# Find authentication failures
gcloud logging read '
  resource.type="k8s_cluster"
  AND protoPayload.status.code!=0
  AND protoPayload.authenticationInfo.principalEmail!=""
  AND timestamp>="2026-02-18T00:00:00Z"
' --format json | jq -r '.[] | .protoPayload.authenticationInfo.principalEmail' | sort | uniq -c
```

**Sample Audit Log Entry:**

```json
{
  "protoPayload": {
    "@type": "type.googleapis.com/google.cloud.audit.AuditLog",
    "authenticationInfo": {
      "principalEmail": "attacker@external.com"
    },
    "authorizationInfo": [
      {
        "permission": "io.k8s.core.v1.secrets.get",
        "resource": "core/v1/secrets/production/db-password",
        "granted": false
      }
    ],
    "methodName": "io.k8s.core.v1.secrets.get",
    "requestMetadata": {
      "callerIp": "203.0.113.42",
      "callerSuppliedUserAgent": "kubectl/v1.25.0"
    },
    "resourceName": "core/v1/namespaces/production/secrets/db-password",
    "status": {
      "code": 7,
      "message": "Forbidden"
    }
  },
  "timestamp": "2026-02-18T14:23:15.123456Z"
}
```

### Audit Log Alerting

```bash
# Create log-based metric for failed secret access
gcloud logging metrics create failed_secret_access \
  --description="Failed attempts to access secrets" \
  --log-filter='
    resource.type="k8s_cluster"
    AND protoPayload.resourceName=~"secrets"
    AND protoPayload.status.code!=0
  '

# Create alert policy
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="Failed Secret Access Alert" \
  --condition-display-name="Failed secret access rate" \
  --condition-threshold-value=5 \
  --condition-threshold-duration=60s \
  --condition-filter='
    metric.type="logging.googleapis.com/user/failed_secret_access"
    resource.type="k8s_cluster"
  '
```

---

## 19.8 Network Security

### Network Policies for Micro-segmentation

Network policies control traffic between pods (layer 3/4 firewall).

**Default Deny All:**

```yaml
# Deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
  - Ingress

---
# Deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

**Allow Specific Ingress:**

```yaml
# Allow ingress to web app from ingress controller only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-web
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    # Allow from ingress controller namespace
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

**Allow Egress to External Services:**

```yaml
# Allow egress to specific external IPs (e.g., GCP APIs)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-gcp-apis
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # DNS (required for name resolution)
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # GCP APIs (Private Google Access)
  - to:
    - ipBlock:
        cidr: 199.36.153.8/30  # GCP Private Google Access
    ports:
    - protocol: TCP
      port: 443
  # Specific external service
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

**Multi-tier Application:**

```yaml
# Frontend → Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
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
# Backend → Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: production
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
      port: 5432
```

**Cross-namespace Communication:**

```yaml
# Allow production app to access shared services namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-production
  namespace: shared-services
spec:
  podSelector:
    matchLabels:
      app: shared-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 8080
```

### Service Mesh mTLS

**Istio mTLS Configuration:**

```yaml
# Enable strict mTLS for entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all traffic

---
# Per-workload mTLS override
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: web-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: web
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: PERMISSIVE  # Allow plaintext on port 8080 (e.g., health checks)

---
# Authorization policy (after mTLS authentication)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  # Allow from frontend service account
  - from:
    - source:
        principals:
        - cluster.local/ns/production/sa/frontend-sa
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]

---
# Deny policy (takes precedence)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  action: DENY
  rules:
  - from:
    - source:
        notNamespaces: ["production", "monitoring"]
```

**Verify mTLS:**

```bash
# Check mTLS status
istioctl x describe pod web-pod-xxx -n production

# Expected output shows:
# * mTLS: STRICT
# * Certificate: Present
```

### Egress Control

**1. Block All Egress by Default:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS only
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**2. Istio Egress Gateway:**

```yaml
# Egress Gateway for external traffic
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: egress-gateway
  namespace: production
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: tls
      protocol: TLS
    hosts:
    - "api.external.com"
    tls:
      mode: PASSTHROUGH

---
# VirtualService to route through egress gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: external-api
  namespace: production
spec:
  hosts:
  - "api.external.com"
  gateways:
  - mesh
  - egress-gateway
  http:
  - match:
    - gateways:
      - mesh
      port: 443
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 443
  - match:
    - gateways:
      - egress-gateway
      port: 443
    route:
    - destination:
        host: api.external.com
        port:
          number: 443

---
# ServiceEntry for external service
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
  namespace: production
spec:
  hosts:
  - "api.external.com"
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

**3. GKE Private Clusters:**

```bash
# Create private cluster (nodes have no public IPs)
gcloud container clusters create private-cluster \
  --region us-central1 \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr 172.16.0.0/28 \
  --enable-ip-alias \
  --create-subnetwork=""

# Control plane authorized networks (for kubectl access)
gcloud container clusters update private-cluster \
  --region us-central1 \
  --enable-master-authorized-networks \
  --master-authorized-networks 203.0.113.0/24
```

### Encryption in Transit

**GKE Features:**

1. **Control Plane to Node:** Always encrypted (TLS)
2. **Node to Node:** Enable Dataplane V2 with encryption

```bash
# Enable Dataplane V2 (Cilium-based) with encryption
gcloud container clusters create encrypted-cluster \
  --region us-central1 \
  --enable-dataplane-v2 \
  --enable-dataplane-v2-encryption \
  --network-policy-provider=CALICO
```

3. **Application to Application:** Use service mesh (Istio, Anthos Service Mesh)

```bash
# Install Anthos Service Mesh
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.18.1-asm.1-linux-amd64.tar.gz
tar xzf istio-1.18.1-asm.1-linux-amd64.tar.gz
./istio-1.18.1-asm.1/bin/istioctl install \
  --set profile=asm-gcp \
  --set revision=asm-1181-1

# Label namespace for sidecar injection
kubectl label namespace production istio-injection=enabled
```

---

## Chapter 20 Review Questions

1. **Multi-layered Defense:** Explain the 4 C's of Cloud Native Security and provide one concrete security control at each layer for a GKE application. How does failure at one layer get mitigated by controls at other layers?

2. **RBAC Design:** You need to grant a CI/CD service account permission to deploy applications to the `production` namespace, including creating/updating Deployments and Services, but NOT reading Secrets or modifying RBAC. Write the Role and RoleBinding YAML. What principle guides this design?

3. **Pod Security Standards:** Compare the Privileged, Baseline, and Restricted Pod Security Standards. For a production web application handling PCI data, which standard would you enforce and why? What specific SecurityContext fields are required?

4. **Secrets Management:** Compare the security characteristics of: (1) K8s Secrets with encryption at rest, (2) External Secrets Operator with GCP Secret Manager, and (3) HashiCorp Vault. When would you choose each approach?

5. **Runtime Security:** You deploy Falco and receive an alert: "Suspicious outbound connection from container 'app' to IP 198.51.100.42". Describe your incident investigation workflow using audit logs, network policies, and container inspection. What immediate mitigation steps would you take?

---

## Chapter 20 Hands-On Exercises

### Exercise 1: Implement Defense in Depth

Deploy a three-tier application (frontend → backend → database) with comprehensive security controls:

```bash
# 1. Create namespace with Pod Security Standards
kubectl create namespace secure-app
kubectl label namespace secure-app \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# 2. Create service accounts
kubectl create sa frontend-sa -n secure-app
kubectl create sa backend-sa -n secure-app
kubectl create sa database-sa -n secure-app

# 3. Create RBAC (backend needs to read ConfigMaps only)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: secure-app
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  resourceNames: ["backend-config"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-binding
  namespace: secure-app
subjects:
- kind: ServiceAccount
  name: backend-sa
roleRef:
  kind: Role
  name: backend-role
  apiGroup: rbac.authorization.k8s.io
EOF

# 4. Deploy with security contexts
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: secure-app
spec:
  replicas: 2
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
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: backend
        image: gcr.io/google-samples/hello-app:2.0
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        resources:
          limits:
            memory: "128Mi"
            cpu: "250m"
          requests:
            memory: "64Mi"
            cpu: "100m"
      volumes:
      - name: tmp
        emptyDir: {}
EOF

# 5. Apply network policies
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
  namespace: secure-app
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
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
EOF

# 6. Test security controls
kubectl run test-pod --rm -it --image=busybox -n secure-app -- sh
# Try to violate policies and observe denials
```

### Exercise 2: Image Supply Chain Security

Set up a secure container image pipeline with scanning, signing, and admission control:

```bash
# 1. Scan image with Trivy
docker build -t gcr.io/${PROJECT_ID}/app:v1 .
trivy image --severity HIGH,CRITICAL gcr.io/${PROJECT_ID}/app:v1

# 2. Fix vulnerabilities and rebuild
# ... update Dockerfile with patched base image ...
docker build -t gcr.io/${PROJECT_ID}/app:v1 .

# 3. Sign image with Cosign
docker push gcr.io/${PROJECT_ID}/app:v1
cosign sign --key cosign.key gcr.io/${PROJECT_ID}/app@sha256:$(crane digest gcr.io/${PROJECT_ID}/app:v1)

# 4. Configure Binary Authorization
gcloud container binauthz policy import policy.yaml

# 5. Try to deploy unsigned image (should fail)
kubectl run test --image=gcr.io/${PROJECT_ID}/unsigned:latest
# Expected: Error from admission controller

# 6. Deploy signed image (should succeed)
kubectl run app --image=gcr.io/${PROJECT_ID}/app@sha256:xxx
```

### Exercise 3: Runtime Security with Falco

Deploy Falco and create custom detection rules:

```bash
# 1. Install Falco
helm install falco falcosecurity/falco \
  --namespace falco-system \
  --create-namespace \
  --set driver.kind=ebpf

# 2. Create custom rules ConfigMap
kubectl apply -f custom-falco-rules.yaml

# 3. Deploy test workload
kubectl run test-pod --image=ubuntu -- sleep 3600

# 4. Trigger alerts
kubectl exec test-pod -- bash -c "curl http://suspicious-site.com"
kubectl exec test-pod -- cat /etc/shadow

# 5. View alerts
kubectl logs -n falco-system -l app.kubernetes.io/name=falco

# 6. Create incident response runbook based on alerts
```

---

## Key Takeaways

1. **Defense in Depth is Essential:** Kubernetes security requires layered controls across cloud infrastructure, cluster configuration, container runtime, and application code. A breach at one layer should be contained by controls at other layers.

2. **RBAC Least Privilege:** Every service account should have only the minimum permissions required for its function. Use namespace-scoped Roles instead of ClusterRoles when possible, avoid wildcard permissions, and regularly audit RBAC configurations.

3. **Pod Security Standards Enforcement:** The Restricted Pod Security Standard should be the default for production workloads. It requires running as non-root, dropping all capabilities, setting seccomp profiles, and blocking privilege escalation.

4. **Secrets Should Never Be in Git or Containers:** Use external secret management systems (GCP Secret Manager, Vault) with encryption at rest and short-lived, bound service account tokens. Kubernetes Secrets alone provide only encoding, not encryption.

5. **Image Supply Chain Security Matters:** Scan images for vulnerabilities with Trivy or GCP Container Analysis, sign images with Cosign, pin images by digest not tags, use minimal base images (distroless), and enforce admission policies with Binary Authorization.

6. **Runtime Security Provides Real-time Threat Detection:** Tools like Falco and Tetragon detect anomalous behavior (unexpected process execution, file access, network connections) that static policies cannot catch. Integrate with incident response workflows.

7. **Network Micro-segmentation Limits Blast Radius:** Default-deny network policies prevent lateral movement after container compromise. Service mesh mTLS provides identity-based authorization and encryption in transit between services.

[Next Chapter: Kubernetes Cluster Operations →](20-cluster-operations.md)