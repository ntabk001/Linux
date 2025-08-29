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

## 8. Workflow Development

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

- **Security**: Chỉ commit file `.enc.yaml` lên GitHub, không bao giờ commit file gốc
- **GPG Key**: Backup GPG private key an toàn, mất key = mất toàn bộ secrets
- **Timeout**: Pod ArgoCD có thể start chậm do phải load GPG keys và SOPS tools
- **Permissions**: Đảm bảo ArgoCD service account có quyền tạo secrets trong target namespace

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