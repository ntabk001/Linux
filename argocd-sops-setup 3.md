# Hướng dẫn ArgoCD + SOPS + GPG với Tools có sẵn trên Server

## 1. Chuẩn bị GPG Key

### Tạo GPG Key
```bash
# Tạo GPG key
gpg --full-generate-key

# Chọn RSA, 4096 bit, không hết hạn
# Export fingerprint
gpg --list-secret-keys --keyid-format LONG
export GPG_KEY_ID="your-key-id"
export GPG_FINGERPRINT="your-full-fingerprint"

# Export private key cho ArgoCD
gpg --armor --export-secret-keys $GPG_KEY_ID > argocd-gpg-private.key
```

## 2. Chuẩn bị Tools trên Server

### Tạo thư mục và download tools
```bash
# Tạo thư mục tools trên tất cả worker nodes
sudo mkdir -p /opt/k8s-tools/bin

# Download SOPS
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 \
  -o /opt/k8s-tools/bin/sops
sudo chmod +x /opt/k8s-tools/bin/sops

# Download Kustomize
cd /tmp
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /opt/k8s-tools/bin/
sudo chmod +x /opt/k8s-tools/bin/kustomize

# Set ownership
sudo chown -R 999:999 /opt/k8s-tools
```

## 3. Cài đặt ArgoCD với SOPS Support

### ArgoCD Installation với custom configuration
```yaml
# argocd-install.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
# GPG Secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-gpg-keys
  namespace: argocd
type: Opaque
stringData:
  key: |
    -----BEGIN PGP PRIVATE KEY BLOCK-----
    # Paste nội dung của argocd-gpg-private.key ở đây
    -----END PGP PRIVATE KEY BLOCK-----
---
# SOPS Plugin ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: sops-kustomize-plugin
  namespace: argocd
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: sops-kustomize
    spec:
      version: v1.0
      init:
        command: [sh, -c]
        args:
          - |
            echo "Initializing SOPS and GPG..."
            gpg --import /gpg-keys/key
            echo "GPG keys imported successfully"
      generate:
        command: [sh, -c]
        args:
          - |
            # Decrypt all .enc.yaml files
            echo "Starting decryption process..."
            find . -name "*.enc.yaml" -type f | while read -r file; do
              echo "Decrypting $file"
              /opt/k8s-tools/bin/sops --decrypt "$file" > "${file%.enc.yaml}.yaml"
            done
            
            # Run kustomize build
            echo "Running kustomize build..."
            /opt/k8s-tools/bin/kustomize build .
      discover:
        find:
          command: [sh, -c, "find . -name kustomization.yaml"]
---
# ArgoCD Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cmd-params-cm
    app.kubernetes.io/part-of: argocd
data:
  server.insecure: "true"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  configManagementPlugins: |
    - name: sops-kustomize
      init:
        command: [sh, -c]
        args:
          - |
            gpg --import /gpg-keys/key
      generate:
        command: [sh, -c]
        args:
          - |
            find . -name "*.enc.yaml" -type f | while read -r file; do
              /opt/k8s-tools/bin/sops --decrypt "$file" > "${file%.enc.yaml}.yaml"
            done
            /opt/k8s-tools/bin/kustomize build .
```

### ArgoCD Repository Server Deployment
```yaml
# argocd-repo-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-repo-server
    spec:
      serviceAccountName: argocd-repo-server
      containers:
      - name: argocd-repo-server
        image: quay.io/argoproj/argocd:v2.9.0
        command:
        - argocd-repo-server
        env:
        - name: SOPS_PGP_FP
          value: "YOUR_GPG_FINGERPRINT"  # Thay bằng fingerprint thật
        - name: GNUPGHOME
          value: /home/argocd/.gnupg
        ports:
        - containerPort: 8081
        - containerPort: 8084
        volumeMounts:
        - name: ssh-known-hosts
          mountPath: /app/config/ssh
        - name: tls-certs
          mountPath: /app/config/tls
        - name: gpg-keys
          mountPath: /gpg-keys
          readOnly: true
        - name: gpg-keyring
          mountPath: /home/argocd/.gnupg
        - name: argocd-repo-server-tls
          mountPath: /app/config/server/tls
        - name: tools-volume
          mountPath: /opt/k8s-tools
          readOnly: true
        - name: sops-plugin
          mountPath: /home/argocd/cmp-server/plugins
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8084
          initialDelaySeconds: 30
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8084
          initialDelaySeconds: 5
          periodSeconds: 10
      initContainers:
      - name: setup-gpg
        image: alpine:latest
        command: [sh, -c]
        args:
        - |
          apk add --no-cache gnupg
          mkdir -p /home/argocd/.gnupg
          chmod 700 /home/argocd/.gnupg
          gpg --homedir /home/argocd/.gnupg --import /gpg-keys/key
          chown -R 999:999 /home/argocd/.gnupg
        volumeMounts:
        - name: gpg-keys
          mountPath: /gpg-keys
          readOnly: true
        - name: gpg-keyring
          mountPath: /home/argocd/.gnupg
        securityContext:
          runAsUser: 0
      volumes:
      - name: ssh-known-hosts
        configMap:
          name: argocd-ssh-known-hosts-cm
      - name: tls-certs
        configMap:
          name: argocd-tls-certs-cm
      - name: gpg-keys
        secret:
          secretName: argocd-gpg-keys
          defaultMode: 0600
      - name: gpg-keyring
        emptyDir: {}
      - name: argocd-repo-server-tls
        secret:
          secretName: argocd-repo-server-tls
          optional: true
      - name: tools-volume
        hostPath:
          path: /opt/k8s-tools
          type: Directory
      - name: sops-plugin
        configMap:
          name: sops-kustomize-plugin
```

### Các Services và RBAC cần thiết
```yaml
# argocd-services.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-repo-server
  namespace: argocd
---
apiVersion: v1
kind: Service
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  ports:
  - name: server
    port: 8081
    protocol: TCP
    targetPort: 8081
  - name: metrics
    port: 8084
    protocol: TCP
    targetPort: 8084
  selector:
    app.kubernetes.io/name: argocd-repo-server
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-repo-server
  namespace: argocd
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-repo-server
  namespace: argocd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-repo-server
subjects:
- kind: ServiceAccount
  name: argocd-repo-server
  namespace: argocd
```

## 4. Cấu trúc Repository

### Cấu trúc thư mục
```
my-encrypted-app/
├── .sops.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── secrets.enc.yaml
│   │   └── config.enc.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── secrets.enc.yaml
│       └── config.enc.yaml
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── scripts/
    └── encrypt-secrets.sh
```

### SOPS Configuration
```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    pgp: 'YOUR_GPG_FINGERPRINT'
  - path_regex: overlays/.*/secrets\.yaml$
    pgp: 'YOUR_GPG_FINGERPRINT'
  - path_regex: overlays/.*/config\.yaml$
    pgp: 'YOUR_GPG_FINGERPRINT'
```

### Base Resources
```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: my-encrypted-app
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        ports:
        - containerPort: 80
        envFrom:
        - secretRef:
            name: app-secrets
        - configMapRef:
            name: app-config-encrypted
```

### Development Overlay
```yaml
# overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

resources:
  - ../../base

patchesStrategicMerge:
  - secrets.yaml  # Sẽ được decrypt từ secrets.enc.yaml
  - config.yaml   # Sẽ được decrypt từ config.enc.yaml

replicas:
  - name: my-app
    count: 1
```

### Tạo và mã hóa secrets
```bash
# Tạo file secrets gốc
cat > overlays/development/secrets.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://dev-user:dev-pass@db:5432/myapp"
  API_KEY: "dev-api-key-123456"
  REDIS_URL: "redis://redis:6379/0"
EOF

# Mã hóa file
sops --encrypt overlays/development/secrets.yaml > overlays/development/secrets.enc.yaml

# Xóa file gốc
rm overlays/development/secrets.yaml
```

### Encrypted secrets sẽ như thế này:
```yaml
# overlays/development/secrets.enc.yaml
apiVersion: v1
kind: Secret
metadata:
    name: app-secrets
type: Opaque
stringData:
    DATABASE_URL: ENC[AES256_GCM,data:vB7+Tx...encrypted...==,type:str]
    API_KEY: ENC[AES256_GCM,data:gH8kLm...encrypted...==,type:str]
    REDIS_URL: ENC[AES256_GCM,data:pX9nQw...encrypted...==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2025-08-29T10:00:00Z"
    mac: ENC[AES256_GCM,data:mac...==,type:str]
    pgp:
        - created_at: "2025-08-29T10:00:00Z"
          enc: |
            -----BEGIN PGP MESSAGE-----
            hQIMA...encrypted_key...=
            -----END PGP MESSAGE-----
          fp: YOUR_GPG_FINGERPRINT
    version: 3.8.1
```

## 5. Deploy ArgoCD với Custom Configuration

### Complete ArgoCD deployment
```yaml
# argocd-custom-install.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
# Import standard ArgoCD resources
apiVersion: v1
kind: List
items:
# Thêm tất cả resources từ https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# NHƯNG thay thế argocd-repo-server deployment bằng custom version bên dưới
---
# Custom GPG Secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-gpg-keys
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-repo-server
type: Opaque
stringData:
  private.key: |
    -----BEGIN PGP PRIVATE KEY BLOCK-----
    # Paste GPG private key content here
    -----END PGP PRIVATE KEY BLOCK-----
---
# SOPS Plugin Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: sops-kustomize-plugin
  namespace: argocd
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: sops-kustomize
    spec:
      version: v1.0
      init:
        command: [sh, -c]
        args:
          - |
            echo "Setting up GPG for SOPS..."
            gpg --batch --import /gpg-keys/private.key 2>/dev/null || true
            gpg --list-secret-keys
      generate:
        command: [sh, -c]
        args:
          - |
            echo "=== SOPS Decryption Process ==="
            
            # Find and decrypt .enc.yaml files
            find . -name "*.enc.yaml" -type f | while read -r file; do
              decrypted_file="${file%.enc.yaml}.yaml"
              echo "Decrypting: $file -> $decrypted_file"
              
              if /opt/k8s-tools/bin/sops --decrypt "$file" > "$decrypted_file"; then
                echo "✓ Successfully decrypted $file"
              else
                echo "✗ Failed to decrypt $file"
                exit 1
              fi
            done
            
            echo "=== Kustomize Build Process ==="
            /opt/k8s-tools/bin/kustomize build . || exit 1
      discover:
        find:
          command: [sh, -c]
          args: ["find . -name kustomization.yaml"]
---
# Custom Repository Server Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
  labels:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-repo-server
    spec:
      serviceAccountName: argocd-repo-server
      automountServiceAccountToken: false
      containers:
      - name: argocd-repo-server
        image: quay.io/argoproj/argocd:v2.9.0
        command:
        - uid_entrypoint.sh
        - argocd-repo-server
        - --redis
        - argocd-redis:6379
        env:
        - name: ARGOCD_RECONCILIATION_TIMEOUT
          valueFrom:
            configMapKeyRef:
              name: argocd-cm
              key: timeout.reconciliation
              optional: true
        - name: ARGOCD_REPO_SERVER_LOGFORMAT
          valueFrom:
            configMapKeyRef:
              name: argocd-cmd-params-cm
              key: reposerver.log.format
              optional: true
        - name: SOPS_PGP_FP
          value: "YOUR_GPG_FINGERPRINT"  # Thay bằng fingerprint thật
        - name: GNUPGHOME
          value: /home/argocd/.gnupg
        ports:
        - containerPort: 8081
        - containerPort: 8084
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8084
          initialDelaySeconds: 30
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8084
          initialDelaySeconds: 5
          periodSeconds: 10
        volumeMounts:
        - name: ssh-known-hosts
          mountPath: /app/config/ssh
        - name: tls-certs
          mountPath: /app/config/tls
        - name: gpg-keys
          mountPath: /gpg-keys
          readOnly: true
        - name: gpg-keyring
          mountPath: /home/argocd/.gnupg
        - name: argocd-repo-server-tls
          mountPath: /app/config/server/tls
        - name: tools-volume
          mountPath: /opt/k8s-tools
          readOnly: true
        - name: plugins
          mountPath: /home/argocd/cmp-server/plugins
        - name: tmp
          mountPath: /tmp
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 999
      initContainers:
      - name: copyutil
        image: quay.io/argoproj/argocd:v2.9.0
        command:
        - cp
        - -n
        - /usr/local/bin/argocd
        - /var/run/argocd/argocd-cmp-server
        volumeMounts:
        - name: var-files
          mountPath: /var/run/argocd
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 999
      - name: setup-tools
        image: alpine:latest
        command: [sh, -c]
        args:
        - |
          echo "Verifying tools availability..."
          ls -la /opt/k8s-tools/bin/
          /opt/k8s-tools/bin/sops --version
          /opt/k8s-tools/bin/kustomize version
          echo "Tools verification completed"
        volumeMounts:
        - name: tools-volume
          mountPath: /opt/k8s-tools
          readOnly: true
      volumes:
      - name: ssh-known-hosts
        configMap:
          name: argocd-ssh-known-hosts-cm
      - name: tls-certs
        configMap:
          name: argocd-tls-certs-cm
      - name: gpg-keys
        secret:
          secretName: argocd-gpg-keys
          defaultMode: 0400
      - name: gpg-keyring
        emptyDir:
          sizeLimit: 10Mi
      - name: argocd-repo-server-tls
        secret:
          secretName: argocd-repo-server-tls
          optional: true
      - name: tools-volume
        hostPath:
          path: /opt/k8s-tools
          type: Directory
      - name: plugins
        configMap:
          name: sops-kustomize-plugin
      - name: var-files
        emptyDir: {}
      - name: tmp
        emptyDir: {}
```

## 6. Triển khai ArgoCD

```bash
# 1. Download ArgoCD base manifests
curl -o argocd-base.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Apply base manifests (trừ repo-server deployment)
kubectl apply -f argocd-base.yaml

# 3. Delete default repo-server deployment
kubectl delete deployment argocd-repo-server -n argocd

# 4. Apply custom configuration
kubectl apply -f argocd-custom-install.yaml

# 5. Verify deployment
kubectl get pods -n argocd
kubectl logs deployment/argocd-repo-server -n argocd -c setup-tools
```

## 7. Tạo ArgoCD Application

```yaml
# my-app-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-encrypted-app-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/my-encrypted-app
    targetRevision: HEAD
    path: overlays/development
    plugin:
      name: sops-kustomize
  destination:
    server: https://kubernetes.default.svc
    namespace: development
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

## 8. Upload lên GitHub

```bash
# Chuẩn bị gitignore
cat > .gitignore << EOF
# Chỉ allow encrypted files
*.yaml
!*.enc.yaml
!kustomization.yaml
!base/*.yaml

# Private keys
*.key
*.asc
.sops.yaml.bak
EOF

# Git commands
git init
git add .
git commit -m "Initial encrypted application"
git remote add origin https://github.com/your-username/my-encrypted-app.git
git branch -M main
git push -u origin main
```

## 9. Deploy và Verify

```bash
# Deploy application
kubectl apply -f my-app-application.yaml

# Kiểm tra sync status
kubectl get application my-encrypted-app-dev -n argocd

# Force sync nếu cần
argocd app sync my-encrypted-app-dev

# Verify pods đã chạy với secrets đã decrypt
kubectl get pods -n development
kubectl get secret app-secrets -n development -o yaml

# Test env vars trong pod
kubectl exec deployment/my-app -n development -- env | grep -E "(DATABASE_URL|API_KEY|REDIS_URL)"
```

## Workflow hàng ngày

### Cập nhật secrets
```bash
# Edit encrypted file (sẽ tự động decrypt khi mở)
sops overlays/development/secrets.enc.yaml

# Commit và push
git add .
git commit -m "Update development secrets"
git push

# ArgoCD sẽ tự động detect thay đổi và sync
```

### Thêm secret mới cho production
```bash
# Tạo và mã hóa production secrets
sops overlays/production/secrets.enc.yaml

# Update kustomization nếu cần
# Commit và push
git add overlays/production/
git commit -m "Add production secrets"
git push
```

## Troubleshooting

### Kiểm tra tools availability
```bash
kubectl exec deployment/argocd-repo-server -n argocd -- ls -la /opt/k8s-tools/bin/
kubectl exec deployment/argocd-repo-server -n argocd -- /opt/k8s-tools/bin/sops --version
```

### Debug decryption process
```bash
kubectl logs deployment/argocd-repo-server -n argocd -c argocd-repo-server
kubectl exec deployment/argocd-repo-server -n argocd -- gpg --list-keys
```