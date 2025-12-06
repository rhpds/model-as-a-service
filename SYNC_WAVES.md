# ArgoCD Sync Wave Configuration

This document describes the sync wave order for proper resource deployment.

## Sync Wave Order

ArgoCD applies resources in order based on their sync-wave annotation. Lower numbers are applied first.

### Wave -10: Namespace
**Purpose**: Create the namespace before any resources

- `templates/namespace.yaml` - Creates the maas-api namespace

### Wave -5: Database Secrets & Service Accounts
**Purpose**: Create credentials and permissions before they're needed

- `templates/database/postgres-secret.yaml` - PostgreSQL credentials
- `templates/database/postgres-serviceaccount.yaml` - Service account for postgres pod

### Wave -4: Database RBAC
**Purpose**: Grant permissions to service account

- `templates/database/postgres-rolebinding.yaml` - Binds service account to anyuid SCC

### Wave -3: Database PVC and Connection Secrets
**Purpose**: Create storage and connection secrets (PVC won't bind until pod is scheduled)

- `templates/database/postgres-pvc.yaml` - 1Gi PVC with gp3-csi storage class
- `templates/database/database-url-secret.yaml` - Database connection URL for maas-api

### Wave -2: Database Deployment & Service
**Purpose**: Deploy PostgreSQL and its service together

- `templates/database/postgres-deployment.yaml` - PostgreSQL deployment
- `templates/database/postgres-service.yaml` - PostgreSQL service (postgres-service:5432)

**Note**: Deployment and Service are in the same wave. The PVC will bind when the pod is scheduled (WaitForFirstConsumer mode).

### Wave -1: MaaS API RBAC Prerequisites
**Purpose**: Create service account and cluster role before binding

- `templates/maas-api/rbac/serviceaccount.yaml` - Service account for maas-api
- `templates/maas-api/rbac/clusterrole.yaml` - Cluster-level permissions

### Wave 0: MaaS API RBAC Binding
**Purpose**: Bind service account to cluster role

- `templates/maas-api/rbac/clusterrolebinding.yaml` - Binds maas-api SA to cluster role

### Wave 1: MaaS API Application
**Purpose**: Deploy the main application after all dependencies are ready

- `templates/maas-api/deployment/deployment.yaml` - MaaS API deployment
- `templates/maas-api/deployment/service.yaml` - MaaS API service

## Why This Order Matters

### 1. Namespace First (Wave -10)
ArgoCD needs the namespace to exist before creating any namespaced resources. Without this, all other resources will fail to create.

### 2. Secrets & Service Accounts Before Consumers (Waves -5 to -4)
- PostgreSQL deployment needs the `postgres-credentials` secret
- PostgreSQL pod needs the `postgres` service account
- Service account needs to exist before the role binding

### 3. PVC Before Deployment (Wave -3)
- The PVC is created before the deployment
- With `volumeBindingMode: WaitForFirstConsumer`, the PVC stays in Pending state
- When the postgres deployment is created (Wave -2), the pod is scheduled
- The PVC then binds to a volume on the node where the pod is scheduled

### 4. Database Before MaaS API (Waves -2 to 1)
- PostgreSQL must be running before maas-api starts
- The wave gap ensures postgres is healthy before maas-api attempts to connect
- MaaS API gets the DATABASE_URL from the secret created in Wave -3

### 5. RBAC in Order (Waves -1, 0)
- ServiceAccount and ClusterRole must exist before ClusterRoleBinding
- All RBAC must be ready before the Deployment that uses it

## Key Points About PVC with WaitForFirstConsumer

The `gp3-csi` storage class uses `volumeBindingMode: WaitForFirstConsumer`. This means:

1. **PVC created (Wave -3)**: PVC is created and stays in Pending state
2. **Deployment created (Wave -2)**: Deployment tries to create a pod
3. **Pod scheduled**: Kubernetes schedules the pod to a node
4. **Volume provisioned**: AWS EBS volume is created in the same AZ as the node
5. **PVC bound**: PVC transitions from Pending to Bound
6. **Pod starts**: Pod can now mount the volume and start

**Important**: ArgoCD should NOT wait for the PVC to bind. The PVC and Deployment should be in close waves (PVC at -3, Deployment at -2) so ArgoCD moves forward with the deployment, allowing Kubernetes to bind the PVC when the pod is scheduled.

## Troubleshooting

### ArgoCD Stuck on PVC
**Symptom**: ArgoCD shows PVC in Pending state and won't progress

**Cause**: ArgoCD is waiting for PVC to bind, but it can't bind without a pod

**Solution**:
- Ensure PVC sync-wave (-3) is before Deployment sync-wave (-2)
- Ensure namespace exists (Wave -10)
- Check that the deployment is actually created (look for postgres deployment)

### Pod CrashLoopBackOff
**Symptom**: PostgreSQL pod keeps restarting

**Possible causes**:
1. Missing secrets (check Wave -5)
2. Missing service account (check Wave -5)
3. SCC permissions denied (check Wave -4)
4. Environment variables wrong (check postgres-deployment.yaml)

### MaaS API Can't Connect to Database
**Symptom**: maas-api pod logs show database connection errors

**Possible causes**:
1. PostgreSQL not ready (check postgres pod status)
2. DATABASE_URL secret missing (check Wave -3)
3. DATABASE_URL not injected (check maas-api deployment env vars)
4. Service not created (check Wave -2)

## Testing Sync Waves

To verify sync wave order:

```bash
# Render templates and check sync waves
helm template . --namespace maas-api | grep -B 5 "sync-wave"

# Apply with ArgoCD and watch order
kubectl get events -n maas-api --sort-by='.lastTimestamp' -w

# Check resource creation order
kubectl get all,pvc,secrets,sa,clusterrole,clusterrolebinding -n maas-api -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp
```

## Summary

The complete deployment order:
1. Namespace (-10)
2. PostgreSQL secrets & SA (-5)
3. PostgreSQL RBAC (-4)
4. PostgreSQL PVC & DB URL secret (-3)
5. PostgreSQL deployment & service (-2)
6. MaaS API RBAC (-1, 0)
7. MaaS API application (1)

This ensures all dependencies are satisfied before each resource is created, and ArgoCD won't get stuck waiting for the PVC to bind.
