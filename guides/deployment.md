# VOLUND Production Deployment Guide

This guide covers deploying the VOLUND AI agent platform to a production Kubernetes cluster.

## 1. Prerequisites

| Requirement | Minimum Version | Notes |
|---|---|---|
| Kubernetes | 1.28+ | Tested on EKS, GKE, AKS |
| Helm | 3.14+ | |
| kubectl | 1.28+ | Matching cluster version |
| Container Runtime | containerd 1.7+ | CRI-O 1.28+ also supported |
| Storage Class | `gp3` / `pd-ssd` / equivalent | Must support `ReadWriteOnce`; block storage for PostgreSQL |
| Ingress Controller | nginx-ingress 1.9+ | Or any networking.k8s.io/v1 compatible controller |
| cert-manager | 1.14+ | For automated TLS |

Cluster sizing (minimum for production):

- 3 nodes, 4 vCPU / 16 GiB each (dedicated node group for warm pools recommended)
- Storage class with `allowVolumeExpansion: true`

```bash
# Verify cluster
kubectl version --short
kubectl get storageclasses
kubectl get nodes -o wide
```

## 2. Namespace Setup

```bash
kubectl create namespace volund-system
kubectl create namespace volund-infra
```

- `volund-system` -- control plane, operator, and shared services
- `volund-infra` -- PostgreSQL, NATS, Redis, MinIO
- Tenant agent namespaces are provisioned dynamically by the operator (see section 8)

## 3. Infrastructure

### 3.1 PostgreSQL (CloudNativePG)

Install the CloudNativePG operator, then create a cluster with pgvector:

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm install cnpg-operator cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace \
  --version 0.22.0
```

```yaml
# postgres-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: volund-db
  namespace: volund-infra
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16.4
  postgresql:
    shared_preload_libraries:
      - "vectors.so"
    parameters:
      max_connections: "200"
      shared_buffers: "1GB"
      effective_cache_size: "3GB"
      work_mem: "16MB"
  storage:
    size: 50Gi
    storageClass: gp3
  bootstrap:
    initdb:
      database: volund
      owner: volund
      postInitSQL:
        - CREATE EXTENSION IF NOT EXISTS vector;
        - CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
  backup:
    barmanObjectStore:
      destinationPath: "s3://volund-backups/pg/"
      s3Credentials:
        accessKeyId:
          name: backup-s3-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-s3-creds
          key: SECRET_ACCESS_KEY
    retentionPolicy: "30d"
```

```bash
kubectl apply -f postgres-cluster.yaml
# Wait for cluster ready
kubectl -n volund-infra wait --for=condition=Ready cluster/volund-db --timeout=300s
```

Retrieve the connection string:

```bash
# The operator creates a secret named <cluster>-app
kubectl -n volund-infra get secret volund-db-app -o jsonpath='{.data.uri}' | base64 -d
# Example: postgres://volund:PASSWORD@volund-db-rw.volund-infra:5432/volund?sslmode=verify-full
```

### 3.2 NATS

```bash
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm install nats nats/nats \
  --namespace volund-infra \
  --set config.jetstream.enabled=true \
  --set config.jetstream.fileStore.pvc.size=10Gi \
  --set config.jetstream.fileStore.pvc.storageClassName=gp3 \
  --set config.cluster.enabled=true \
  --set config.cluster.replicas=3
```

Connection URL: `nats://nats.volund-infra:4222`

### 3.3 Redis

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis \
  --namespace volund-infra \
  --set architecture=replication \
  --set auth.enabled=true \
  --set auth.existingSecret=redis-auth \
  --set auth.existingSecretPasswordKey=redis-password \
  --set replica.replicaCount=2 \
  --set master.persistence.size=8Gi
```

Create the auth secret first:

```bash
kubectl -n volund-infra create secret generic redis-auth \
  --from-literal=redis-password="$(openssl rand -base64 32)"
```

Connection address: `redis-master.volund-infra:6379`

### 3.4 MinIO (or S3-compatible storage)

Skip this if using AWS S3 or GCS directly.

```bash
helm repo add minio https://charts.min.io/
helm install minio minio/minio \
  --namespace volund-infra \
  --set replicas=4 \
  --set persistence.size=100Gi \
  --set resources.requests.memory=1Gi \
  --set rootUser=volund-admin \
  --set rootPassword="$(openssl rand -base64 32)" \
  --set buckets[0].name=volund-attachments \
  --set buckets[0].policy=none
```

## 4. Secrets Management

Generate secrets before deploying Helm charts. Store these in a secrets manager (Vault, AWS Secrets Manager, SOPS) and reference them via External Secrets Operator or sealed-secrets in production.

```bash
# Generate required secrets
JWT_SECRET=$(openssl rand -base64 32)
CREDENTIAL_KEY=$(openssl rand -hex 32)  # 32 bytes = 64 hex chars, for AES-256-GCM

# Retrieve the database URL from CloudNativePG
DB_URL=$(kubectl -n volund-infra get secret volund-db-app -o jsonpath='{.data.uri}' | base64 -d)

# Retrieve the Redis password
REDIS_PASS=$(kubectl -n volund-infra get secret redis-auth -o jsonpath='{.data.redis-password}' | base64 -d)
```

### Using External Secrets Operator (recommended)

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: volund-secrets
  namespace: volund-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager  # or vault, gcp-sm, etc.
    kind: ClusterSecretStore
  target:
    name: volund-secrets
  data:
    - secretKey: jwt-secret
      remoteRef:
        key: volund/production
        property: jwt-secret
    - secretKey: credential-encryption-key
      remoteRef:
        key: volund/production
        property: credential-encryption-key
    - secretKey: database-url
      remoteRef:
        key: volund/production
        property: database-url
```

### Direct secret creation (non-production or bootstrapping)

```bash
kubectl -n volund-system create secret generic volund-secrets \
  --from-literal=jwt-secret="${JWT_SECRET}" \
  --from-literal=credential-encryption-key="${CREDENTIAL_KEY}" \
  --from-literal=database-url="${DB_URL}"
```

## 5. TLS Setup

### 5.1 Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

### 5.2 Create ClusterIssuer

```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@yourdomain.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

```bash
kubectl apply -f cluster-issuer.yaml
```

## 6. Helm Chart Deployment

### 6.1 VOLUND Gateway (Control Plane)

Create a values file:

```yaml
# values-prod.yaml
replicaCount: 3

image:
  repository: ghcr.io/ai-volund/volund
  tag: "0.1.0"

gateway:
  jwtSecret: ""        # Overridden by existing secret
  credentialEncryptionKey: ""  # Overridden by existing secret

database:
  url: ""  # Overridden by existing secret

nats:
  url: "nats://nats.volund-infra:4222"

otel:
  enabled: true
  endpoint: "otel-collector.volund-system:4317"

oidc:
  providers: |
    [
      {
        "name": "google",
        "issuer": "https://accounts.google.com",
        "clientId": "YOUR_CLIENT_ID",
        "clientSecret": "YOUR_CLIENT_SECRET",
        "redirectUrl": "https://volund.yourdomain.com/auth/callback/google"
      }
    ]

environment: prod

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
  hosts:
    - host: volund.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: volund-tls
      hosts:
        - volund.yourdomain.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi

networkPolicy:
  enabled: true
  defaultDeny: true
  allowAgentIngress: true

resourceQuota:
  enabled: true
  hard:
    pods: "100"
    requests.cpu: "20"
    requests.memory: "40Gi"

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - volund
          topologyKey: kubernetes.io/hostname
```

Deploy:

```bash
helm install volund ./deploy/helm/volund \
  --namespace volund-system \
  --values values-prod.yaml \
  --set gateway.jwtSecret="PLACEHOLDER" \
  --timeout 5m

# If using pre-created secrets, patch the deployment to reference them
# instead of the chart-generated secret (see section 4).
```

If you pre-created secrets via External Secrets Operator, override the secret name in the deployment or patch envFrom accordingly. The chart-generated secret template (`volund-secrets`) uses `stringData` from values -- for production, point to the externally managed secret.

### 6.2 VOLUND Operator

```bash
# The operator chart lives in the volund-operator repo
helm install volund-operator ./deploy/helm/volund-operator \
  --namespace volund-system \
  --set image.tag="0.1.0" \
  --set warmPool.defaultSize=3 \
  --set warmPool.maxSize=20 \
  --set warmPool.agentImage="ghcr.io/ai-volund/volund-agent:0.1.0" \
  --set controlPlane.gatewayUrl="http://volund.volund-system:8080" \
  --set controlPlane.natsUrl="nats://nats.volund-infra:4222" \
  --timeout 5m
```

Verify CRDs are installed:

```bash
kubectl get crds | grep volund
# Expected: agentinstances.volund.io, warmpools.volund.io, skills.volund.io
```

## 7. Configuration Reference

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `VOLUND_DATABASE_URL` | Yes | PostgreSQL connection string with `sslmode=verify-full` for production |
| `VOLUND_NATS_URL` | Yes | NATS server URL (`nats://host:4222`) |
| `VOLUND_REDIS_ADDR` | No | Redis address (`host:6379`). Used for caching and rate limiting |
| `VOLUND_JWT_SECRET` | Yes | Secret for signing JWTs. Minimum 32 characters |
| `VOLUND_CREDENTIAL_KEY` | Yes | 32-byte hex string for AES-256-GCM encryption of stored credentials |
| `VOLUND_ENV` | No | Environment name: `prod`, `staging`, `dev`. Defaults to `prod` |
| `VOLUND_OTLP_ENDPOINT` | No | OTel Collector gRPC endpoint (`host:4317`) |
| `VOLUND_OPENAI_API_KEY` | No | Platform-level OpenAI API key (tenants can supply their own) |
| `VOLUND_ANTHROPIC_API_KEY` | No | Platform-level Anthropic API key |
| `VOLUND_OLLAMA_URL` | No | Ollama base URL for self-hosted models |
| `VOLUND_OIDC_PROVIDERS` | No | JSON array of OIDC provider configs (see below) |
| `VOLUND_STORAGE_BACKEND` | No | `s3` or `local`. Defaults to `local` |
| `VOLUND_S3_BUCKET` | No | S3 bucket name for file attachments |
| `VOLUND_S3_REGION` | No | AWS region or MinIO region |
| `VOLUND_S3_ENDPOINT` | No | Custom S3 endpoint (for MinIO: `http://minio.volund-infra:9000`) |
| `VOLUND_S3_ACCESS_KEY` | No | S3 access key |
| `VOLUND_S3_SECRET_KEY` | No | S3 secret key |
| `VOLUND_MAX_UPLOAD_SIZE` | No | Max file upload size. Defaults to `10MB` |
| `VOLUND_GATEWAY_HTTP_ADDR` | No | HTTP listen address. Defaults to `:8080` |
| `VOLUND_GATEWAY_GRPC_ADDR` | No | gRPC listen address. Defaults to `:9090` |
| `VOLUND_SKILL_CONFIG_PATH` | No | Path to skills configuration YAML |

### OIDC Provider Config Format

```json
[
  {
    "name": "google",
    "issuer": "https://accounts.google.com",
    "clientId": "...",
    "clientSecret": "...",
    "redirectUrl": "https://volund.yourdomain.com/auth/callback/google",
    "scopes": ["openid", "profile", "email"]
  }
]
```

## 8. Observability Stack

Deploy the Grafana LGTM stack (Loki, Grafana, Tempo, Mimir/Prometheus) with an OTel Collector.

### 8.1 OpenTelemetry Collector

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace volund-system \
  --set mode=deployment \
  --set config.exporters.otlp/tempo.endpoint="tempo.volund-system:4317" \
  --set config.exporters.prometheusremotewrite.endpoint="http://prometheus.volund-system:9090/api/v1/write" \
  --set config.exporters.loki.endpoint="http://loki.volund-system:3100/loki/api/v1/push"
```

A full OTel Collector config for VOLUND:

```yaml
# otel-collector-values.yaml
config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
  processors:
    batch:
      timeout: 5s
      send_batch_size: 1024
    memory_limiter:
      check_interval: 1s
      limit_mib: 512
  exporters:
    otlp/tempo:
      endpoint: "tempo.volund-system:4317"
      tls:
        insecure: true
    prometheusremotewrite:
      endpoint: "http://prometheus.volund-system:9090/api/v1/write"
    loki:
      endpoint: "http://loki.volund-system:3100/loki/api/v1/push"
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp/tempo]
      metrics:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [prometheusremotewrite]
      logs:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [loki]
```

### 8.2 Grafana + Tempo + Loki + Prometheus

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace volund-system \
  --set server.persistentVolume.size=50Gi \
  --set server.retention=30d

# Tempo (traces)
helm install tempo grafana/tempo \
  --namespace volund-system \
  --set tempo.storage.trace.backend=s3 \
  --set persistence.enabled=true \
  --set persistence.size=20Gi

# Loki (logs)
helm install loki grafana/loki \
  --namespace volund-system \
  --set loki.auth_enabled=false \
  --set singleBinary.replicas=1 \
  --set singleBinary.persistence.size=20Gi

# Grafana
helm install grafana grafana/grafana \
  --namespace volund-system \
  --set persistence.enabled=true \
  --set adminPassword="CHANGE_ME" \
  --set "datasources.datasources\\.yaml.apiVersion=1" \
  --set "datasources.datasources\\.yaml.datasources[0].name=Prometheus" \
  --set "datasources.datasources\\.yaml.datasources[0].type=prometheus" \
  --set "datasources.datasources\\.yaml.datasources[0].url=http://prometheus-server.volund-system" \
  --set "datasources.datasources\\.yaml.datasources[1].name=Tempo" \
  --set "datasources.datasources\\.yaml.datasources[1].type=tempo" \
  --set "datasources.datasources\\.yaml.datasources[1].url=http://tempo.volund-system:3100" \
  --set "datasources.datasources\\.yaml.datasources[2].name=Loki" \
  --set "datasources.datasources\\.yaml.datasources[2].type=loki" \
  --set "datasources.datasources\\.yaml.datasources[2].url=http://loki.volund-system:3100"
```

### Key Metrics to Monitor

- `volund_gateway_request_duration_seconds` -- API latency by endpoint
- `volund_agent_instances_active` -- active agent count per tenant
- `volund_warm_pool_available` -- available warm pods
- `volund_warm_pool_pending` -- pods waiting to be ready
- `volund_credential_decrypt_errors_total` -- credential decryption failures (alert on any non-zero)

## 9. Multi-Tenant Namespace Provisioning

The operator manages tenant namespaces automatically. Each tenant gets an isolated namespace with:

- NetworkPolicy restricting cross-tenant traffic
- ResourceQuota limiting compute and storage
- RBAC roles scoped to the tenant

Create a tenant via the API:

```bash
curl -X POST https://volund.yourdomain.com/api/v1/tenants \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "acme-corp",
    "plan": "standard",
    "quotas": {
      "maxAgents": 10,
      "maxConcurrentSessions": 50,
      "cpuLimit": "8",
      "memoryLimit": "16Gi"
    }
  }'
```

The operator will create namespace `volund-tenant-acme-corp` with the following resources:

```yaml
# Auto-generated by the operator
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: volund-tenant-acme-corp
spec:
  hard:
    pods: "20"
    requests.cpu: "8"
    requests.memory: "16Gi"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: volund-tenant-acme-corp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              volund.io/system: "true"
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              volund.io/system: "true"
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - port: 443
          protocol: TCP
```

## 10. Verification

Run these checks after deployment to confirm everything is healthy.

### 10.1 Control Plane Health

```bash
# Gateway pods running
kubectl -n volund-system get pods -l app.kubernetes.io/name=volund
# All replicas should be Running and Ready

# Health endpoint
kubectl -n volund-system port-forward svc/volund 8080:8080 &
curl -s http://localhost:8080/healthz
# Expected: {"status":"ok","database":"connected","nats":"connected"}
```

### 10.2 Operator

```bash
# Operator running
kubectl -n volund-system get pods -l app.kubernetes.io/name=volund-operator

# CRDs registered
kubectl get crds | grep volund.io

# Warm pool status
kubectl get warmpools -A
# Should show desired/ready counts
```

### 10.3 Infrastructure

```bash
# PostgreSQL
kubectl -n volund-infra get cluster volund-db
# STATUS should be "Cluster in healthy state"

# NATS
kubectl -n volund-infra get pods -l app.kubernetes.io/name=nats
# All pods Running

# Redis
kubectl -n volund-infra get pods -l app.kubernetes.io/name=redis
```

### 10.4 End-to-End Test

```bash
# Create a test tenant and agent session
TOKEN=$(curl -s -X POST https://volund.yourdomain.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@yourdomain.com","password":"..."}' | jq -r '.token')

# Create an agent session (should allocate from warm pool)
curl -s -X POST https://volund.yourdomain.com/api/v1/sessions \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"agentId":"test-agent"}' | jq .

# Check warm pool replenished
kubectl get warmpools -A
```

### 10.5 TLS

```bash
# Verify certificate issued
kubectl -n volund-system get certificate
# READY should be True

# Test TLS externally
curl -v https://volund.yourdomain.com/healthz 2>&1 | grep "SSL certificate verify ok"
```

## Appendix: Upgrade Procedure

```bash
# 1. Update image tags in values
# 2. Run database migrations (built into the binary)
kubectl -n volund-system exec deploy/volund -- /volund migrate up

# 3. Helm upgrade with --atomic for automatic rollback on failure
helm upgrade volund ./deploy/helm/volund \
  --namespace volund-system \
  --values values-prod.yaml \
  --set image.tag="0.2.0" \
  --atomic \
  --timeout 5m

# 4. Upgrade the operator
helm upgrade volund-operator ./deploy/helm/volund-operator \
  --namespace volund-system \
  --set image.tag="0.2.0" \
  --atomic \
  --timeout 5m

# 5. Verify
kubectl -n volund-system rollout status deploy/volund
kubectl -n volund-system rollout status deploy/volund-operator
```

## Appendix: Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Gateway CrashLoopBackOff | Missing `VOLUND_JWT_SECRET` or bad DB URL | Check `kubectl logs` and verify secrets |
| Warm pool pods stuck Pending | Insufficient cluster resources or missing node selector | Check `kubectl describe pod` for scheduling events |
| 502 from ingress | Gateway not ready or service port mismatch | Verify service ports match (8080 HTTP, 9090 gRPC) |
| Agent sessions time out | NATS not reachable from agent namespace | Check NetworkPolicy egress rules |
| Credential decryption errors | Wrong `VOLUND_CREDENTIAL_KEY` after rotation | Key must match the key used to encrypt stored credentials |
| Certificate not issuing | cert-manager solver failing | Check `kubectl describe challenge` in cert-manager namespace |
