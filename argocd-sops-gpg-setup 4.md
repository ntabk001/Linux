# ArgoCD + SOPS + GPG Integration Setup

## 1. Cài đặt ArgoCD với Custom Configuration

### Tạo namespace và cài đặt ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Custom ArgoCD ConfigMap với SOPS Secret Generator plugin
```yaml
# argocd-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  configManagementPlugins: |
    - name: sops-secret-generator
      generate:
        command: [sh, -c]
        args:
          - |
            echo "Using SOPS Secret Generator..."
            # SOPS Secret Generator sẽ tự động detect và decrypt các file .sops.yaml
            # Plugin sẽ generate secrets từ encrypted files
            find . -name "*.sops.yaml" -exec echo "Processing {}" \;
            sops-secret-generator
```

## 2. Tạo GPG Key và Secret

### Tạo GPG key cho môi trường dev
```bash
# Tạo GPG key
gpg --batch --full-generate-key <<EOF
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: ArgoCD Dev
Name-Email: argocd@example.com
Expire-Date: 1y
Passphrase: 
%commit
%echo done
EOF

# Export GPG key
gpg --armor --export-secret-keys argocd@example.com > gpg-private-key.asc
gpg --armor --export argocd@example.com > gpg-public-key.asc
```

### Tạo Kubernetes Secret cho GPG key
```bash
kubectl create secret generic gpg-keys \
  --from-file=private.asc=gpg-private-key.asc \
  --from-file=public.asc=gpg-public-key.asc \
  -n argocd
```

## 3. Custom ArgoCD Repo Server với SOPS

### ArgoCD Repo Server Deployment với SOPS
```yaml
# argocd-repo-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-repo-server
    spec:
      initContainers:
      - name: install-sops-secret-generator
        image: alpine:3.18
        command:
        - sh
        - -c
        - |
          apk add --no-cache curl gnupg git
          # Install SOPS
          curl -LO https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
          chmod +x sops-v3.8.1.linux.amd64
          mv sops-v3.8.1.linux.amd64 /custom-tools/sops
          # Install SOPS Secret Generator
          curl -LO https://github.com/goabout/kustomize-sopssecretgenerator/releases/latest/download/SopsSecretGenerator.so
          mv SopsSecretGenerator.so /custom-tools/
        volumeMounts:
        - name: custom-tools
          mountPath: /custom-tools
      containers:
      - name: argocd-repo-server
        image: quay.io/argoproj/argocd:v2.8.4
        command:
        - sh
        - -c
        - |
          # Import GPG keys
          gpg --import /gpg-keys/private.asc
          gpg --import /gpg-keys/public.asc
          # Start repo server
          /var/run/argocd/argocd-repo-server
        env:
        - name: GNUPGHOME
          value: /tmp/.gnupg
        - name: XDG_CONFIG_HOME
          value: /tmp/.config
        - name: KUSTOMIZE_PLUGIN_HOME
          value: /custom-tools
        volumeMounts:
        - name: custom-tools
          mountPath: /usr/local/bin/sops
          subPath: sops
        - name: custom-tools
          mountPath: /usr/local/lib/kustomize/plugin/goabout.com/v1beta1/sopssecretgenerator/SopsSecretGenerator.so
          subPath: SopsSecretGenerator.so
        - name: gpg-keys
          mountPath: /gpg-keys
          readOnly: true
        - name: tmp
          mountPath: /tmp
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8081
          initialDelaySeconds: 60  # Tăng timeout
          periodSeconds: 30
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8081
          initialDelaySeconds: 30  # Tăng timeout
          periodSeconds: 10
          timeoutSeconds: 5
      volumes:
      - name: custom-tools
        emptyDir: {}
      - name: gpg-keys
        secret:
          secretName: gpg-keys
      - name: tmp
        emptyDir: {}
```

## 4. ArgoCD Application Configuration

### Tạo Application sử dụng SOPS plugin
```yaml
# app-with-sops.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo
    targetRevision: HEAD
    path: deploy/dev
    plugin:
      name: sops-secret-generator
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 5. Ví dụ Project Structure

### Cấu trúc thư mục project với SOPS Secret Generator
```
your-repo/
├── deploy/
│   └── dev/
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── secrets.sops.yaml  # File đã mã hóa với SOPS
└── .sops.yaml                 # SOPS config
```

### File kustomization.yaml sử dụng SOPS Secret Generator
```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

generators:
- secrets.sops.yaml

transformers:
- goabout.com/v1beta1/sopssecretgenerator
```

### File secrets.sops.yaml (template cho SOPS Secret Generator)
```yaml
# secrets.sops.yaml - File này sẽ được encrypt bằng SOPS
apiVersion: goabout.com/v1beta1
kind: SopsSecretGenerator
metadata:
  name: app-secrets-generator
spec:
  secretName: app-secrets
  namespace: default
  data:
    database-url: postgres://user:pass@db.example.com/db
    api-key: my_super_secret_api_key
```

### File .sops.yaml (cấu hình SOPS)
```yaml
creation_rules:
  - path_regex: \.sops\.yaml$
    pgp: "YOUR_GPG_KEY_FINGERPRINT"
```

### Mã hóa file secrets với SOPS Secret Generator
```bash
# Mã hóa file secrets.sops.yaml
sops -e secrets.sops.yaml > secrets.sops.yaml.tmp
mv secrets.sops.yaml.tmp secrets.sops.yaml

# Hoặc in-place encryption
sops -e -i secrets.sops.yaml
```

## 6. Apply Configuration

### Deploy custom ArgoCD với SOPS support
```bash
# Apply custom repo server
kubectl apply -f argocd-repo-server.yaml -n argocd

# Apply configmap
kubectl apply -f argocd-cm.yaml -n argocd

# Restart ArgoCD components
kubectl rollout restart deployment/argocd-repo-server -n argocd
kubectl rollout restart deployment/argocd-server -n argocd

# Wait for pods ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-repo-server -n argocd
```

### Tạo và deploy application
```bash
# Deploy application
kubectl apply -f app-with-sops.yaml -n argocd
```

## 7. Verification và Troubleshooting

### Kiểm tra logs
```bash
# Check repo server logs
kubectl logs deployment/argocd-repo-server -n argocd

# Check application sync status
kubectl get application my-app-dev -n argocd -o yaml
```

### Test GPG decryption trong pod
```bash
# Exec vào repo server pod
kubectl exec -it deployment/argocd-repo-server -n argocd -- bash

# Test SOPS decryption
sops -d /path/to/secrets.enc.yaml
```

## 8. Ví dụ Project Dev - Simple Web App

### Cấu trúc hoàn chỉnh project
```
my-app-repo/
├── src/                     # Source code
│   ├── app.js
│   ├── package.json
│   └── Dockerfile
├── deploy/
│   └── dev/
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       └── secrets.sops.yaml    # Encrypted secrets
├── .sops.yaml
└── README.md
```

### Source Code - Simple Express App

#### package.json
```json
{
  "name": "my-simple-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

#### app.js
```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from my simple app!',
    database: process.env.DATABASE_URL ? 'Connected' : 'Not configured',
    apiKey: process.env.API_KEY ? 'Loaded' : 'Missing',
    environment: process.env.NODE_ENV || 'development'
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

#### Dockerfile
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

### Kubernetes Manifests

#### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-simple-app
  labels:
    app: my-simple-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-simple-app
  template:
    metadata:
      labels:
        app: my-simple-app
    spec:
      containers:
      - name: my-simple-app
        image: my-simple-app:v1.0.0  # Build và push lên registry
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: environment
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: port
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

#### service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-simple-app-service
spec:
  selector:
    app: my-simple-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
```

#### configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  environment: "development"
  port: "3000"
  log-level: "debug"
```

#### secrets.sops.yaml (TRƯỚC KHI MÃ HÓA)
```yaml
# secrets.sops.yaml - File này sẽ được mã hóa bằng SOPS
apiVersion: goabout.com/v1beta1
kind: SopsSecretGenerator
metadata:
  name: app-secrets-generator
spec:
  secretName: app-secrets
  namespace: default
  type: Opaque
  data:
    database-url: postgres://devuser:devpass123@postgres-dev.internal:5432/myappdb
    api-key: dev-api-key-12345-abcdef
    jwt-secret: super-secret-jwt-signing-key-for-dev
    redis-password: redis-dev-password-789
```

#### kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: default

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

generators:
- secrets.sops.yaml
```

## 9. Setup và Deploy Process

### Bước 1: Build và push Docker image
```bash
# Build image
docker build -t your-registry.com/my-simple-app:v1.0.0 .

# Push to registry
docker push your-registry.com/my-simple-app:v1.0.0
```

### Bước 2: Encrypt secrets với GPG
```bash
# Trong thư mục project
cd deploy/dev/

# Encrypt secrets file
sops -e -i secrets.sops.yaml

# Verify encryption
cat secrets.sops.yaml  # Should show encrypted content
```

### Bước 3: Commit và push lên GitHub
```bash
git add .
git commit -m "Add encrypted secrets for dev environment"
git push origin main
```

### Bước 4: Deploy ArgoCD Application
```yaml
# my-simple-app-argocd.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-simple-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/my-app-repo
    targetRevision: HEAD
    path: deploy/dev
    plugin:
      name: sops-secret-generator
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

```bash
kubectl apply -f my-simple-app-argocd.yaml -n argocd
```

### Test Application
```bash
# Port forward để test app
kubectl port-forward service/my-simple-app-service 8080:80

# Test endpoint
curl http://localhost:8080
# Response:
# {
#   "message": "Hello from my simple app!",
#   "database": "Connected",
#   "apiKey": "Loaded", 
#   "environment": "development"
# }

# Test health endpoint
curl http://localhost:8080/health
# Response: {"status":"healthy"}
```

### Kiểm tra secrets đã được decrypt
```bash
# Check generated secrets
kubectl get secret app-secrets -o yaml

# Verify secret values (base64 decoded)
kubectl get secret app-secrets -o jsonpath='{.data.database-url}' | base64 -d
```

## 11. Workflow Development

### Quy trình làm việc với SOPS Secret Generator
1. **Developer**: Tạo/sửa secrets.sops.yaml với plaintext values
2. **Encrypt**: `sops -e -i secrets.sops.yaml` (encrypt in-place)
3. **Commit**: Push secrets.sops.yaml đã mã hóa lên GitHub  
4. **Deploy**: ArgoCD + SOPS Secret Generator tự động decrypt và tạo K8s secrets
5. **K8s**: Secrets được generate và apply vào cluster

### Script helper cho SOPS Secret Generator
```bash
#!/bin/bash
# encrypt-sops-secrets.sh
for file in $(find . -name "*.sops.yaml" -not -path "*/.*"); do
    echo "Encrypting $file"
    # Check if already encrypted
    if ! grep -q "sops:" "$file"; then
        sops -e -i "$file"
        echo "✓ Encrypted $file"
    else
        echo "⚠ $file already encrypted"
    fi
done
```

## Notes quan trọng

- **Security**: Chỉ commit file `.sops.yaml` đã mã hóa lên GitHub
- **GPG Key**: Backup GPG private key an toàn, mất key = mất toàn bộ secrets  
- **SOPS Secret Generator**: Tự động generate K8s secrets từ encrypted SOPS files
- **Kustomize Integration**: Plugin hoạt động native với Kustomize build process
- **Timeout**: Pod ArgoCD có thể start chậm do phải load GPG keys và SOPS tools
- **Permissions**: Đảm bảo ArgoCD service account có quyền tạo secrets trong target namespace

## File sau khi mã hóa sẽ trông như thế này:
```yaml
# secrets.sops.yaml (AFTER ENCRYPTION)
apiVersion: ENC[AES256_GCM,data:Rm5h...]
kind: ENC[AES256_GCM,data:U29w...]
metadata:
    name: ENC[AES256_GCM,data:YXBw...]
spec:
    secretName: ENC[AES256_GCM,data:YXBw...]
    namespace: ENC[AES256_GCM,data:ZGVm...]
    data:
        database-url: ENC[AES256_GCM,data:cG9z...]
        api-key: ENC[AES256_GCM,data:ZGV2...]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2024-01-15T10:30:00Z"
    pgp:
        - created_at: "2024-01-15T10:30:00Z"
          fp: YOUR_GPG_KEY_FINGERPRINT
    version: 3.8.1
```

## Timeout Configuration cho slow startup

```yaml
# Trong argocd-repo-server.yaml, tăng các timeout:
livenessProbe:
  initialDelaySeconds: 60    # Tăng từ 30s
  timeoutSeconds: 10         # Tăng từ 5s
readinessProbe:
  initialDelaySeconds: 30    # Tăng từ 15s
  timeoutSeconds: 5
```