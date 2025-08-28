# Hướng dẫn tích hợp GPG, SOPS và ArgoCD

## 1. Tổng quan về GPG (GNU Privacy Guard)

### 1.1 GPG là gì?
GPG (GNU Privacy Guard) là một công cụ mã hóa mã nguồn mở dựa trên tiêu chuẩn OpenPGP. GPG cho phép:
- Mã hóa và ký số dữ liệu
- Tạo và quản lý cặp khóa công khai/riêng tư
- Xác thực danh tính

### 1.2 Các thành phần chính của GPG
- **Public Key**: Khóa công khai dùng để mã hóa dữ liệu
- **Private Key**: Khóa riêng tư dùng để giải mã dữ liệu
- **Key ID**: Định danh duy nhất của khóa
- **Fingerprint**: Dấu vân tay của khóa để xác thực

### 1.3 Cài đặt GPG
```bash
# Ubuntu/Debian
sudo apt-get install gnupg

# CentOS/RHEL
sudo yum install gnupg2

# macOS
brew install gnupg
```

### 1.4 Tạo cặp khóa GPG
```bash
# Tạo khóa mới
gpg --gen-key

# Hoặc tạo với tùy chọn nâng cao
gpg --full-gen-key

# Liệt kê các khóa
gpg --list-keys
gpg --list-secret-keys

# Export khóa công khai
gpg --export --armor your-email@example.com > public-key.asc

# Export khóa riêng tư
gpg --export-secret-keys --armor your-email@example.com > private-key.asc
```

## 2. SOPS (Secrets OPerationS)

### 2.1 SOPS là gì?
SOPS là một công cụ mã hóa file được phát triển bởi Mozilla. SOPS hỗ trợ:
- Mã hóa các file YAML, JSON, ENV, INI, và BINARY
- Tích hợp với nhiều backend mã hóa (GPG, AWS KMS, Azure Key Vault, GCP KMS)
- Chỉ mã hóa values, giữ nguyên structure và keys

### 2.2 Cài đặt SOPS
```bash
# Tải từ GitHub releases
curl -LO https://github.com/mozilla/sops/releases/latest/download/sops-v3.7.3.linux

# Di chuyển vào PATH
sudo mv sops-v3.7.3.linux /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# Kiểm tra cài đặt
sops --version
```

### 2.3 Cấu hình SOPS với GPG
Tạo file `.sops.yaml` trong thư mục dự án:
```yaml
creation_rules:
  - path_regex: \.secrets\.yaml$
    pgp: 'YOUR_GPG_KEY_FINGERPRINT'
  - path_regex: secrets/.*\.yaml$
    pgp: 'YOUR_GPG_KEY_FINGERPRINT'
```

### 2.4 Sử dụng SOPS cơ bản
```bash
# Mã hóa file
sops -e -i secrets.yaml

# Giải mã và xem nội dung
sops -d secrets.yaml

# Chỉnh sửa file đã mã hóa
sops secrets.yaml

# Mã hóa với GPG key cụ thể
sops -e --pgp YOUR_GPG_FINGERPRINT secrets.yaml
```

## 3. SopsSecretGenerator cho Kustomize

### 3.1 SopsSecretGenerator là gì?
SopsSecretGenerator là một plugin Kustomize cho phép tự động giải mã SOPS files và tạo Kubernetes secrets.

### 3.2 Cài đặt SopsSecretGenerator
```bash
# Tạo thư mục plugins
mkdir -p ~/.config/kustomize/plugin/viaduct.ai/v1/sopssecretgenerator

# Tải plugin
curl -Lo ~/.config/kustomize/plugin/viaduct.ai/v1/sopssecretgenerator/SopsSecretGenerator \
  https://github.com/viaduct-ai/kustomize-sops/releases/latest/download/SopsSecretGenerator_1.0_linux_amd64

# Cấp quyền thực thi
chmod +x ~/.config/kustomize/plugin/viaduct.ai/v1/sopssecretgenerator/SopsSecretGenerator
```

### 3.3 Sử dụng SopsSecretGenerator
Tạo file `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

generators:
- sopssecret.yaml

resources:
- deployment.yaml
```

Tạo file `sopssecret.yaml`:
```yaml
apiVersion: viaduct.ai/v1
kind: SopsSecretGenerator
metadata:
  name: app-secrets
spec:
  secretFrom:
  - file: secrets.enc.yaml
  type: Opaque
```

## 4. Sử dụng SOPS với Kustomize (Standalone)

### 4.1 Chuẩn bị GPG key cho Kustomize

#### Bước 1: Export GPG key cho local development
```bash
# Export private key để sử dụng local
gpg --export-secret-keys --armor YOUR_KEY_ID > local-gpg-key.asc

# Import vào keyring local (nếu cần)
gpg --import local-gpg-key.asc
```

### 4.2 Build và Deploy với Kustomize

#### Bước 1: Build Kustomize với SOPS
```bash
# Build và xem output
kustomize build --enable-alpha-plugins overlays/dev

# Build và apply trực tiếp
kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -

# Hoặc save ra file
kustomize build --enable-alpha-plugins overlays/dev > nginx-app.yaml
kubectl apply -f nginx-app.yaml
```

#### Bước 2: Kiểm tra deployment
```bash
# Kiểm tra namespace
kubectl get ns nginx-dev

# Kiểm tra pods
kubectl get pods -n nginx-dev

# Kiểm tra services
kubectl get svc -n nginx-dev

# Test nginx service
kubectl port-forward -n nginx-dev svc/nginx-service 8080:80

# Trong terminal khác
curl http://localhost:8080
curl http://localhost:8080/health
```

### 4.3 Quản lý Secrets

#### Xem secrets đã được tạo
```bash
# Liệt kê secrets
kubectl get secrets -n nginx-dev

# Xem nội dung secret (đã được giải mã)
kubectl get secret app-secret -n nginx-dev -o yaml

# Decode base64 value
kubectl get secret app-secret -n nginx-dev -o jsonpath='{.data.config}' | base64 -d
```

#### Cập nhật secrets
```bash
# Chỉnh sửa file secrets đã mã hóa
sops secrets/app-secrets.yaml

# Sau khi sửa, rebuild và redeploy
kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -
```

## 5. Ví dụ thực tế chi tiết

### 5.1 Tạo dự án nginx demo

#### Bước 1: Khởi tạo dự án
```bash
mkdir nginx-sops-demo
cd nginx-sops-demo

# Tạo cấu trúc thư mục
mkdir -p {base,overlays/dev,overlays/prod,secrets}
```

#### Bước 2: Tạo GPG key cho demo
```bash
# Tạo key với thông tin demo
cat > gpg-key-config <<EOF
Key-Type: RSA
Key-Length: 2048
Subkey-Type: RSA
Subkey-Length: 2048
Name-Real: Nginx Demo
Name-Email: nginx-demo@example.com
Expire-Date: 1y
%no-protection
EOF

gpg --batch --generate-key gpg-key-config

# Lấy key fingerprint
GPG_FINGERPRINT=$(gpg --list-keys --with-colons nginx-demo@example.com | grep fpr | head -1 | cut -d: -f10)
echo "GPG Fingerprint: $GPG_FINGERPRINT"
```

#### Bước 3: Tạo file secrets
```bash
# Tạo file secrets.yaml
cat > secrets/app-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  config: YXBwX25hbWU9bmdpbngtZGVtbwphcHBfZW52PXByb2R1Y3Rpb24K  # app_name=nginx-demo\napp_env=production
EOF

# Tạo file cấu hình SOPS
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: secrets/.*\.yaml$
    pgp: '$GPG_FINGERPRINT'
EOF

# Mã hóa file secrets
sops -e -i secrets/app-secrets.yaml
```

#### Bước 4: Tạo Kustomize configuration

**base/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
          protocol: TCP
        env:
        - name: APP_CONFIG
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: config
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: ClusterIP
---
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
        
        location /health {
            return 200 "OK";
            add_header Content-Type text/plain;
        }
    }
```

**base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
```

**overlays/dev/sopssecret.yaml:**
```yaml
apiVersion: viaduct.ai/v1
kind: SopsSecretGenerator
metadata:
  name: app-secret
spec:
  secretFrom:
  - file: ../../secrets/app-secrets.yaml
  type: Opaque
```

**overlays/dev/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nginx-dev

resources:
- ../../base

generators:
- sopssecret.yaml

patchesStrategicMerge:
- deployment-patch.yaml
```

**overlays/dev/deployment-patch.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: nginx
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
        env:
        - name: ENVIRONMENT
          value: "development"
```

### 5.2 Deploy và kiểm tra ứng dụng nginx

#### Bước 1: Tạo namespace và deploy
```bash
# Tạo namespace
kubectl create namespace nginx-dev

# Build và deploy với Kustomize
kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -

# Kiểm tra deployment status
kubectl get all -n nginx-dev
```

#### Bước 2: Kiểm tra và test nginx
```bash
# Kiểm tra pods đang chạy
kubectl get pods -n nginx-dev -w

# Kiểm tra logs của nginx
kubectl logs -n nginx-dev deployment/nginx-app

# Kiểm tra secrets đã được tạo và giải mã
kubectl get secrets -n nginx-dev
kubectl get secret app-secret -n nginx-dev -o yaml

# Port forward để test local
kubectl port-forward -n nginx-dev svc/nginx-service 8080:80

# Test trong terminal khác
curl http://localhost:8080
curl http://localhost:8080/health

# Kiểm tra environment variables trong container
kubectl exec -n nginx-dev deployment/nginx-app -- env | grep APP_CONFIG
```

#### Bước 3: Test cập nhật secrets
```bash
# Chỉnh sửa secrets
sops secrets/app-secrets.yaml

# Thêm thông tin mới, ví dụ:
# data:
#   config: <new-base64-encoded-config>
#   api_key: <base64-encoded-api-key>

# Rebuild và redeploy
kustomize build --enable-alpha-plugins overlays/dev | kubectl apply -f -

# Kiểm tra pod restart và nhận config mới
kubectl get pods -n nginx-dev
kubectl rollout status deployment/nginx-app -n nginx-dev
```

## 6. Best Practices

### 6.1 Bảo mật
- Luôn backup GPG private keys một cách an toàn
- Sử dụng passphrase mạnh cho GPG keys
- Rotate GPG keys định kỳ
- Không commit GPG private keys vào Git
- Sử dụng separate keys cho từng environment

### 6.2 Quản lý
- Sử dụng `.sops.yaml` để cấu hình rules cho từng path
- Tạo CI/CD pipeline để validate SOPS files
- Monitor việc truy cập secrets
- Implement proper RBAC cho ArgoCD

### 6.3 Troubleshooting

#### Lỗi thường gặp:
```bash
# Kiểm tra GPG keys có sẵn
gpg --list-keys
gpg --list-secret-keys

# Test SOPS decryption locally
sops -d secrets/app-secrets.yaml

# Kiểm tra Kustomize build với plugins
kustomize build --enable-alpha-plugins overlays/dev

# Debug khi secret không được tạo
kubectl describe secret app-secret -n nginx-dev

# Kiểm tra pod logs nếu nginx không start
kubectl logs -n nginx-dev deployment/nginx-app

# Kiểm tra sự kiện trong namespace
kubectl get events -n nginx-dev --sort-by=.metadata.creationTimestamp
```

#### Các lệnh hữu ích:
```bash
# Xem cấu hình nginx trong container
kubectl exec -n nginx-dev deployment/nginx-app -- cat /etc/nginx/conf.d/default.conf

# Test nginx config syntax
kubectl exec -n nginx-dev deployment/nginx-app -- nginx -t

# Reload nginx config sau khi update
kubectl exec -n nginx-dev deployment/nginx-app -- nginx -s reload

# Kiểm tra network connectivity
kubectl exec -n nginx-dev deployment/nginx-app -- wget -qO- localhost/health
```

## 7. Kết luận

Việc sử dụng GPG và SOPS với Kustomize mang lại:
- **Bảo mật cao**: Secrets được mã hóa end-to-end
- **Quản lý đơn giản**: Secrets có thể được quản lý trong Git một cách an toàn
- **Linh hoạt**: Dễ dàng tạo nhiều environment với secrets khác nhau
- **Audit trail**: Tất cả thay đổi đều được track trong Git
- **Nginx HTTP**: Ứng dụng nginx đơn giản chỉ chạy HTTP, dễ test và debug

Workflow cơ bản:
1. Tạo GPG key để mã hóa
2. Sử dụng SOPS để mã hóa file secrets
3. Kustomize với SopsSecretGenerator tự động giải mã và tạo Kubernetes secrets
4. Nginx deployment sử dụng secrets như ConfigMap thông thường
5. Test và verify nginx hoạt động đúng trên HTTP port 80

Hệ thống này phù hợp cho việc phát triển và testing các ứng dụng web đơn giản với secrets được bảo mật.