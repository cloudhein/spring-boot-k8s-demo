# Todo API Helm Chart

This Helm chart deploys the **Todo API** Spring Boot application with **MongoDB** as a dependency using the [cloudpirates/mongodb](https://artifacthub.io/packages/helm/cloudpirates-mongodb/mongodb) chart.

## Features

- ✅ **Least-privilege security**: Application user with `readWrite` only on `todo` database
- ✅ **Auto-generated passwords**: Secure random passwords for MongoDB root and app user
- ✅ **Helm dependency management**: MongoDB chart included as a dependency
- ✅ **Production-ready**: Resource limits, liveness/readiness probes, configurable replicas

## 📋 Prerequisites

- Kubernetes cluster (v1.19+)
- Helm 3.x installed
- kubectl configured to access your cluster
- Container runtime with access to Docker Hub (for MongoDB image)

## 🚀 Quick Start

### 1. Build Helm Dependencies

The chart depends on the `cloudpirates/mongodb` chart from Docker Hub OCI registry. Build dependencies before installation:

```bash
cd helm/todo-api
helm dependency build
# Or: helm dependency update
```

This downloads the MongoDB chart into the `charts/` directory.

### 2. Install the Chart

```bash
# Install with default values
helm install todo-api . --namespace todo-app --create-namespace

# Install with custom release name
helm install my-todo-api . --namespace todo-app --create-namespace

# Wait for deployment to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=my-todo-api -n todo-app --timeout=300s
```

### 3. Verify Installation

```bash
# Check all resources
kubectl get all -n todo-app

# Check pods status
kubectl get pods -n todo-app -w

# Get service details
kubectl get svc -n todo-app

# Access the API (LoadBalancer)
EXTERNAL_IP=$(kubectl get svc todo-api -n todo-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP:8080/api/todos

# Or use port-forward for local access
kubectl port-forward svc/todo-api 8080:8080 -n todo-app
```

## 🔧 Configuration

### Values Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| **Global** |
| `replicaCount` | Number of Todo API replicas | `1` |
| `image.repository` | Docker image repository | `mrnanda/to-do-api` |
| `image.tag` | Docker image tag | `2.0` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| **MongoDB** |
| `mongodb.enabled` | Enable/disable MongoDB dependency | `true` |
| `mongodb.replicaCount` | MongoDB replicas | `1` |
| `mongodb.auth.rootUsername` | MongoDB root username | `admin` |
| `mongodb.auth.rootPassword` | MongoDB root password (empty = auto-generated) | `""` |
| `mongodb.auth.appUsername` | Application user (least privilege) | `todo-user` |
| `mongodb.auth.appDatabase` | Application database | `todo` |
| `mongodb.auth.appAuthDatabase` | Auth database for app user | `todo` |
| `mongodb.customUsers` | Custom users configuration (creates app user) | See below |
| `mongodb.persistence.size` | MongoDB storage size | `2Gi` |
| `mongodb.persistence.storageClass` | Storage class | `standard` |
| **Application** |
| `app.mongodb.username` | Override MongoDB username | `""` (uses `mongodb.auth.appUsername`) |
| `app.mongodb.database` | Override MongoDB database | `""` (uses `mongodb.auth.appDatabase`) |
| `app.mongodb.authDatabase` | Override auth database | `""` (uses `mongodb.auth.appAuthDatabase`) |
| `config.serverPort` | Application server port | `8080` |
| `config.springProfile` | Spring active profile | `default` |
| **Service** |
| `service.type` | Kubernetes service type | `LoadBalancer` |
| `service.port` | Service port | `8080` |
| `service.targetPort` | Container port | `8080` |
| **Probes** |
| `probes.liveness.path` | Liveness probe endpoint | `/api/todos` |
| `probes.liveness.initialDelaySeconds` | Initial delay | `60` |
| `probes.readiness.path` | Readiness probe endpoint | `/api/todos` |
| `probes.readiness.initialDelaySeconds` | Initial delay | `30` |
| **Resources** |
| `resources.requests` | Resource requests | See `values.yaml` |
| `resources.limits` | Resource limits | See `values.yaml` |

### Custom Values Example

Create `custom-values.yaml`:

```yaml
# Increase replicas
replicaCount: 2

# Configure MongoDB persistence
mongodb:
  persistence:
    size: 4Gi
    storageClass: "fast-ssd"

# App configuration
app:
  mongodb:
    username: "app-user"  # Override default app username
    database: "myapp"
    authDatabase: "myapp"

# Resource limits
resources:
  limits:
    memory: "1Gi"
    cpu: "1000m"
  requests:
    memory: "512Mi"
    cpu: "500m"
```

Install with custom values:

```bash
helm install todo-api . -f custom-values.yaml --namespace todo-app --create-namespace
```

### Overriding MongoDB Password

By default, passwords are auto-generated for security. To set a specific root password:

```yaml
mongodb:
  auth:
    rootPassword: "your-secure-password-here"
```

⚠️ **Security Note**: Setting passwords via command line (`--set`) may expose them in shell history. Use `-f` with a YAML file for sensitive values.

## 🔒 Security

### Principle of Least Privilege

The application connects to MongoDB using a dedicated user (`todo-user`) with only `readWrite` permission on the `todo` database. This user is created automatically via the `mongodb.customUsers` configuration.

**User Permissions:**
- Can read/write to the `todo` database only
- Cannot create/drop other databases
- Cannot modify user accounts
- Minimum necessary access for the application

### Password Management

- **Root password**: Auto-generated if left empty (recommended for production)
- **App password**: Auto-generated by MongoDB chart and stored in Kubernetes secret
- Passwords can be retrieved from Kubernetes secrets if needed:

```bash
# Get MongoDB root password
kubectl get secret -n todo-app <release>-mongodb -o jsonpath='{.data.mongodb-root-password}' | base64 -d

# Get app user password
kubectl get secret -n todo-app <release>-mongodb-custom-user-0-secret -o jsonpath='{.data.CUSTOM_PASSWORD}' | base64 -d
```

### Secrets

This chart uses Kubernetes Secrets for sensitive data. For production, consider integrating with external secret management solutions like:
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

## 📦 Chart Structure

```
helm/todo-api/
├── Chart.yaml              # Chart metadata and dependencies
├── values.yaml             # Default configuration values
├── charts/                 # Dependency charts (after helm dependency build)
│   └── mongodb-0.14.2.tgz
├── templates/
│   ├── _helpers.tpl        # Helper templates (fullname, labels, etc.)
│   ├── configmap.yaml      # Application ConfigMap (application.properties)
│   ├── deployment.yaml     # Todo API Deployment
│   ├── ingress.yaml        # Ingress resource (optional)
│   ├── service.yaml        # LoadBalancer Service
└── README.md               # This file
```

**Note**: The `secret.yaml` file was removed because MongoDB credentials are provided by the MongoDB dependency chart's auto-generated secret.

## 🔄 Upgrading

Upgrade an existing release:

```bash
# Upgrade with new values file
helm upgrade todo-api . -f new-values.yaml --namespace todo-app

# Preview changes without applying
helm upgrade todo-api . -f new-values.yaml --namespace todo-app --dry-run --debug

# Upgrade with inline values
helm upgrade todo-api . --set replicaCount=2 --namespace todo-app
```

Rollback if needed:

```bash
# List revision history
helm history todo-api -n todo-app

# Rollback to previous release
helm rollback todo-api 1 -n todo-app
```

## 🗑️ Uninstalling

```bash
# Uninstall the release
helm uninstall todo-api -n todo-app

# Uninstall MongoDB separately (if desired)
helm uninstall my-mongodb -n todo-app  # if deployed with different release name

# Delete all PVCs (data will be lost!)
kubectl delete pvc -n todo-app -l app.kubernetes.io/instance=todo-api
```

## 🐛 Troubleshooting

### Pods Stuck in Pending

**Cause**: Storage class not available or PVC cannot bind.

**Check:**
```bash
kubectl get storageclass
kubectl get pvc -n todo-app
```

**Fix**: Ensure your cluster has a default storage class or specify `mongodb.persistence.storageClass`.

### MongoDB Authentication Failures

**Symptom**: Todo API logs show `AuthenticationFailed`.

**Check:**
```bash
# Verify app user exists in MongoDB
kubectl exec -it mongodb-todo-api-app-0 -n todo-app -- mongosh admin -u admin -p <root-password> --eval "db.getSiblingDB('todo').getUser('todo-user')"

# Check secret values match
kubectl get secret mongodb-todo-api-app-custom-user-0-secret -n todo-app -o yaml
```

**Fix**: The custom user should be created automatically during MongoDB initialization. If missing, the init script may have failed. Check MongoDB logs: `kubectl logs statefulset/mongodb-todo-api-app -n todo-app`.

### Connection Refused from Todo API

**Verify:**
```bash
# Check MongoDB service
kubectl get svc mongodb-todo-api-app -n todo-app

# Test connectivity from Todo API pod
kubectl exec -it deployment/todo-api-todo-api-app -n todo-app -- nc -zv mongodb-todo-api-app 27017
```

### LoadBalancer IP Pending

On kind/minikube, LoadBalancer services won't get an external IP. Use port-forward:

```bash
kubectl port-forward svc/todo-api 8080:8080 -n todo-app
```

For cloud providers, ensure your cluster supports LoadBalancer provisioning (e.g., AWS ELB, GCP LB, Azure LB).

### Pod Restarting / CrashLoopBackOff

Check logs:
```bash
kubectl logs deployment/todo-api -n todo-app
kubectl logs statefulset/mongodb-todo-api-app -n todo-app

# Check events
kubectl describe pod <pod-name> -n todo-app
```

Common issues:
- **Insufficient resources**: Increase `resources.requests`/`limits`
- **Readiness probe failing**: Check app health at `/api/todos` endpoint
- **Database connection timeout**: Verify MongoDB is ready before app starts

## 📊 Monitoring

```bash
# Pod resource usage
kubectl top pods -n todo-app

# Persistent Volume Claims
kubectl get pvc -n todo-app

# Watch all resources
kubectl get all -n todo-app -w
```

## 📝 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/todos` | Retrieve all todos |
| `POST` | `/api/todos` | Create a new todo |
| `PUT` | `/api/todos/{id}` | Update a todo (mark complete/incomplete) |
| `DELETE` | `/api/todos/{id}` | Delete a todo |

### Example Usage

```bash
# Create a todo
curl -X POST http://<service-ip>:8080/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Kubernetes","completed":false}'

# List todos
curl http://<service-ip>:8080/api/todos
```

## 🤝 Contributing

To modify this chart:

1. Update `values.yaml` for configuration changes
2. Modify templates in `templates/` for structural changes
3. Test changes: `helm install --dry-run --debug .`
4. Update `Chart.yaml` version (if needed)
5. Run `helm dependency build` if dependencies change

## 📄 License

[Add your license here]
