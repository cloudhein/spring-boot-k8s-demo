# Kubernetes Manifests for Todo API

This directory contains Kubernetes manifest files to deploy the Todo API application with MongoDB.

## Architecture

- **Namespace**: `todo-app`
- **MongoDB**: StatefulSet with 1 replica, using local-path storage class
- **Todo API**: Deployment with LoadBalancer service
- **Secrets**: Stored separately (update passwords before deploying!)

## Files Structure

```
k8s/
├── 00-namespace.yaml          - Namespace definition
├── 01-mongodb-secret.yaml     - MongoDB root credentials
├── 02-mongodb-configmap.yaml  - MongoDB configuration
├── 03-mongodb-service.yaml    - MongoDB headless service
├── 04-mongodb-statefulset.yaml - MongoDB StatefulSet with PVC
├── 05-todo-api-secret.yaml    - Todo API MongoDB credentials
├── 06-todo-api-configmap.yaml - Todo API application properties
├── 07-todo-api-deployment.yaml - Todo API deployment
├── 08-todo-api-service.yaml   - Todo API LoadBalancer service
├── kustomization.yaml         - Kustomize config for easy deployment
└── README.md                  - This file
```

## Prerequisites

- Kubernetes cluster (kind cluster with `standard` storage class)
- `kubectl` configured
- Docker image `mrnanda/to-do-api:2.0` available (or update image in deployment)

## Deployment

### Option 1: Using Kustomize (Recommended)

```bash
# Apply all manifests
kubectl apply -k k8s/

# Check status
kubectl get all -n todo-app

# View pods
kubectl get pods -n todo-app -o wide

# Get service URL
kubectl get svc todo-api -n todo-app
# External IP will be assigned (may show <pending> on kind without MetalLB)
```

### Option 2: Apply individually in order

```bash
# Create namespace
kubectl apply -f k8s/00-namespace.yaml

# Create MongoDB
kubectl apply -f k8s/01-mongodb-secret.yaml
kubectl apply -f k8s/02-mongodb-configmap.yaml
kubectl apply -f k8s/03-mongodb-service.yaml
kubectl apply -f k8s/04-mongodb-statefulset.yaml

# Wait for MongoDB to be ready
kubectl wait --for=condition=ready pod/mongodb-0 -n todo-app --timeout=120s

# Create Todo API
kubectl apply -f k8s/05-todo-api-secret.yaml
kubectl apply -f k8s/06-todo-api-configmap.yaml
kubectl apply -f k8s/07-todo-api-deployment.yaml
kubectl apply -f k8s/08-todo-api-service.yaml
```

## Accessing the Application

- **Internal** (within cluster): `http://todo-api:8080`
- **External** (LoadBalancer): Get IP with `kubectl get svc todo-api -n todo-app -o wide`
  - On kind without MetalLB: Use port-forward: `kubectl port-forward svc/todo-api 8080:8080 -n todo-app`
  - On cloud/with MetalLB: Access at `http://<EXTERNAL-IP>:8080`
- **MongoDB**: `mongodb:27017` (from within cluster)

## Verification

```bash
# Check all resources
kubectl get all -n todo-app

# Check pods status
kubectl get pods -n todo-app

# Check PVCs
kubectl get pvc -n todo-app

# View logs
kubectl logs -f deployment/todo-api -n todo-app
kubectl logs -f statefulset/mongodb -n todo-app

# Test MongoDB connection from todo-api pod
kubectl exec -it deployment/todo-api -n todo-app -- mongosh \
  --username root \
  --password password \
  --authenticationDatabase admin
```

## Cleanup

```bash
# Delete all resources
kubectl delete -k k8s/

# Or delete namespace (removes all resources)
kubectl delete namespace todo-app
```

## Security Notes

⚠️ **Change default passwords before production use!**

Update the following files with secure credentials:
- `01-mongodb-secret.yaml`
- `05-todo-api-secret.yaml`

Generate secure passwords:
```bash
# Generate random password
openssl rand -base64 32
```

## Resource Limits

- **MongoDB**: 256Mi-512Mi memory, 250m-500m CPU
- **Todo API**: 256Mi-512Mi memory, 250m-500m CPU
- **MongoDB PVC**: 2Gi storage (local-path provisioner)

## Monitoring

Check resource usage:
```bash
kubectl top pods -n todo-app
```

## Troubleshooting

1. **Pods stuck in Pending**: Check storage class and node resources
2. **CrashLoopBackOff**: Check logs with `kubectl logs`
3. **Connection refused**: Verify MongoDB pod is ready before Todo API starts
4. **PVC not binding**: Ensure `standard` storage class exists
5. **LoadBalancer IP pending** (kind cluster): Kind doesn't support LoadBalancer by default. Use port-forward:
   ```bash
   kubectl port-forward svc/todo-api 8080:8080 -n todo-app
   ```
   Or install [MetalLB](https://metallb.universe.tf/) for LoadBalancer support.
6. **Service not reachable**: Check pod status and firewall rules
