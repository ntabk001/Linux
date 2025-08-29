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

### Cài đặt SOPS và Kustomize trên tất cả worker nodes
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

# Set ownership cho ArgoCD user
sudo chown -R 999:999 /opt/k8s-tools

# Verify tools
/opt/k8s-tools/bin/sops --version
/opt/k8s-tools/bin/kustomize version
```

## 3. Cấu hình SOPS

### Tạo SOPS config
```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    pgp: 'YOUR_GPG_FINGERPRINT'  # Thay bằng fingerprint thật
  - path_regex: overlays/.*/secrets\.yaml$
    pgp: 'YOUR_GPG_FINGERPRINT'
  - path_regex: overlays/.*/config\.yaml$
    pgp: 'YOUR_GPG_FINGERPRINT'
```

## 4. Cài đặt ArgoCD với Custom Configuration

### Complete ArgoCD deployment với SOPS support
```yaml
# argocd-sops-install.yaml
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
  labels:
    app.kubernetes.io/name: argocd-repo-server
type: Opaque
stringData:
  private.key: |
    -----BEGIN PGP PRIVATE KEY BLOCK-----
    # Paste nội dung từ file argocd-gpg-private.key ở đây
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
            export GNUPGHOME=/gpg-home
            gpg --batch --import /gpg-keys/private.key 2>/dev/null || true
            gpg --list-secret-keys
            echo "GPG setup completed"
      generate:
        command: [sh, -c]
        args:
          - |
            echo "=== SOPS Decryption Process ==="
            export GNUPGHOME=/gpg-home
            
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
# ArgoCD Configuration
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
            export GNUPGHOME=/gpg-home
            gpg --batch --import /gpg-keys/private.key 2>/dev/null || true
      generate:
        command: [sh, -c]
        args:
          - |
            export GNUPGHOME=/gpg-home
            find . -name "*.enc.yaml" -type f | while read -r file; do
              /opt/k8s-tools/bin/sops --decrypt "$file" > "${file%.enc.yaml}.yaml"
            done
            /opt/k8s-tools/bin/kustomize build .
---
# Custom Repository Server ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-repo-server
  namespace: argocd
---
# Repository Server Deployment
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
        - name: SOPS_PGP_FP
          value: "YOUR_GPG_FINGERPRINT"  # Thay bằng fingerprint thật
        - name: GNUPGHOME
          value: /gpg-home
        - name: PATH
          value: "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/k8s-tools/bin"
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
          mountPath: /gpg-home
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
      - name: setup-gpg
        image: alpine:latest
        command: [sh, -c]
        args:
        - |
          apk add --no-cache gnupg
          
          # Setup GPG home directory
          mkdir -p /gpg-home
          chmod 700 /gpg-home
          export GNUPGHOME=/gpg-home
          
          # Import GPG key
          gpg --batch --import /gpg-keys/private.key
          gpg --list-secret-keys
          
          # Set proper ownership
          chown -R 999:999 /gpg-home
          
          echo "Verifying tools..."
          /opt/k8s-tools/bin/sops --version
          /opt/k8s-tools/bin/kustomize version
          echo "GPG and tools setup completed successfully"
        volumeMounts:
        - name: gpg-keys
          mountPath: /gpg-keys
          readOnly: true
        - name: gpg-keyring
          mountPath: /gpg-home
        - name: tools-volume
          mountPath: /opt/k8s-tools
          readOnly: true
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
      - name: tmp
        emptyDir:
          sizeLimit: 100Mi
---
# Repository Server Service
apiVersion: v1
kind: Service
metadata:
  name: argocd-repo-server
  namespace: argocd
  labels:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
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
```

## 5. Cấu trúc Repository

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

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
```

```yaml
# base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  PORT: "80"
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
  - secrets.yaml      # Sẽ được decrypt từ secrets.enc.yaml
  - config.yaml       # Sẽ được decrypt từ config.enc.yaml

replicas:
  - name: my-app
    count: 1
```

## 6. Tạo và mã hóa Secrets

### Script mã hóa secrets
```bash
#!/bin/bash
# scripts/encrypt-secrets.sh

# Development secrets
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
  JWT_SECRET: "dev-jwt-secret-789"
EOF

# Mã hóa development secrets
sops --encrypt overlays/development/secrets.yaml > overlays/development/secrets.enc.yaml
rm overlays/development/secrets.yaml

# Development config
cat > overlays/development/config.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-encrypted
data:
  DATABASE_HOST: "dev-database.local"
  CACHE_SIZE: "512"
  DEBUG_MODE: "true"
EOF

# Mã hóa development config
sops --encrypt overlays/development/config.yaml > overlays/development/config.enc.yaml
rm overlays/development/config.yaml

echo "Secrets và config đã được mã hóa thành công!"
```

### Ví dụ file đã mã hóa
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
    JWT_SECRET: ENC[AES256_GCM,data:mK4pVz...encrypted...==,type:str]
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

## 7. Deploy ArgoCD

### Cài đặt ArgoCD base và custom repo-server
```bash
# 1. Cài đặt ArgoCD base components
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Xóa default repo-server deployment
kubectl delete deployment argocd-repo-server -n argocd

# 3. Apply custom configuration với SOPS support
# Nhớ thay YOUR_GPG_FINGERPRINT trong file trước khi apply
kubectl apply -f argocd-sops-install.yaml

# 4. Verify deployment
kubectl get pods -n argocd
kubectl logs deployment/argocd-repo-server -n argocd -c setup-gpg
```

## 8. Tạo ArgoCD Application

```yaml
# application-dev.yaml
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
---
# Production Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-encrypted-app-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/my-encrypted-app
    targetRevision: HEAD
    path: overlays/production
    plugin:
      name: sops-kustomize
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    # Production không auto sync để kiểm soát tốt hơn
```

## 9. Upload lên GitHub

### Gitignore configuration
```bash
# .gitignore
# Không commit file secrets đã decrypt
*.yaml
!*.enc.yaml
!kustomization.yaml
!base/*.yaml

# Private keys và sensitive files
*.key
*.asc
.sops.yaml.bak
private/
.gnupg/

# Temporary files
.tmp/
*.tmp
```

### Git workflow
```bash
# Initialize repository
git init
git add .
git commit -m "Initial encrypted application with SOPS"

# Add remote và push
git remote add origin https://github.com/your-username/my-encrypted-app.git
git branch -M main
git push -u origin main
```

## 10. Deploy và Verify

### Deploy applications
```bash
# Deploy development environment
kubectl apply -f application-dev.yaml

# Deploy production environment  
kubectl apply -f application-prod.yaml

# Kiểm tra application status
kubectl get applications -n argocd

# Xem chi tiết sync process
kubectl describe application my-encrypted-app-dev -n argocd
```

### Verification
```bash
# Kiểm tra pods đã chạy
kubectl get pods -n development
kubectl get pods -n production

# Verify secrets đã được giải mã và tạo
kubectl get secret app-secrets -n development -o yaml

# Test environment variables trong pods
kubectl exec deployment/my-app -n development -- env | grep -E "(DATABASE_URL|API_KEY)"

# Kiểm tra logs để đảm bảo app hoạt động bình thường
kubectl logs deployment/my-app -n development
```

## 11. Quy trình làm việc hàng ngày

### Cập nhật secrets
```bash
# Edit encrypted secrets (SOPS sẽ tự động decrypt khi mở editor)
sops overlays/development/secrets.enc.yaml

# Sau khi chỉnh sửa và save, file sẽ được tự động mã hóa lại
git add overlays/development/secrets.enc.yaml
git commit -m "Update development database credentials"
git push

# ArgoCD sẽ tự động detect changes và sync
```

### Thêm secrets mới
```bash
# Tạo secrets mới cho production
sops overlays/production/new-feature-secrets.enc.yaml

# Update kustomization.yaml để include file mới
# Commit và push changes
```

### Rotate GPG keys
```bash
# 1. Tạo GPG key mới
gpg --full-generate-key

# 2. Re-encrypt tất cả secrets với key mới
find . -name "*.enc.yaml" -exec sops updatekeys {} \;

# 3. Update ArgoCD với key mới
# 4. Commit và deploy
```

## Troubleshooting

### Kiểm tra tools availability
```bash
kubectl exec deployment/argocd-repo-server -n argocd -- ls -la /opt/k8s-tools/bin/
kubectl exec deployment/argocd-repo-server -n argocd -- /opt/k8s-tools/bin/sops --version
kubectl exec deployment/argocd-repo-server -n argocd -- /opt/k8s-tools/bin/kustomize version
```

### Debug GPG setup
```bash
# Kiểm tra GPG keyring location
kubectl exec deployment/argocd-repo-server -n argocd -- ls -la /gpg-home/

# Test GPG trong pod
kubectl exec deployment/argocd-repo-server -n argocd -- sh -c "
export GNUPGHOME=/gpg-home
gpg --list-keys
gpg --list-secret-keys
"

# Test SOPS decrypt với GPG home đúng
kubectl exec deployment/argocd-repo-server -n argocd -- sh -c "
export GNUPGHOME=/gpg-home
echo 'test' | /opt/k8s-tools/bin/sops --encrypt --pgp YOUR_GPG_FINGERPRINT /dev/stdin
"
```

### Sửa lỗi read-only filesystem
```bash
# Nếu vẫn gặp lỗi read-only, kiểm tra securityContext
kubectl get deployment argocd-repo-server -n argocd -o yaml | grep -A 10 securityContext

# Đảm bảo tmp và gpg-home volumes có quyền write
kubectl exec deployment/argocd-repo-server -n argocd -- df -h | grep tmp
```

### Debug decryption process
```bash
# Xem logs của repo server
kubectl logs deployment/argocd-repo-server -n argocd -c argocd-repo-server

# Test manual decryption
kubectl exec deployment/argocd-repo-server -n argocd -- sh -c "
cd /tmp && 
echo 'test' | /opt/k8s-tools/bin/sops --encrypt --pgp YOUR_GPG_FINGERPRINT /dev/stdin
"
```

### Sync issues
```bash
# Force refresh repository
argocd app get my-encrypted-app-dev --refresh

# Manual sync với logs
argocd app sync my-encrypted-app-dev --prune

# Xem sync logs chi tiết
kubectl logs deployment/argocd-repo-server -n argocd | grep -A 10 -B 10 sops
```

## Lưu ý quan trọng

### Bảo mật
- ✅ Private GPG key chỉ tồn tại trong ArgoCD cluster
- ✅ GitHub chỉ chứa dữ liệu đã mã hóa
- ✅ Decryption process hoàn toàn tự động trong cluster
- ✅ Tools có sẵn trên server, kiểm soát version tốt

### Performance  
- ⚡ Khởi tạo nhanh vì tools đã có sẵn
- ⚡ Không phụ thuộc internet để download tools
- ⚡ HostPath mount nhanh hơn PV

### Quản lý
- 🔄 Dễ dàng cập nhật tools version trên server
- 🔄 Restart ArgoCD repo-server để áp dụng tools mới
- 🔄 Backup và restore GPG keys đơn giản