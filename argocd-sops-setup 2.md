# Hướng dẫn thiết lập ArgoCD với GPG + SOPS + Kustomize

## 1. Chuẩn bị và tạo GPG Key

### Tạo GPG Key
```bash
# Tạo GPG key mới
gpg --full-generate-key

# Chọn:
# - RSA and RSA (default)
# - 4096 bit
# - Key không hết hạn (0)
# - Nhập thông tin: Real name, Email, Comment

# Liệt kê keys
gpg --list-secret-keys --keyid-format LONG

# Export public key
gpg --armor --export your-key-id > public.key

# Export private key (để import vào ArgoCD)
gpg --armor --export-secret-keys your-key-id > private.key
```

### Cấu hình SOPS
Tạo file `.sops.yaml` trong root của repository:
```yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    pgp: 'your-gpg-key-fingerprint'
  - path_regex: secrets/.*\.yaml$
    pgp: 'your-gpg-key-fingerprint'
```

## 2. Cài đặt ArgoCD

### Cài đặt ArgoCD trên Kubernetes
```bash
# Tạo namespace
kubectl create namespace argocd

# Cài đặt ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Đợi pods ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### Cấu hình ArgoCD với SOPS

#### Tùy chọn 1: Download SOPS và Kustomize (như trước)
Tạo ConfigMap chứa SOPS binary và cấu hình:

```yaml
# argocd-sops-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-sops-config
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
data:
  sops.sh: |
    #!/bin/bash
    # Download and install sops
    curl -L https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 -o /usr/local/bin/sops
    chmod +x /usr/local/bin/sops
    
    # Download and install kustomize with sops support
    curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
    mv kustomize /usr/local/bin/

#### Tùy chọn 2: Sử dụng SOPS và Kustomize có sẵn trên server

##### Tạo PersistentVolume cho tools binary
```yaml
# tools-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sops-tools-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /opt/sops-tools  # Đường dẫn trên node chứa sops và kustomize
    type: Directory
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sops-tools-pvc
  namespace: argocd
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
  volumeName: sops-tools-pv
```

##### Cấu trúc thư mục trên server
```bash
# Tạo thư mục và copy tools vào server
sudo mkdir -p /opt/sops-tools/bin

# Download và copy sops
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 -o /opt/sops-tools/bin/sops
sudo chmod +x /opt/sops-tools/bin/sops

# Download và copy kustomize
cd /tmp
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /opt/sops-tools/bin/
sudo chmod +x /opt/sops-tools/bin/kustomize

# Set permissions
sudo chown -R 999:999 /opt/sops-tools  # ArgoCD user ID
```

##### ConfigMap cho tùy chọn 2
```yaml
# argocd-sops-config-preinstalled.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-sops-config
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
data:
  sops.sh: |
    #!/bin/bash
    # Tools đã có sẵn, chỉ cần tạo symlink
    ln -sf /opt/sops-tools/bin/sops /usr/local/bin/sops
    ln -sf /opt/sops-tools/bin/kustomize /usr/local/bin/kustomize
    echo "SOPS and Kustomize tools ready from preinstalled location"
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-sops-gpg
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
type: Opaque
stringData:
  sops.asc: |
    -----BEGIN PGP PRIVATE KEY BLOCK-----
    # Paste your private GPG key here
    -----END PGP PRIVATE KEY BLOCK-----
```

### Patch ArgoCD Repository Server

#### Tùy chọn 1: Download tools (InitContainer)
```yaml
# argocd-repo-server-patch-download.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      initContainers:
      - name: sops-installer
        image: alpine:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            apk add --no-cache curl gnupg
            /scripts/sops.sh
            gpg --import /gpg-keys/sops.asc
        volumeMounts:
        - name: sops-config
          mountPath: /scripts
        - name: sops-gpg-keys
          mountPath: /gpg-keys
        - name: sops-tools
          mountPath: /usr/local/bin
      containers:
      - name: argocd-repo-server
        env:
        - name: SOPS_PGP_FP
          value: "your-gpg-key-fingerprint"
        volumeMounts:
        - name: sops-tools
          mountPath: /usr/local/bin
        - name: sops-gpg-keys
          mountPath: /home/argocd/.gnupg
      volumes:
      - name: sops-config
        configMap:
          name: argocd-sops-config
          defaultMode: 0755
      - name: sops-gpg-keys
        secret:
          secretName: argocd-sops-gpg
          defaultMode: 0600
      - name: sops-tools
        emptyDir: {}
```

#### Tùy chọn 2: Sử dụng tools có sẵn (PV Mount)
```yaml
# argocd-repo-server-patch-preinstalled.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      initContainers:
      - name: sops-installer
        image: alpine:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            apk add --no-cache gnupg
            /scripts/sops.sh
            gpg --import /gpg-keys/sops.asc
            echo "Tools setup completed"
        volumeMounts:
        - name: sops-config
          mountPath: /scripts
        - name: sops-gpg-keys
          mountPath: /gpg-keys
        - name: sops-tools-preinstalled
          mountPath: /opt/sops-tools
          readOnly: true
        - name: sops-tools
          mountPath: /usr/local/bin
      containers:
      - name: argocd-repo-server
        env:
        - name: SOPS_PGP_FP
          value: "your-gpg-key-fingerprint"
        - name: PATH
          value: "/usr/local/bin:/opt/sops-tools/bin:$(PATH)"
        volumeMounts:
        - name: sops-tools
          mountPath: /usr/local/bin
        - name: sops-gpg-keys
          mountPath: /home/argocd/.gnupg
        - name: sops-tools-preinstalled
          mountPath: /opt/sops-tools
          readOnly: true
      volumes:
      - name: sops-config
        configMap:
          name: argocd-sops-config
          defaultMode: 0755
      - name: sops-gpg-keys
        secret:
          secretName: argocd-sops-gpg
          defaultMode: 0600
      - name: sops-tools
        emptyDir: {}
      - name: sops-tools-preinstalled
        persistentVolumeClaim:
          claimName: sops-tools-pvc
```

#### Tùy chọn 3: Mount trực tiếp từ hostPath (Đơn giản hơn)
```yaml
# argocd-repo-server-patch-hostpath.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      initContainers:
      - name: sops-installer
        image: alpine:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            apk add --no-cache gnupg
            # Tạo symlinks cho tools có sẵn
            ln -sf /opt/sops-tools/bin/sops /usr/local/bin/sops
            ln -sf /opt/sops-tools/bin/kustomize /usr/local/bin/kustomize
            chmod +x /usr/local/bin/sops /usr/local/bin/kustomize
            gpg --import /gpg-keys/sops.asc
        volumeMounts:
        - name: sops-gpg-keys
          mountPath: /gpg-keys
        - name: sops-tools-host
          mountPath: /opt/sops-tools
          readOnly: true
        - name: sops-tools
          mountPath: /usr/local/bin
      containers:
      - name: argocd-repo-server
        env:
        - name: SOPS_PGP_FP
          value: "your-gpg-key-fingerprint"
        - name: PATH
          value: "/usr/local/bin:/opt/sops-tools/bin:$(PATH)"
        volumeMounts:
        - name: sops-tools
          mountPath: /usr/local/bin
        - name: sops-gpg-keys
          mountPath: /home/argocd/.gnupg
        - name: sops-tools-host
          mountPath: /opt/sops-tools
          readOnly: true
      volumes:
      - name: sops-gpg-keys
        secret:
          secretName: argocd-sops-gpg
          defaultMode: 0600
      - name: sops-tools
        emptyDir: {}
      - name: sops-tools-host
        hostPath:
          path: /opt/sops-tools
          type: Directory
```

Áp dụng patch theo tùy chọn bạn chọn:

#### Tùy chọn 1: Download tools
```bash
kubectl apply -f argocd-sops-config.yaml
kubectl patch deployment argocd-repo-server -n argocd --patch-file argocd-repo-server-patch-download.yaml
```

#### Tùy chọn 2: Sử dụng PV với tools có sẵn
```bash
# Tạo PV và PVC trước
kubectl apply -f tools-pv.yaml

# Áp dụng config và patch
kubectl apply -f argocd-sops-config-preinstalled.yaml
kubectl patch deployment argocd-repo-server -n argocd --patch-file argocd-repo-server-patch-preinstalled.yaml
```

#### Tùy chọn 3: Mount trực tiếp từ hostPath
```bash
# Chỉ cần patch deployment (không cần ConfigMap)
kubectl patch deployment argocd-repo-server -n argocd --patch-file argocd-repo-server-patch-hostpath.yaml
```

### So sánh các tùy chọn

| Tùy chọn | Ưu điểm | Nhược điểm | Phù hợp khi |
|----------|---------|-------------|-------------|
| **Download** | - Tự động, không phụ thuộc server<br>- Dễ setup | - Chậm khởi tạo<br>- Cần internet<br>- Tăng thời gian deploy | - Môi trường cloud<br>- Không có quyền truy cập server |
| **PV Mount** | - Nhanh khởi tạo<br>- Kiểm soát version<br>- Shared across nodes | - Phức tạp setup<br>- Cần quản lý PV | - Multi-node cluster<br>- Cần chia sẻ tools |
| **HostPath** | - Đơn giản nhất<br>- Nhanh nhất<br>- Dễ cập nhật tools | - Phụ thuộc node<br>- Không work với multi-node | - Single node<br>- Development environment<br>- On-premise |

## 3. Tạo cấu trúc Repository

### Cấu trúc thư mục
```
my-app/
├── .sops.yaml
├── kustomization.yaml
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── secrets.enc.yaml
│   │   └── config.enc.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── secrets.enc.yaml
│       └── config.enc.yaml
└── scripts/
    └── encrypt-secrets.sh
```

### Base Kustomization
```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: my-app
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
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
        image: nginx:1.20
        ports:
        - containerPort: 80
        envFrom:
        - secretRef:
            name: app-secrets
        - configMapRef:
            name: app-config
```

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
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
  - secrets.enc.yaml
  - config.enc.yaml

replicas:
  - name: my-app
    count: 1
```

## 4. Tạo và mã hóa Secrets

### Tạo file secrets gốc
```yaml
# overlays/development/secrets.yaml (file tạm)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:password@localhost:5432/myapp_dev"
  API_KEY: "dev-api-key-12345"
  JWT_SECRET: "dev-jwt-secret-67890"
```

### Mã hóa secrets
```bash
# Script để mã hóa secrets
#!/bin/bash
# scripts/encrypt-secrets.sh

# Mã hóa development secrets
sops --encrypt --in-place overlays/development/secrets.yaml
mv overlays/development/secrets.yaml overlays/development/secrets.enc.yaml

# Mã hóa production secrets
sops --encrypt --in-place overlays/production/secrets.yaml
mv overlays/production/secrets.yaml overlays/production/secrets.enc.yaml

# Mã hóa config files
sops --encrypt --in-place overlays/development/config.yaml
mv overlays/development/config.yaml overlays/development/config.enc.yaml
```

### File secrets đã mã hóa sẽ trông như thế này:
```yaml
# overlays/development/secrets.enc.yaml
apiVersion: v1
kind: Secret
metadata:
    name: app-secrets
type: Opaque
stringData:
    DATABASE_URL: ENC[AES256_GCM,data:encrypted_data_here,type:str]
    API_KEY: ENC[AES256_GCM,data:encrypted_data_here,type:str]
    JWT_SECRET: ENC[AES256_GCM,data:encrypted_data_here,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2023-12-01T10:00:00Z"
    mac: ENC[AES256_GCM,data:mac_here,type:str]
    pgp:
        - created_at: "2023-12-01T10:00:00Z"
          enc: |
            -----BEGIN PGP MESSAGE-----
            encrypted_key_here
            -----END PGP MESSAGE-----
          fp: your-gpg-key-fingerprint
    version: 3.8.1
```

## 5. Cấu hình ArgoCD Application với SOPS Plugin

### Tạo SOPS Generator Plugin
```yaml
# argocd-sops-plugin.yaml
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
      generate:
        command: [sh, -c]
        args:
          - |
            # Import GPG key
            gpg --import /home/argocd/.gnupg/sops.asc
            
            # Find and decrypt all .enc.yaml files
            find . -name "*.enc.yaml" -type f | while read -r file; do
              echo "Decrypting $file"
              sops --decrypt "$file" > "${file%.enc.yaml}.yaml"
            done
            
            # Run kustomize build
            kustomize build .
      discover:
        find:
          command: [sh, -c, "find . -name '*.enc.yaml' | head -1"]
```

### Cập nhật ArgoCD ConfigMap
```bash
# Áp dụng plugin
kubectl apply -f argocd-sops-plugin.yaml -n argocd

# Restart ArgoCD repo server
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

## 6. Tạo ArgoCD Application

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/my-app
    targetRevision: main
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
```

## 7. Upload lên GitHub

### Gitignore
```bash
# .gitignore
*.yaml
!*.enc.yaml
!kustomization.yaml
!base/*.yaml
.sops.yaml.bak
private.key
```

### Git commands
```bash
# Initialize git
git init
git add .
git commit -m "Initial commit with encrypted secrets"

# Add remote và push
git remote add origin https://github.com/your-username/my-app.git
git branch -M main
git push -u origin main
```

## 8. Deploy Application

```bash
# Áp dụng ArgoCD application
kubectl apply -f argocd-application.yaml

# Kiểm tra trạng thái
kubectl get applications -n argocd

# Sync application
kubectl patch application my-app-dev -n argocd --type merge --patch '{"operation":{"initiatedBy":{"username":"admin"},"sync":{}}}'
```

## 9. Xác minh triển khai

```bash
# Kiểm tra pods
kubectl get pods -n development

# Kiểm tra secrets đã được giải mã
kubectl get secret app-secrets -n development -o yaml

# Kiểm tra logs để đảm bảo env vars được load
kubectl logs deployment/my-app -n development
```

## 10. Quy trình làm việc hàng ngày

### Cập nhật secrets
```bash
# 1. Giải mã file
sops overlays/development/secrets.enc.yaml

# 2. Chỉnh sửa trong editor
# 3. File sẽ được tự động mã hóa lại khi save

# 4. Commit và push
git add .
git commit -m "Update secrets"
git push
```

### Thêm secrets mới
```bash
# Tạo file secrets mới
sops overlays/production/new-secrets.yaml

# Cập nhật kustomization.yaml
# Commit và push
```

## Lưu ý quan trọng

### Chọn tùy chọn phù hợp
- **Download tools**: Phù hợp cho môi trường cloud, không cần quyền admin server
- **PV Mount**: Tốt cho production, multi-node cluster, cần kiểm soát version tools
- **HostPath**: Đơn giản nhất cho development, single-node cluster

### Bảo mật và Performance
1. **Backup GPG key**: Lưu trữ private key ở nơi an toàn
2. **Rotation**: Định kỳ thay đổi GPG key và re-encrypt tất cả secrets
3. **Access Control**: Chỉ những người cần thiết mới có quyền truy cập GPG key
4. **Audit**: Log tất cả các hoạt động decrypt/encrypt
5. **Testing**: Test việc giải mã trên môi trường staging trước khi production
6. **Tools Version**: Với PV/HostPath mount, dễ dàng cập nhật version SOPS/Kustomize
7. **Cache**: Tools có sẵn giúp giảm thời gian khởi tạo ArgoCD repo server

### Quản lý Tools trên Server (cho tùy chọn 2 & 3)
```bash
# Cập nhật SOPS version
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 -o /opt/sops-tools/bin/sops
sudo chmod +x /opt/sops-tools/bin/sops

# Cập nhật Kustomize version
cd /tmp && curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /opt/sops-tools/bin/
sudo chmod +x /opt/sops-tools/bin/kustomize

# Restart ArgoCD repo server để áp dụng tools mới
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

## Troubleshooting

### Lỗi GPG key không tìm thấy
```bash
# Kiểm tra GPG key trong ArgoCD pod
kubectl exec -it deployment/argocd-repo-server -n argocd -- gpg --list-keys
```

### Lỗi SOPS không tìm thấy
```bash
# Kiểm tra SOPS binary (tùy chọn download)
kubectl exec -it deployment/argocd-repo-server -n argocd -- which sops
kubectl exec -it deployment/argocd-repo-server -n argocd -- sops --version

# Kiểm tra SOPS binary (tùy chọn mount)
kubectl exec -it deployment/argocd-repo-server -n argocd -- ls -la /opt/sops-tools/bin/
kubectl exec -it deployment/argocd-repo-server -n argocd -- /opt/sops-tools/bin/sops --version
```

### Lỗi permission denied
```bash
# Kiểm tra quyền truy cập GPG home directory
kubectl exec -it deployment/argocd-repo-server -n argocd -- ls -la /home/argocd/.gnupg

# Kiểm tra quyền truy cập tools directory (mount option)
kubectl exec -it deployment/argocd-repo-server -n argocd -- ls -la /opt/sops-tools/bin/
```

### Lỗi PV không mount được
```bash
# Kiểm tra PV status
kubectl get pv sops-tools-pv
kubectl get pvc sops-tools-pvc -n argocd

# Kiểm tra node có thư mục tools không
kubectl get nodes
# SSH vào node và kiểm tra
ls -la /opt/sops-tools/bin/
```

### Lỗi hostPath không tồn tại
```bash
# Kiểm tra ArgoCD repo server đang chạy trên node nào
kubectl get pods -n argocd -o wide | grep repo-server

# SSH vào node đó và kiểm tra
ssh node-name
ls -la /opt/sops-tools/bin/

# Nếu chưa có, tạo thư mục và copy tools
sudo mkdir -p /opt/sops-tools/bin
# ... copy tools như hướng dẫn ở trên
```

### Debug SOPS decryption
```bash
# Test decrypt một file cụ thể trong pod
kubectl exec -it deployment/argocd-repo-server -n argocd -- bash
cd /tmp
# Copy một file .enc.yaml để test
sops --decrypt your-file.enc.yaml
```