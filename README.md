# Convex Self-Hosted Kubernetes Helm Chart

This Helm chart provides a deployable instance of the Convex self-hosted stack on a Kubernetes cluster. It is designed to orchestrate the `convex-backend` and `convex-dashboard` components, along with the necessary services, configuration, and networking required for a functional deployment.

## 💡 Official Documentation Reference

This chart is based on the official Convex self-hosted documentation. It is highly recommended to review the official source for conceptual understanding and troubleshooting:

**Official Convex Self-Hosted Guide:** [https://github.com/get-convex/convex-backend/tree/main/self-hosted](https://github.com/get-convex/convex-backend/tree/main/self-hosted)

## ⚠️ Executive Summary: A Multi-Step Process

This is **not** a simple "one-click" `helm install` chart. A successful deployment requires you to follow a precise, three-stage process to provision external dependencies and manage a "chicken-and-egg" secret-generation problem.

1. **Step 0: Provision Prerequisites** - Create your external S3-compatible object storage and PostgreSQL database
2. **Step 1: Create Prerequisite Secrets** - Manually create the Kubernetes secrets that point to your external dependencies
3. **Step 2: Deploy the Chart** - Run `helm install` using a custom `values.yaml` file
4. **Step 3: Post-Install Configuration** - Generate an admin key, create the final secret, and restart the backend pod

**Failure to follow these steps in this exact order will result in a non-functional deployment.**

---

## 1. Architectural Overview

This chart deploys the following resources into your Kubernetes cluster:

- **StatefulSet (Backend)** - The core `convex-backend` application with stable network identity
- **Deployment (Dashboard)** - The `convex-dashboard` web interface
- **ConfigMap** - Central configuration that dynamically populates environment variables
- **Services:**
  - `{{.Release.Name}}-backend-svc` - ClusterIP service exposing backend API (port 3210) and HTTP actions (port 3211)
  - `{{.Release.Name}}-dashboard-svc` - ClusterIP service for the dashboard
  - `{{.Release.Name}}-backend-headless` - Headless service for StatefulSet identity
- **Ingress (Optional)** - Ingress resource to expose services publicly (recommended)
- **PersistentVolumeClaim (Optional)** - PVC for backend storage (strongly discouraged)

---

## 2. ⚠️ Step 0: Critical Prerequisites

Before you can deploy the chart, you must provision the external stateful dependencies.

### 2A: S3-Compatible Object Storage

The Convex backend requires S3-compatible object storage for all persistent data (modules, files, snapshots, etc.).

**Supported providers:** AWS S3, MinIO, Hetzner Object Storage, or any S3-compatible service.

#### 🚨 CRITICAL WARNING: Hardcoded Bucket Name

This version of the Helm chart has the S3 bucket name hardcoded to **`convexpro`**.

- You **must** create a bucket named exactly `convexpro` in your object storage provider
- The S3 credentials must have full read/write access to this specific bucket
- Attempting to use a different bucket name will fail
- This will be parameterized in a future release

### 2B: External PostgreSQL Database

The backend requires a PostgreSQL database for its core metadata.

- Provision a PostgreSQL database (e.g., AWS RDS, Google Cloud SQL, or self-managed)
- The chart does **not** deploy a database for you
- You will need: database connection URL, instance name, and a generated secret

### 2C: (Discouraged) PVC-Based Storage

The `values.yaml` contains an option for `backend.persistence.enabled: true`.

**⚠️ We strongly discourage using this option** for anything other than basic testing.

- The application is architected for the resilience and scalability of object storage
- The PVC option is less-tested and less-resilient
- For production deployments, use S3-compatible object storage (Step 2A)

---

## 3. Step 1: Create Prerequisite Secrets

The backend pod will fail to start if these secrets don't exist before `helm install`.

### 3A: S3 Credentials Secret (Required)

Create a secret named `convex-s3-credentials`:

```bash
kubectl create secret generic convex-s3-credentials \
  --from-literal=AWS_ACCESS_KEY_ID='YOUR_AWS_ACCESS_KEY_ID_HERE' \
  --from-literal=AWS_SECRET_ACCESS_KEY='YOUR_AWS_SECRET_ACCESS_KEY_HERE' \
  --from-literal=AWS_ENDPOINT='YOUR_S3_ENDPOINT_URL_HERE' \
  --from-literal=AWS_REGION='YOUR_S3_REGION_HERE' \
  --namespace YOUR_NAMESPACE
```

**Required keys:**
- `AWS_ACCESS_KEY_ID` - Your S3 access key
- `AWS_SECRET_ACCESS_KEY` - Your S3 secret key
- `AWS_ENDPOINT` - The S3 endpoint URL (e.g., `s3.amazonaws.com` for AWS)
- `AWS_REGION` - The S3 region (e.g., `us-east-1`)

### 3B: PostgreSQL Secret (Recommended)

Create a secret named `convex-postgres-secret`:

```bash
kubectl create secret generic convex-postgres-secret \
  --from-literal=DATABASE_URL='postgresql://USER:PASSWORD@HOST:PORT/' \
  --from-literal=INSTANCE_NAME='YOUR_DATABASE_NAME_HERE' \
  --from-literal=INSTANCE_SECRET='YOUR_GENERATED_SECRET_HERE' \
  --namespace YOUR_NAMESPACE
```

**Required keys:**
- `DATABASE_URL` - Full PostgreSQL connection string
- `INSTANCE_NAME` - Database name (e.g., `convex`)
- `INSTANCE_SECRET` - Arbitrary secret string (generate with `openssl rand -hex 32`)

---

## 4. Step 2: Deploy the Helm Chart

### 4A: Create a Custom values.yaml

**Do not deploy using default values.** Create a file named `my-values.yaml`:

```yaml
# -----------------------------------------------------------------
# Example Production 'my-values.yaml'
# -----------------------------------------------------------------

# Use external PostgreSQL (requires 'convex-postgres-secret' from Step 1B)
postgres:
  enabled: true
  existingSecret: "convex-postgres-secret"

# Backend configuration
backend:
  # This 'secretsName' must match the secret you create in Step 3
  secretsName: convex-secrets
  
  # Disable PVC persistence (using S3 instead)
  persistence:
    enabled: false

# Ingress configuration (RECOMMENDED)
ingress:
  enabled: true
  
  # Set to your Ingress Controller (e.g., "nginx", "traefik")
  className: "nginx"

  # Annotations for NGINX and cert-manager
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

  # Hostname for the Convex Dashboard
  dashboard:
    host: "dashboard.your-domain.com"
    paths:
      - path: /
        pathType: Prefix

  # Hostname for the Convex Backend API
  backend:
    host: "api.your-domain.com"
    paths:
      - path: /
        pathType: Prefix
  
  # (Optional) Hostname for HTTP Actions
  # site:
  #   host: "site.your-domain.com"
  #   paths:
  #     - path: /
  #       pathType: Prefix

  # TLS configuration for cert-manager
  tls:
    - secretName: convex-dashboard-tls
      hosts:
        - "dashboard.your-domain.com"
    - secretName: convex-api-tls
      hosts:
        - "api.your-domain.com"
```

### 4B: Install the Chart

Add the Helm repository:

```bash
helm repo add convex-charts https://YOUR_HELM_REPO_URL
helm repo update
```

Install the chart:

```bash
helm install convex-release \
  convex-charts/convex-self-hosted \
  -f my-values.yaml \
  --namespace YOUR_NAMESPACE
```

At this point, the `convex-dashboard` pod should start correctly. The `convex-backend` pod will be running but logging errors internally (missing admin key). **This is expected.**

---

## 5. Step 3: Post-Installation Configuration (Admin Key)

This final stage resolves the "chicken-and-egg" problem.

### 5A: Find Your Backend Pod Name

```bash
kubectl get pods -n YOUR_NAMESPACE | grep backend
```

Output example:
```
convex-release-backend-0   1/1   Running   0   2m
```

### 5B: Generate the Admin Key

Execute into the pod and run the key generation script:

```bash
kubectl exec -it convex-release-backend-0 -n YOUR_NAMESPACE \
  -- ./generate_admin_key.sh
```

This will output a long alphanumeric string. **Copy this key.**

Output example:
```
cvx_selfhosted_......
```

### 5C: Create the Final Admin Secret

Create the `convex-secrets` secret with the generated key:

```bash
kubectl create secret generic convex-secrets \
  --from-literal=CONVEX_SELF_HOSTED_ADMIN_KEY='cvx_selfhosted_......' \
  --namespace YOUR_NAMESPACE
```

### 5D: 🚨 CRITICAL: Restart the Backend Pod

The backend pod only reads secrets on startup. Restart the StatefulSet:

```bash
kubectl rollout restart statefulset/convex-release-backend -n YOUR_NAMESPACE
```

Once the pod restarts, it will load the admin key and become fully operational.

**✅ Your Convex deployment is now complete!**

---

## 6. Accessing Your Deployment

### Production (Ingress Enabled)

If you deployed with `ingress.enabled: true`:

- **Dashboard:** `https://dashboard.your-domain.com`
- **Backend API:** `https://api.your-domain.com`

### Local Testing (Port-Forward)

If you deployed with `ingress.enabled: false`:

```bash
# Forward the Dashboard
kubectl port-forward svc/convex-release-dashboard-svc 8080:80 -n YOUR_NAMESPACE
# Access at: http://localhost:8080

# Forward the Backend API
kubectl port-forward svc/convex-release-backend-svc 3210:3210 -n YOUR_NAMESPACE
# Access at: http://localhost:3210
```

---

## 7. Configuration Parameters

### Backend

| Parameter | Description | Default |
|-----------|-------------|---------|
| `backend.image.repository` | Image repository for the Convex backend | `ghcr.io/get-convex/convex-backend` |
| `backend.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `backend.image.tag` | Image tag | `latest` |
| `backend.secretsName` | Name of the secret containing `CONVEX_SELF_HOSTED_ADMIN_KEY` | `convex-secrets` |
| `backend.persistence.enabled` | Enable PVC persistence (not recommended) | `false` |
| `backend.persistence.storageClassName` | Storage class for PVC | `longhorn-standard` |
| `backend.persistence.accessMode` | Access mode for PVC | `ReadWriteOnce` |
| `backend.persistence.size` | Size of PVC | `5Gi` |

### PostgreSQL

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgres.enabled` | Use external PostgreSQL database (recommended) | `false` |
| `postgres.existingSecret` | Name of secret containing PostgreSQL credentials | `convex-postgres-secret` |

### Dashboard

| Parameter | Description | Default |
|-----------|-------------|---------|
| `dashboard.image.repository` | Image repository for dashboard | `ghcr.io/get-convex/convex-dashboard` |
| `dashboard.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `dashboard.image.tag` | Image tag | `latest` |

### Service

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.backend.type` | Service type for backend | `ClusterIP` |
| `service.backend.port` | Backend API port | `3210` |
| `service.backend.actionsPort` | Backend HTTP actions port | `3211` |
| `service.dashboard.type` | Service type for dashboard | `ClusterIP` |
| `service.dashboard.port` | Dashboard service port | `80` |
| `service.dashboard.targetPort` | Dashboard container port | `6791` |

### Ingress

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ingress.enabled` | Create Ingress resources (recommended) | `false` |
| `ingress.className` | Ingress class name (e.g., `nginx`, `traefik`) | `""` |
| `ingress.annotations` | Annotations for Ingress | `{}` |
| `ingress.dashboard.host` | Public hostname for dashboard | `""` |
| `ingress.backend.host` | Public hostname for backend API | `""` |
| `ingress.site.host` | (Optional) Hostname for HTTP actions | `""` |
| `ingress.tls` | TLS configuration | `[]` |

### Convex Config (Local Mode)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `convexConfig.cloudOrigin` | `CONVEX_CLOUD_ORIGIN` when Ingress disabled | `http://localhost:3210` |
| `convexConfig.siteOrigin` | `CONVEX_SITE_ORIGIN` when Ingress disabled | `http://localhost:3211` |
| `convexConfig.deploymentUrl` | `NEXT_PUBLIC_DEPLOYMENT_URL` when Ingress disabled | `http://localhost:3210` |
| `convexConfig.requireSsl` | `DO_NOT_REQUIRE_SSL` (inverted) when Ingress disabled | `true` |

---

## 8. Uninstallation

To uninstall the deployment:

```bash
helm uninstall convex-release -n YOUR_NAMESPACE
```

### ⚠️ Data Destruction Warning

Uninstallation will **not** delete:

- The Kubernetes secrets you created manually:
  - `convex-s3-credentials`
  - `convex-postgres-secret`
  - `convex-secrets`
- Your external PostgreSQL database and its data
- Your S3 bucket (`convexpro`) and all data within it
- The PersistentVolumeClaim (if `backend.persistence.enabled: true`)

**You must manually delete these resources if you wish to completely remove all data.**

---

## Support & Contributing

For issues, questions, or contributions, please refer to the [official Convex self-hosted documentation](https://github.com/get-convex/convex-backend/tree/main/self-hosted).

## License

[Add your license information here]
