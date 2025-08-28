## Bước 13: Troubleshooting và Validation

### Lỗi thường gặp và cách khắc phục:

#### 1. "no key available" hoặc "decryption failed"
```bash
# Kiểm tra GPG key có sẵn
gpg --list-secret-keys

# Kiểm tra fingerprint trong .sops.yaml
cat .sops.yaml

# Import key nếu thiếu
gpg --import private-key.asc

# Test decrypt thủ công
sops -d overlays/dev/secrets.enc.yaml
```

#### 2. "plugin not found" - KSOPS
```bash
# Reinstall ksops plugin
curl -s https://raw.githubusercontent.com/viaduct-ai/kustomize-sops/master/scripts/install-ksops-archive.sh | bash

# Verify installation
kustomize config list-plugins

# Alternative: manual installation
mkdir -p ~/.config/kustomize/plugin/viaduct.ai/v1/sopssecretgenerator
# Download và extract plugin files
```

#### 3. Certificate validation errors
```bash
# Verify certificate validity
openssl x509 -in certs/server-cert.pem -text -noout

# Check expiry date
openssl x509 -in certs/server-cert.pem -enddate -noout

# Verify certificate chain
openssl verify -CAfile certs/ca-cert.pem certs/server-cert.pem

# Test SSL connection
openssl s_client -connect your-domain.com:443 -servername your-domain.com
```

#### 4. Kustomize build failures
```bash
# Validate syntax trước khi build
kustomize build --dry-run overlays/dev

# Check individual resources
kubectl --dry-run=client apply -f base/deployment.yaml

# Debug với verbose output
kustomize build --enable-alpha-plugins overlays/dev --load-restrictor=none -v=2
```

### Scripts kiểm tra tổng quan

```bash
# Script validation toàn diện
cat > scripts/validate-environment.sh << 'EOF'
#!/bin/bash
set -euo pipefail

ENVIRONMENT=${1:-dev}

echo "🔍 Validating ${ENVIRONMENT} environment..."

# 1. Check GPG keys
echo "✅ Checking GPG keys..."
if gpg --list-secret-keys | grep -q "$(cat .sops.yaml | grep pgp | cut -d"'" -f2)"; then
  echo "  ✓ GPG key available"
else
  echo "  # Hướng dẫn tạo Kustomize Template cho Nginx với SOPS và GPG

## Yêu cầu hệ thống

- `kubectl`
- `kustomize` 
- `sops`
- `gpg`
- `ksops` plugin cho kustomize

## Bước 1: Cài đặt các công cụ cần thiết

### Cài đặt SOPS
```bash
# MacOS
brew install sops

# Linux
curl -LO https://github.com/mozilla/sops/releases/download/v3.7.3/sops-v3.7.3.linux.amd64
sudo mv sops-v3.7.3.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops
```

### Cài đặt KSOPS Plugin
```bash
# Tải và cài đặt ksops
curl -s https://raw.githubusercontent.com/viaduct-ai/kustomize-sops/master/scripts/install-ksops-archive.sh | bash
```

## Bước 2: Tạo GPG Key

### Tạo GPG Key với thông số chi tiết

```bash
# Tạo GPG key mới - sẽ có interactive prompts
gpg --full-generate-key
```

**Các thông số cần điền:**

1. **Loại key**: Chọn `(1) RSA and RSA (default)`
2. **Kích thước key**: Chọn `4096` bits (bảo mật cao hơn)
3. **Thời gian hết hạn**: 
   - Chọn `1y` (1 năm) hoặc `0` (không hết hạn)
   - Khuyến nghị: `2y` cho production
4. **Xác nhận**: Nhập `y` để confirm

**Thông tin cá nhân:**
```
Real name: DevOps Team
Email address: devops@yourcompany.com
Comment: Kubernetes Secrets Encryption Key
```

**Passphrase**: Tạo mật khẩu mạnh (ít nhất 12 ký tự)

### Quản lý GPG Keys

```bash
# Liệt kê keys với fingerprint đầy đủ
gpg --list-secret-keys --keyid-format LONG

# Lấy fingerprint (sử dụng cho .sops.yaml)
gpg --list-secret-keys --with-fingerprint

# Export public key (để chia sẻ với team)
gpg --armor --export YOUR_KEY_ID > public-key.asc

# Export private key (giữ bí mật - chỉ cho backup)
gpg --armor --export-secret-keys YOUR_KEY_ID > private-key.asc
```

**Ví dụ output:**
```
/home/user/.gnupg/pubring.kbx
-----------------------------
sec   rsa4096/1234567890ABCDEF 2024-01-15 [SC] [expires: 2026-01-15]
      ABCD1234EFGH5678IJKL9012MNOP3456QRST7890
uid                 [ultimate] DevOps Team <devops@yourcompany.com>
ssb   rsa4096/FEDCBA0987654321 2024-01-15 [E] [expires: 2026-01-15]
```

Lưu ý fingerprint: `ABCD1234EFGH5678IJKL9012MNOP3456QRST7890`

## Bước 3: Tạo cấu trúc thư mục

```
nginx-kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── secrets.enc.yaml
│   │   └── nginx-dev.conf
│   └── prod/
│       ├── kustomization.yaml
│       ├── secrets.enc.yaml
│       └── nginx-prod.conf
└── .sops.yaml
```

## Bước 4: Tạo file cấu hình SOPS

Tạo file `.sops.yaml` trong thư mục gốc:

```yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    pgp: 'ABCD1234EFGH5678IJKL9012MNOP3456QRST7890'  # Thay bằng fingerprint của bạn
    encrypted_regex: '^(data|stringData)

## Bước 5: Tạo base resources

### base/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
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
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```

### base/service.yaml
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
  type: ClusterIP
```

### base/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
```

### base/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app.kubernetes.io/name: nginx
  app.kubernetes.io/managed-by: kustomize
```

## Bước 6: Tạo overlay cho development

### overlays/dev/nginx-dev.conf
```nginx
server {
    listen 80;
    server_name dev.example.com;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    
    location /api {
        proxy_pass http://backend-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Tạo secrets cho dev environment (Làm rõ workflow)

#### Bước 1: Di chuyển vào thư mục dev

```bash
cd overlays/dev
```

#### Bước 2: Tạo file secrets đơn giản

```bash
# Tạo file secrets hoàn chỉnh (không có SSL certificates)
cat > temp-dev-secrets.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: "dev_postgresql_password_123"
  db-username: "nginx_dev_user"
  db-host: "postgresql-dev.example.com" 
  api-key: "dev_api_key_abc123xyz"
  jwt-secret: "dev_jwt_secret_for_development"
EOF
```

#### Bước 3: Mã hóa file

```bash
# Mã hóa toàn bộ file manifest
sops -e temp-dev-secrets.yaml > secrets.enc.yaml

# Cleanup
rm temp-dev-secrets.yaml

# Verify
head -10 secrets.enc.yaml
```

### overlays/dev/kustomization.yaml

**Cấu hình chính xác để sử dụng secrets đã mã hóa:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nginx-dev

resources:
- ../../base
- secrets.enc.yaml  # File chứa toàn bộ Secret manifest đã mã hóa

configMapGenerator:
- name: nginx-config
  files:
  - default.conf=nginx-dev.conf
  behavior: replace

patchesStrategicMerge:
- |
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx
  spec:
    replicas: 2
    template:
      spec:
        containers:
        - name: nginx
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"

commonLabels:
  environment: development
```

**Quan trọng:** 
- **KHÔNG** sử dụng `generators` section
- **KHÔNG** cần SopsSecretGenerator  
- File `secrets.enc.yaml` được reference trực tiếp trong `resources`

## Bước 7: Tạo overlay cho production

### overlays/prod/nginx-prod.conf
```nginx
server {
    listen 80;
    server_name prod.example.com;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    
    location /api {
        proxy_pass http://backend-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### Tạo secrets cho prod environment

#### Bước 1: Di chuyển vào thư mục prod

```bash
cd overlays/prod
```

#### Bước 2: Tạo production secrets

```bash
cat > temp-prod-secrets.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: "super_secure_prod_password_xyz789"
  db-username: "nginx_prod_user"
  db-host: "postgresql-prod-cluster.internal.com"
  api-key: "prod_api_key_secure_abc123def456"
  jwt-secret: "production_jwt_secret_very_secure_key"
EOF
```

#### Bước 3: Mã hóa production secrets

```bash
# Mã hóa file
sops -e temp-prod-secrets.yaml > secrets.enc.yaml

# Cleanup an toàn
rm temp-prod-secrets.yaml

# Verify
head -10 secrets.enc.yaml
```

### overlays/prod/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nginx-prod

resources:
- ../../base
- secrets.enc.yaml  # File chứa production secrets đã mã hóa

configMapGenerator:
- name: nginx-config
  files:
  - default.conf=nginx-prod.conf
  behavior: replace

patchesStrategicMerge:
- |
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx
  spec:
    replicas: 3
    template:
      spec:
        containers:
        - name: nginx
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"

commonLabels:
  environment: production
```

## Bước 8: Cấu hình SopsSecretGenerator trong Kustomization

**Quan trọng:** Có 2 cách sử dụng SOPS với Kustomize:

### Cách 1: Sử dụng SopsSecretGenerator (Khuyến nghị)

File `secrets.enc.yaml` sẽ chỉ chứa data được mã hóa, không có metadata Kubernetes:

```yaml
# overlays/dev/secrets.enc.yaml - chỉ chứa data
db-password: ENC[AES256_GCM,data:8vw+Uf7w==,tag:example]
db-username: ENC[AES256_GCM,data:Kj9fNm4pL==,tag:example]
api-key: ENC[AES256_GCM,data:abc123xyz==,tag:example]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    lastmodified: "2024-01-15T10:30:45Z"
    mac: ENC[AES256_GCM,data:example]
    pgp:
        - created_at: "2024-01-15T10:30:45Z"
          enc: |
            -----BEGIN PGP MESSAGE-----
            hQGMA...
            -----END PGP MESSAGE-----
          fp: ABCD1234EFGH5678IJKL9012MNOP3456QRST7890
```

Trong `kustomization.yaml`:
```yaml
generators:
- generators.yaml  # File chứa SopsSecretGenerator config
```

File `generators.yaml`:
```yaml
apiVersion: viaduct.ai/v1
kind: SopsSecretGenerator
metadata:
  name: app-secrets-generator
spec:
  name: app-secrets
  type: Opaque
  files:
  - secrets.enc.yaml

---
apiVersion: viaduct.ai/v1  
kind: SopsSecretGenerator
metadata:
  name: ssl-certs-generator
spec:
  name: ssl-certificates
  type: kubernetes.io/tls
  files:
  - ssl-secrets.enc.yaml
```

### Cách 2: Mã hóa toàn bộ Secret manifest (Đang dùng hiện tại)

File `secrets.enc.yaml` chứa toàn bộ Kubernetes Secret manifest được mã hóa:

```yaml
# overlays/dev/secrets.enc.yaml - toàn bộ manifest được encrypt
apiVersion: ENC[AES256_GCM,data:Vm0=,tag:example]
kind: ENC[AES256_GCM,data:U2Vj,tag:example]
metadata:
    name: ENC[AES256_GCM,data:YXBw,tag:example]
type: ENC[AES256_GCM,data:T3Bh,tag:example]
stringData:
    db-password: ENC[AES256_GCM,data:8vw+Uf7w==,tag:example]
    api-key: ENC[AES256_GCM,data:Kj9fNm4pL==,tag:example]
# ... sops metadata
```

## Bước 10: Build và Deploy

### Build cho development
```bash
# Build và xem kết quả
kustomize build overlays/dev

# Build và apply trực tiếp  
kustomize build overlays/dev | kubectl apply -f -

# Verify deployment
kubectl get secrets -n nginx-dev
kubectl get pods -n nginx-dev
```

### Build cho production
```bash
# Build và xem kết quả
kustomize build overlays/prod

# Build và apply trực tiếp
kustomize build overlays/prod | kubectl apply -f -

# Verify deployment
kubectl get secrets -n nginx-prod
kubectl get pods -n nginx-prod
```

### Verify secrets đã được decrypt đúng
```bash
# Check secret content (sẽ thấy base64 encoded values)
kubectl get secret app-secrets -n nginx-dev -o yaml

# Decode để verify
kubectl get secret app-secrets -n nginx-dev -o jsonpath='{.data.db-password}' | base64 -d
```

## Bước 11: Quản lý Secrets - Workflow thực tế

### Chỉnh sửa secrets đã mã hóa

**Cách 1: Chỉnh sửa trực tiếp file đã mã hóa**
```bash
# SOPS sẽ tự động decrypt, mở editor, và encrypt lại
sops overlays/dev/secrets.enc.yaml

# Trong editor, bạn sẽ thấy nội dung đã decrypt:
# apiVersion: v1
# kind: Secret
# metadata:
#   name: app-secrets
# stringData:
#   db-password: "dev_postgresql_password_123"
#   api-key: "dev_api_key_abc123"

# Chỉnh sửa values, save và exit
# SOPS sẽ tự động mã hóa lại file
```

**Cách 2: Update một giá trị cụ thể**
```bash
# Update chỉ một field cụ thể
sops --set '["stringData"]["db-password"] "new_password_456"' overlays/dev/secrets.enc.yaml

# Update multiple values cùng lúc
sops --set '["stringData"]["api-key"] "new_api_key_xyz"' \
     --set '["stringData"]["db-password"] "new_db_pass"' \
     overlays/dev/secrets.enc.yaml
```

**Cách 3: Recreate hoàn toàn (khi có nhiều thay đổi)**
```bash
# 1. Decrypt existing file để lấy structure
sops -d overlays/dev/secrets.enc.yaml > temp-current-secrets.yaml

# 2. Chỉnh sửa temp file
nano temp-current-secrets.yaml

# 3. Re-encrypt
sops -e temp-current-secrets.yaml > overlays/dev/secrets.enc.yaml

# 4. Cleanup
rm temp-current-secrets.yaml
```

### Thêm Secret mới vào file existing

```bash
# 1. Decrypt current secrets
sops -d overlays/dev/secrets.enc.yaml > current-secrets.yaml

# 2. Tạo secret mới
cat > new-secret.yaml << EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: monitoring-secrets
type: Opaque
stringData:
  grafana-password: "$(openssl rand -base64 16)"
  prometheus-password: "$(openssl rand -base64 16)"
EOF

# 3. Merge files
cat current-secrets.yaml new-secret.yaml > merged-secrets.yaml

# 4. Re-encrypt merged file
sops -e merged-secrets.yaml > overlays/dev/secrets.enc.yaml

# 5. Cleanup
rm current-secrets.yaml new-secret.yaml merged-secrets.yaml
```

### Xem nội dung secrets (read-only)

```bash
# Xem toàn bộ nội dung đã decrypt
sops -d overlays/dev/secrets.enc.yaml

# Xem chỉ một Secret cụ thể
sops -d overlays/dev/secrets.enc.yaml | yq 'select(.metadata.name == "app-secrets")'

# Xem chỉ values
sops -d overlays/dev/secrets.enc.yaml | yq '.stringData'

# Extract một value cụ thể
sops -d --extract '["stringData"]["db-password"]' overlays/dev/secrets.enc.yaml
```

### Rotate certificates (không dùng trong bài này)

*Phần này được bỏ qua vì không sử dụng TLS certificates*

### Rotate GPG keys

```bash
# Tạo key mới
gpg --full-generate-key

# Lấy fingerprint key mới
gpg --list-secret-keys --with-fingerprint

# Cập nhật .sops.yaml với fingerprint mới  
sed -i.bak "s/OLD_FINGERPRINT/NEW_FINGERPRINT/g" .sops.yaml

# Re-encrypt tất cả secrets với key mới
find . -name "*.enc.yaml" -exec sops -r -i {} \;
```

### Backup GPG keys và secrets

```bash
# Backup GPG keys
gpg --armor --export-secret-keys > private-keys.asc
gpg --armor --export > public-keys.asc

# Backup SOPS config
cp .sops.yaml sops-config-backup.yaml

# Encrypted secrets đã an toàn để backup (đã được mã hóa)
tar -czf secrets-backup.tar.gz overlays/*/secrets.enc.yaml .sops.yaml
```

## Bước 12: CI/CD Integration (Simplified)

### GitLab CI example
```yaml
variables:
  KUSTOMIZE_VERSION: "4.5.7"
  SOPS_VERSION: "3.7.3"

before_script:
  # Import GPG private key từ CI variable
  - echo "$GPG_PRIVATE_KEY" | base64 -d | gpg --import
  
  # Install kustomize
  - curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
  - sudo mv kustomize /usr/local/bin/
  
  # Install SOPS
  - curl -LO "https://github.com/mozilla/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64"
  - chmod +x sops-v${SOPS_VERSION}.linux.amd64
  - sudo mv sops-v${SOPS_VERSION}.linux.amd64 /usr/local/bin/sops

deploy_dev:
  script:
    - kustomize build overlays/dev | kubectl apply -f -
  only:
    - develop

deploy_prod:
  script:
    - kustomize build overlays/prod | kubectl apply -f -
  only:
    - main
```https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
  - curl -LO "https://github.com/mozilla/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64"
  - chmod +x sops-v${SOPS_VERSION}.linux.amd64
  - mv sops-v${SOPS_VERSION}.linux.amd64 /usr/local/bin/sops

deploy_dev:
  script:
    - kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -
  only:
    - develop

deploy_prod:
  script:
    - kustomize build --enable-alpha-plugins overlays/prod | kubectl apply -f -
  only:
    - main
```

## Best Practices

1. **Bảo mật GPG Keys**: Không commit private keys vào Git
2. **Backup Keys**: Lưu trữ private keys an toàn
3. **Key Rotation**: Thường xuyên rotate GPG keys
4. **Access Control**: Chỉ chia sẻ public keys với những người cần thiết
5. **Environment Separation**: Sử dụng keys khác nhau cho từng environment
6. **Validation**: Luôn test build trước khi deploy

## Troubleshooting

### Lỗi thường gặp:

1. **"no key available"**: Kiểm tra GPG key đã được import
2. **"plugin not found"**: Cài đặt lại ksops plugin
3. **"decryption failed"**: Kiểm tra fingerprint trong .sops.yaml

### Debug commands:
```bash
# Kiểm tra GPG keys
gpg --list-keys

# Test SOPS decryption
sops -d overlays/dev/secrets.enc.yaml

# Validate kustomization
kustomize build --dry-run overlays/dev
```
```

**Lưu ý quan trọng:**
- Thay `ABCD1234EFGH5678IJKL9012MNOP3456QRST7890` bằng fingerprint thực tế từ bước 2
- File này xác định luật mã hóa cho các file `.enc.yaml`
- `encrypted_regex` chỉ mã hóa phần `data` và `stringData` của Kubernetes secrets



## Bước 5: Tạo base resources

### base/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
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
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: ssl-certs
          mountPath: /etc/ssl/certs
          readOnly: true
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: ssl-certs
        secret:
          secretName: ssl-certificates
```

### base/service.yaml
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
  type: ClusterIP
```

### base/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
```

### base/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app.kubernetes.io/name: nginx
  app.kubernetes.io/managed-by: kustomize
```

## Bước 6: Tạo overlay cho development

### overlays/dev/nginx-dev.conf
```nginx
server {
    listen 80;
    server_name dev.example.com;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    
    location /api {
        proxy_pass http://backend-service:8080;
    }
}
```

### Tạo secrets cho dev environment

Tạo file tạm thời `temp-secrets.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: "dev-password-123"
  api-key: "dev-api-key-xyz"

---
apiVersion: v1
kind: Secret
metadata:
  name: ssl-certificates
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    # Development SSL certificate content
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    # Development SSL private key content
    -----END PRIVATE KEY-----
```

Mã hóa file bằng SOPS:
```bash
sops -e temp-secrets.yaml > overlays/dev/secrets.enc.yaml
rm temp-secrets.yaml
```

### overlays/dev/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nginx-dev

resources:
- ../../base

generators:
- secrets.enc.yaml

configMapGenerator:
- name: nginx-config
  files:
  - default.conf=nginx-dev.conf
  behavior: replace

patchesStrategicMerge:
- |
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx
  spec:
    replicas: 2
    template:
      spec:
        containers:
        - name: nginx
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"

commonLabels:
  environment: development
```

## Bước 7: Tạo overlay cho production

### overlays/prod/nginx-prod.conf
```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name prod.example.com;
    
    ssl_certificate /etc/ssl/certs/tls.crt;
    ssl_certificate_key /etc/ssl/certs/tls.key;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    
    location /api {
        proxy_pass http://backend-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Tạo secrets cho prod environment

Tạo file tạm thời `temp-prod-secrets.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: "super-secure-prod-password"
  api-key: "production-api-key-abc123"

---
apiVersion: v1
kind: Secret
metadata:
  name: ssl-certificates
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    # Production SSL certificate content
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    # Production SSL private key content
    -----END PRIVATE KEY-----
```

Mã hóa file:
```bash
sops -e temp-prod-secrets.yaml > overlays/prod/secrets.enc.yaml
rm temp-prod-secrets.yaml
```

### overlays/prod/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nginx-prod

resources:
- ../../base

generators:
- secrets.enc.yaml

configMapGenerator:
- name: nginx-config
  files:
  - default.conf=nginx-prod.conf
  behavior: replace

patchesStrategicMerge:
- |
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx
  spec:
    replicas: 3
    template:
      spec:
        containers:
        - name: nginx
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"

commonLabels:
  environment: production
```

## Bước 8: Tạo SopsSecretGenerator

### overlays/dev/secrets.enc.yaml (sau khi mã hóa)
```yaml
apiVersion: viaduct.ai/v1
kind: SopsSecretGenerator
metadata:
  name: app-secrets-generator
spec:
  secretFrom:
  - name: app-secrets
    type: Opaque
    file: secrets.enc.yaml
```

## Bước 9: Build và Deploy

### Build cho development
```bash
# Build và xem kết quả
kustomize build --enable-alpha-plugins overlays/dev

# Build và apply trực tiếp
kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -
```

### Build cho production
```bash
# Build và xem kết quả
kustomize build --enable-alpha-plugins overlays/prod

# Build và apply trực tiếp
kustomize build --enable-alpha-plugins overlays/prod | kubectl apply -f -
```

## Bước 10: Quản lý Secrets

### Chỉnh sửa secrets
```bash
# Edit encrypted secrets
sops overlays/dev/secrets.enc.yaml

# Re-encrypt với key mới
sops -r -i overlays/dev/secrets.enc.yaml
```

### Rotate GPG keys
```bash
# Tạo key mới
gpg --full-generate-key

# Cập nhật .sops.yaml với fingerprint mới
# Re-encrypt tất cả secrets với key mới
find . -name "*.enc.yaml" -exec sops -r -i {} \;
```

## Bước 11: CI/CD Integration

### GitLab CI example
```yaml
variables:
  KUSTOMIZE_VERSION: "4.5.7"
  SOPS_VERSION: "3.7.3"

before_script:
  - echo "$GPG_PRIVATE_KEY" | base64 -d | gpg --import
  - curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
  - curl -LO "https://github.com/mozilla/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64"
  - chmod +x sops-v${SOPS_VERSION}.linux.amd64
  - mv sops-v${SOPS_VERSION}.linux.amd64 /usr/local/bin/sops

deploy_dev:
  script:
    - kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -
  only:
    - develop

deploy_prod:
  script:
    - kustomize build --enable-alpha-plugins overlays/prod | kubectl apply -f -
  only:
    - main
```

## Best Practices

1. **Bảo mật GPG Keys**: Không commit private keys vào Git
2. **Backup Keys**: Lưu trữ private keys an toàn
3. **Key Rotation**: Thường xuyên rotate GPG keys
4. **Access Control**: Chỉ chia sẻ public keys với những người cần thiết
5. **Environment Separation**: Sử dụng keys khác nhau cho từng environment
6. **Validation**: Luôn test build trước khi deploy

## Troubleshooting

### Lỗi thường gặp:

1. **"no key available"**: Kiểm tra GPG key đã được import
2. **"plugin not found"**: Cài đặt lại ksops plugin
3. **"decryption failed"**: Kiểm tra fingerprint trong .sops.yaml

### Debug commands:
```bash
# Kiểm tra GPG keys
gpg --list-keys

# Test SOPS decryption
sops -d overlays/dev/secrets.enc.yaml

# Validate kustomization
kustomize build --dry-run overlays/dev
```