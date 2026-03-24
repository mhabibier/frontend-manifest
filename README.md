## ☸️ Kubernetes Deployment for Auth Service
 
### Struktur Manifest
 
```
k8s/
├── base/
│   ├── namespace.yaml        # Namespace: auth-service
│   ├── serviceaccount.yaml   # ServiceAccount (no token automount)
│   ├── registry-secret.yaml  # Docker credentials untuk private registry
│   ├── configmap.yaml        # Non-sensitive config (PORT, NODE_ENV, OTLP, dll)
│   ├── secret.yaml           # Sensitive credentials (DATABASE_URL, JWT_SECRET, dll)
│   ├── deployment.yaml       # Deployment: imagePullSecrets, probes, security ctx
│   ├── service.yaml          # ClusterIP Service (port 80 → pod 3000)
│   ├── hpa.yaml              # HPA: scale 2–10 pods (CPU 70% / Memory 80%)
│   ├── pdb.yaml              # PodDisruptionBudget (minAvailable: 1)
│   └── kustomization.yaml    # Kustomize base
└── overlays/
    ├── dev/
    │   └── kustomization.yaml  # 1 replica, lower resources, tag: dev
    └── prod/
        └── kustomization.yaml  # 3 replicas, HPA max 20, semver tag
```
 
---
 
### Prerequisites di Kubeadm Cluster
 
```bash
# 1. metrics-server (dibutuhkan HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
 
# Jika cluster menggunakan self-signed TLS, tambahkan flag ini ke metrics-server:
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
 
# 2. Verifikasi metrics-server berjalan
kubectl top nodes
```
 
---
 
### 1. Build & Push Image ke Private Registry
 
```bash
# Login ke private registry
docker login registry.yourdomain.com
 
# Build image
docker build -t registry.yourdomain.com/auth-microservice:1.0.0 .
 
# Push
docker push registry.yourdomain.com/auth-microservice:1.0.0
```
 
Ganti `registry.yourdomain.com` di file berikut dengan hostname registry Anda:
- `k8s/base/deployment.yaml` — field `image:`
- `k8s/overlays/dev/kustomization.yaml` — field `images.name:`
- `k8s/overlays/prod/kustomization.yaml` — field `images.name:`
 
---
 
### 2. Buat Registry Secret
 
Secret ini dibutuhkan agar node Kubernetes bisa pull image dari private registry.
 
**Cara paling mudah (kubectl imperative):**
 
```bash
kubectl create namespace auth-service
 
kubectl create secret docker-registry registry-credentials \
  --namespace auth-service \
  --docker-server=registry.yourdomain.com \
  --docker-username=your-username \
  --docker-password=your-password \
  --docker-email=your-email@example.com
```
 
**Atau dari sesi docker login yang sudah ada:**
 
```bash
docker login registry.yourdomain.com
 
kubectl create secret generic registry-credentials \
  --namespace auth-service \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```
 
> Jika membuat Secret via kubectl, hapus atau kosongkan `registry-secret.yaml` agar tidak di-apply ulang oleh kustomize dan menimpa Secret yang sudah ada.
 
---
 
### 3. Buat App Secret
 
Jangan pernah commit nilai asli ke git. Gunakan kubectl imperative:
 
```bash
kubectl create secret generic auth-service-secret \
  --namespace auth-service \
  --from-literal=DATABASE_URL="postgresql://postgres:yourpassword@postgres-svc.postgres.svc.cluster.local:5432/auth_db" \
  --from-literal=JWT_SECRET="$(openssl rand -hex 64)" \
  --from-literal=ADMIN_USERNAME="admin" \
  --from-literal=ADMIN_PASSWORD="YourStrongAdminPassword!"
```
 
Format `DATABASE_URL` untuk self-hosted cluster:
```
postgresql://<user>:<password>@<postgres-service>.<namespace>.svc.cluster.local:5432/<dbname>
```
 
---
 
### 4. Update ConfigMap
 
Edit `k8s/base/configmap.yaml` — sesuaikan endpoint observability ke namespace yang Anda gunakan di cluster:
 
```yaml
# Jika Alloy dan Loki ada di namespace "monitoring":
OTEL_EXPORTER_OTLP_ENDPOINT: "http://alloy.monitoring.svc.cluster.local:4317"
LOKI_HOST:                   "http://loki.monitoring.svc.cluster.local:3100"
 
# Jika ada di namespace "observability":
OTEL_EXPORTER_OTLP_ENDPOINT: "http://alloy.observability.svc.cluster.local:4317"
LOKI_HOST:                   "http://loki.observability.svc.cluster.local:3100"
```
 
---
 
### 5. Apply ke Cluster
 
```bash
# Development
kubectl apply -k k8s/overlays/dev
 
# Production
kubectl apply -k k8s/overlays/prod
 
# Preview dulu tanpa apply (dry-run)
kubectl apply -k k8s/overlays/prod --dry-run=client
 
# Lihat manifest hasil kustomize sebelum apply
kubectl kustomize k8s/overlays/prod
```
 
---
 
### 6. Verifikasi
 
```bash
# Semua resource di namespace
kubectl get all -n auth-service
 
# Status pod secara detail
kubectl get pods -n auth-service -o wide
 
# Log pod (follow)
kubectl logs -n auth-service -l app=auth-service --tail=100 -f
 
# Describe pod jika ada masalah (ImagePullBackOff, CrashLoopBackOff, dll)
kubectl describe pod -n auth-service -l app=auth-service
 
# Cek apakah image berhasil di-pull dari private registry
kubectl get events -n auth-service --sort-by='.lastTimestamp'
 
# Status HPA
kubectl get hpa -n auth-service
kubectl describe hpa auth-service-hpa -n auth-service
 
# Test health endpoint dari dalam cluster
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never \
  -n auth-service \
  -- curl http://auth-service.auth-service.svc.cluster.local/health
 
# Test metrics endpoint
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never \
  -n auth-service \
  -- curl http://auth-service.auth-service.svc.cluster.local/metrics
```
 
---
 
### Troubleshooting Private Registry
 
| Error | Penyebab | Solusi |
|---|---|---|
| `ImagePullBackOff` | Node tidak bisa pull image | Cek `registry-credentials` Secret sudah ada dan benar |
| `ErrImagePull` | Registry tidak bisa diakses | Pastikan registry hostname resolve dari dalam cluster |
| `unauthorized` di events | Credentials salah | Re-create `registry-credentials` Secret dengan credentials yang benar |
| `x509: certificate signed by unknown authority` | Self-signed cert registry | Tambahkan CA cert ke containerd config di tiap node, atau gunakan `--insecure-registry` |
 
**Tambahkan insecure registry di kubeadm node** (jika registry pakai HTTP atau self-signed HTTPS):
 
```bash
# Di setiap worker node dan control plane
sudo mkdir -p /etc/containerd
sudo tee /etc/containerd/config.toml > /dev/null << 'TOML'
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.yourdomain.com"]
  endpoint = ["http://registry.yourdomain.com"]
TOML
sudo systemctl restart containerd
```
 
---
 
### Rolling Update (Deploy Versi Baru)
 
```bash
# Build + push image baru
docker build -t registry.yourdomain.com/auth-microservice:1.1.0 .
docker push registry.yourdomain.com/auth-microservice:1.1.0
 
# Update tag di overlays/production/kustomization.yaml:
#   newTag: "1.1.0"
 
# Apply — Kubernetes akan rolling update secara otomatis
kubectl apply -k k8s/overlays/production
 
# Monitor rollout
kubectl rollout status deployment/auth-service -n auth-service
 
# Rollback jika ada masalah
kubectl rollout undo deployment/auth-service -n auth-service
```
 
---
 
### Manifest Summary
 
| Manifest | Keterangan |
|---|---|
| `namespace.yaml` | Isolasi ke namespace `auth-service` |
| `sa.yaml` | SA dedicated, automount token dinonaktifkan |
| `registry-secret.yaml` | Credentials untuk pull dari private registry |
| `configmap.yaml` | Config non-sensitif: port, env, OTLP/Loki endpoint |
| `secret.yaml` | DB URL, JWT secret, admin credentials |
| `deployment.yaml` | Pod spec: `imagePullSecrets`, security ctx, probes, anti-affinity |
| `service.yaml` | ClusterIP — akses internal cluster via port 80 |
| `hpa.yaml` | Auto-scale 2–10 pods berdasarkan CPU/memory |
| `pdb.yaml` | Minimal 1 pod tetap hidup saat node drain/upgrade |