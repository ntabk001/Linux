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

#### Bước 1: Tạo Production certificates

```bash
# Di chuyển vào thư mục prod
cd overlays/prod

# Cho production, sử dụng certificates thực từ CA hoặc Let's Encrypt
# Ví dụ copy từ Let's Encrypt
cp /etc/letsencrypt/live/prod.example.com/fullchain.pem ./tls.crt
cp /etc/letsencrypt/live/prod.example.com/privkey.pem ./tls.key

# Hoặc tạo production-grade self-signed (không khuyến khích cho prod thực)
openssl genrsa -out tls.key 4096
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/C=VN/ST=HCM/L=HoChiMinh/O=YourCompany/OU=Production/CN=prod.example.com" \
  -addext "subjectAltName=DNS:prod.example.com,DNS:*.prod.example.com"
```

#### Bước 2: Tạo production secrets với security cao

Tạo file tạm thời `temp-prod-secrets.yaml`:

```bash
cat > temp-prod-secrets.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  # Database credentials - production grade
  db-password: "$(openssl rand -base64 32)"
  db-username: "nginx_prod_user"
  db-host: "postgresql-prod-cluster.internal.com"
  db-name: "nginx_production"
  
  # API credentials với entropy cao
  api-key: "prod_$(openssl rand -hex 32)"
  jwt-secret: "$(openssl rand -base64 64)"
  
  # Monitoring và logging
  datadog-api-key: "$(openssl rand -hex 32)"
  sentry-dsn: "https://$(openssl rand -hex 8)@sentry.io/production"
  
  # External services
  redis-password: "$(openssl rand -base64 32)"
  rabbitmq-password: "$(openssl rand -base64 32)"
  
---
apiVersion: v1
kind: Secret
metadata:
  name: ssl-certificates
type: kubernetes.io/tls
stringData:
  tls.crt: |
$(cat tls.crt | sed 's/^/    /')
  tls.key: |
$(cat tls.key | sed 's/^/    /')

---
apiVersion: v1
kind: Secret
metadata:
  name: nginx-auth
type: Opaque
stringData:
  .htpasswd: |
$(htpasswd -nb admin "$(openssl rand -base64 20)" | sed 's/^/    /')
$(htpasswd -nb monitoring "$(openssl rand -base64 20)" | sed 's/^/    /')

---
apiVersion: v1
kind: Secret  
metadata:
  name: backup-credentials
type: Opaque
stringData:
  aws-access-key-id: "AKIA$(openssl rand -hex 8)"
  aws-secret-access-key: "$(openssl rand -base64 30)"
  s3-bucket: "nginx-prod-backups-$(openssl rand -hex 4)"
EOF
```

#### Bước 3: Mã hóa production secrets

```bash
# Mã hóa với SOPS
sops -e temp-prod-secrets.yaml > secrets.enc.yaml

# Secure cleanup
shred -vfz -n 3 temp-prod-secrets.yaml tls.crt tls.key 2>/dev/null || rm -f temp-prod-secrets.yaml tls.crt tls.key

# Verify encryption
echo "Production secrets encrypted successfully:"
head -20 secrets.enc.yaml
```

#### Bước 4: Lưu trữ passwords an toàn

**Tạo file ghi nhớ passwords (chỉ cho dev/staging):**

```bash
# CHỈ cho development - KHÔNG làm với production
cat > ../DEV_PASSWORDS_REFERENCE.md << 'EOF'
# Development Passwords Reference
# ⚠️  CHỈ sử dụng cho development environment
# ⚠️  KHÔNG commit file này vào Git

## Database
- Username: nginx_dev_user  
- Password: dev_postgresql_password_123
- Host: postgresql-dev.example.com

## Nginx Basic Auth
- admin : dev_admin_pass
- developer : dev_dev_pass

## API Keys
- Generated automatically with openssl rand

## Notes
- Production passwords được generate random và store trong SOPS
- Liên hệ DevOps team để access production credentials
- Sử dụng `sops -d secrets.enc.yaml` để view encrypted values
EOF
```

**Production Security Notes:**
```bash
# Tạo security checklist cho production
cat > ../PROD_SECURITY_CHECKLIST.md << 'EOF'
# Production Security Checklist

## ✅ Pre-deployment
- [ ] All passwords generated với entropy cao (>= 32 bytes)
- [ ] SSL certificates từ trusted CA (Let's Encrypt/Commercial CA)
- [ ] Certificate expiry monitoring setup
- [ ] Backup credentials tested và stored securely
- [ ] Access control policies reviewed

## ✅ Post-deployment  
- [ ] Secrets rotation schedule established
- [ ] Monitoring alerts configured
- [ ] Backup verification completed
- [ ] Security scan passed
- [ ] Compliance requirements met

## 🔐 Emergency Access
- Contact: devops@yourcompany.com
- Backup GPG key location: secure vault
- Certificate renewal process: automated via cert-manager
EOF
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

### Rotate certificates

```bash
# Script tự động renew certificates
cat > scripts/renew-certs.sh << 'EOF'
#!/bin/bash
set -euo pipefail

ENVIRONMENT=${1:-dev}
DOMAIN=${2:-${ENVIRONMENT}.example.com}

echo "Renewing certificates for ${ENVIRONMENT} environment..."

case $ENVIRONMENT in
  "dev")
    # Self-signed cho development
    openssl genrsa -out temp-key.pem 4096
    openssl req -new -x509 -key temp-key.pem -out temp-cert.pem -days 365 \
      -subj "/C=VN/ST=HCM/L=HoChiMinh/O=YourCompany/OU=DevOps/CN=${DOMAIN}"
    ;;
  "prod")
    # Let's Encrypt cho production
    certbot renew --cert-name ${DOMAIN}
    cp /etc/letsencrypt/live/${DOMAIN}/fullchain.pem temp-cert.pem
    cp /etc/letsencrypt/live/${DOMAIN}/privkey.pem temp-key.pem
    ;;
esac

# Update secrets file
cd overlays/${ENVIRONMENT}

# Tạo temporary secrets với cert mới
cat > temp-cert-update.yaml << CERTEOF
apiVersion: v1
kind: Secret
metadata:
  name: ssl-certificates
type: kubernetes.io/tls
stringData:
  tls.crt: |
$(cat ../../temp-cert.pem | sed 's/^/    /')
  tls.key: |
$(cat ../../temp-key.pem | sed 's/^/    /')
CERTEOF

# Merge với existing secrets
sops -d secrets.enc.yaml > current-secrets.yaml
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' \
  current-secrets.yaml temp-cert-update.yaml > updated-secrets.yaml

# Re-encrypt
sops -e updated-secrets.yaml > secrets.enc.yaml

# Cleanup
rm -f current-secrets.yaml temp-cert-update.yaml updated-secrets.yaml
rm -f ../../temp-cert.pem ../../temp-key.pem

echo "Certificates updated for ${ENVIRONMENT} environment"
EOF

chmod +x scripts/renew-certs.sh
```

### Rotate GPG keys

```bash
# Script rotate GPG keys
cat > scripts/rotate-gpg-keys.sh << 'EOF'
#!/bin/bash
set -euo pipefail

OLD_KEY_ID=${1}
NEW_KEY_ID=${2}

echo "Rotating GPG keys from ${OLD_KEY_ID} to ${NEW_KEY_ID}..."

# Update .sops.yaml
sed -i.bak "s/${OLD_KEY_ID}/${NEW_KEY_ID}/g" .sops.yaml

# Re-encrypt all secrets with new key
find . -name "*.enc.yaml" | while read file; do
  echo "Re-encrypting ${file}..."
  sops -r -i "${file}"
done

echo "GPG key rotation completed!"
echo "Don't forget to:"
echo "1. Distribute new public key to team members"
echo "2. Update CI/CD systems with new private key"
echo "3. Revoke old key after verification"
EOF

chmod +x scripts/rotate-gpg-keys.sh
```

### Backup và Recovery procedures

```bash
# Script backup secrets
cat > scripts/backup-secrets.sh << 'EOF'
#!/bin/bash
set -euo pipefail

BACKUP_DIR="backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "${BACKUP_DIR}"

echo "Creating backup in ${BACKUP_DIR}..."

# Backup GPG keys
gpg --armor --export-secret-keys > "${BACKUP_DIR}/private-keys.asc"
gpg --armor --export > "${BACKUP_DIR}/public-keys.asc"

# Backup configuration
cp .sops.yaml "${BACKUP_DIR}/"

# Backup encrypted secrets (safe to store)
find . -name "*.enc.yaml" -exec cp --parents {} "${BACKUP_DIR}" \;

# Create restore instructions
cat > "${BACKUP_DIR}/RESTORE.md" << RESTOREEOF
# Restore Instructions

## 1. Import GPG Keys
\`\`\`bash
gpg --import private-keys.asc
gpg --import public-keys.asc
\`\`\`

## 2. Restore SOPS config  
\`\`\`bash
cp .sops.yaml /path/to/project/
\`\`\`

## 3. Restore encrypted files
\`\`\`bash
# Copy all .enc.yaml files back to project
\`\`\`

## 4. Test decryption
\`\`\`bash
sops -d overlays/dev/secrets.enc.yaml
\`\`\`
RESTOREEOF

echo "Backup completed: ${BACKUP_DIR}"
echo "⚠️  Store private-keys.asc in secure location!"
EOF

chmod +x scripts/backup-secrets.sh
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