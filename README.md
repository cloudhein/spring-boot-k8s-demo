# Todo API - Kubernetes Helm Development

A Spring Boot Todo API application with MongoDB backend, featuring multiple deployment strategies: Docker Compose, raw Kubernetes manifests, and Helm chart.

## 🎯 Project Overview

This project demonstrates a **Todo REST API** built with:
- **Backend**: Spring Boot 3.2.5 (Java 21)
- **Database**: MongoDB 8.0
- **Architecture**: Containerized microservice with persistent data storage
- **Security**: Least-privilege MongoDB user configuration

### Features
- CRUD operations for Todo items
- MongoDB repository with Spring Data
- Health check endpoints for liveness/readiness probes
- Production-ready with resource limits and probes
- Multiple deployment options for different environments

---

## 📁 Repository Structure

```
.
├── src/                    # Spring Boot application source code
│   ├── main/
│   │   ├── java/com/pavan/todo/
│   │   │   ├── TodoApplication.java
│   │   │   ├── controllers/TodoController.java
│   │   │   ├── models/Todo.java
│   │   │   ├── repositories/TodoRepository.java
│   │   │   └── services/TodoService.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/...
├── Dockerfile              # Container image definition
├── pom.xml                 # Maven build configuration
├── compose.yaml            # Docker Compose deployment
├── k8s/                    # Raw Kubernetes manifests
│   ├── 00-namespace.yaml
│   ├── 01-mongodb-secret.yaml
│   ├── 02-mongodb-configmap.yaml
│   ├── 03-mongodb-service.yaml
│   ├── 04-mongodb-statefulset.yaml
│   ├── 05-todo-api-secret.yaml
│   ├── 06-todo-api-configmap.yaml
│   ├── 07-todo-api-deployment.yaml
│   ├── 08-todo-api-service.yaml
│   └── README.md
└── helm/                   # Helm chart for Todo API
    └── todo-api/
        ├── Chart.yaml
        ├── values.yaml
        ├── templates/
        │   ├── _helpers.tpl
        │   ├── configmap.yaml
        │   ├── deployment.yaml
        │   ├── ingress.yaml
        │   ├── service.yaml
        └── README.md
```

---

## 🚀 Deployment Options

Choose the deployment method based on your environment:

| Method | Best For | Complexity | MongoDB Management |
|--------|----------|------------|-------------------|
| **Docker Compose** | Local development, quick testing | Low | Auto-provisioned via Docker volumes |
| **K8s Manifests** | Learning, full control, custom tweaks | Medium | Deployed with app in k8s/ |
| **Helm Chart** | Production, CI/CD, repeatable installs | Low (after chart ready) | Dependency chart (cloudpirates/mongodb) |

---

### 1️⃣ Docker Compose (Simplest - Local Dev)

**Use when**: You want to quickly run the app locally for development or testing.

#### Prerequisites
- Docker Engine or Docker Desktop
- Docker Compose v2

#### Steps

```bash
# Start MongoDB and Todo API
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f todo-api
docker-compose logs -f mongodb

# Access the API
curl http://localhost:8080/api/todos

# Stop and remove containers
docker-compose down

# Remove volumes (deletes data)
docker-compose down -v
```

**Services:**
- MongoDB: `mongodb://root:password@localhost:27017/todo?authSource=admin`
- Todo API: `http://localhost:8080`
- Persistence: Docker named volume `mongodb_data`

**See**: [k8s/README.md](./k8s/README.md) for detailed Kubernetes manifest deployment guide.

---

### 2️⃣ Kubernetes Manifests (Full Control)

**Use when**: You want direct control over Kubernetes resources, need to customize low-level settings, or are learning Kubernetes.

#### Prerequisites
- Kubernetes cluster (kind, minikube, or cloud)
- `kubectl` configured
- Storage class available for PVCs

#### Installation Steps

```bash
# Create namespace
kubectl apply -f k8s/00-namespace.yaml

# Deploy MongoDB (includes secret, configmap, service, statefulset)
kubectl apply -f k8s/01-mongodb-secret.yaml
kubectl apply -f k8s/02-mongodb-configmap.yaml
kubectl apply -f k8s/03-mongodb-service.yaml
kubectl apply -f k8s/04-mongodb-statefulset.yaml

# Wait for MongoDB to be ready (important!)
kubectl wait --for=condition=ready pod/mongodb-0 -n todo-app --timeout=120s

# Deploy Todo API
kubectl apply -f k8s/05-todo-api-secret.yaml
kubectl apply -f k8s/06-todo-api-configmap.yaml
kubectl apply -f k8s/07-todo-api-deployment.yaml
kubectl apply -f k8s/08-todo-api-service.yaml

# Verify deployment
kubectl get all -n todo-app
```

#### Security Features

- ✅ **Least-privilege MongoDB user**: `todo-user` with `readWrite` on `todo` database only
- ✅ **Separate secrets**: MongoDB root credentials and app credentials stored separately
- ✅ **Init script**: Automatically creates app user with proper permissions
- ✅ **Resource limits**: CPU and memory constraints for both containers

**Manifest Details**:
- **MongoDB StatefulSet** (`04-mongodb-statefulset.yaml`):
  - Uses custom `init.sh` to create `todo-user` from environment variables
  - Mounts ConfigMap for `mongod.conf` and init script
  - Persistent storage via PVC with `standard` storage class

- **Todo Deployment** (`07-todo-api-deployment.yaml`):
  - Uses `todo-api-secret` for credentials (`todo-user`/`todo-pass`)
  - ConfigMap for `application.properties`
  - Liveness/readiness probes at `/api/todos`
  - Resource limits: 256-512Mi memory, 250-500m CPU

**See**: [k8s/README.md](./k8s/README.md) for more details.

---

### 3️⃣ Helm Chart (Recommended for Production)

**Use when**: You want package management, versioning, easy upgrades, and dependency handling.

#### Prerequisites
- Helm 3.x installed
- Kubernetes cluster
- Access to Docker Hub OCI registry (for MongoDB chart dependency)

#### Installation Steps

```bash
# Navigate to Helm chart directory
cd helm/todo-api

# Build dependencies (downloads MongoDB chart)
helm dependency build

# Install the chart
helm install todo-api . --namespace todo-app --create-namespace

# Wait for deployment
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=todo-api -n todo-app --timeout=300s

# Check status
helm status todo-api -n todo-app
kubectl get all -n todo-app
```

#### What's Different?

- **MongoDB** is deployed as a Helm dependency (cloudpirates/mongodb chart, v0.14.2)
- **Auto-generated passwords**: Secure random passwords (no plaintext in Git)
- **Custom user creation**: Uses `mongodb.customUsers` in `values.yaml` to create `todo-user`
- **Simpler config**: All settings in `values.yaml`; no separate secret manifest needed
- **Unified management**: Single `helm upgrade` / `helm rollback` for full stack

#### Key Configuration in `values.yaml`

```yaml
mongodb:
  enabled: true
  replicaCount: 1
  auth:
    rootUsername: admin
    rootPassword: ""  # Auto-generated
    appUsername: todo-user
    appDatabase: todo
    appAuthDatabase: todo
  customUsers:
    - name: todo-user
      database: todo
      roles:
        - readWrite
  persistence:
    size: 2Gi
    storageClass: standard

app:
  mongodb:
    username: ""  # Uses mongodb.auth.appUsername by default
    database: ""  # Uses mongodb.auth.appDatabase
    authDatabase: ""  # Uses mongodb.auth.appAuthDatabase
```

#### Upgrade & Rollback

```bash
# Upgrade with custom values
helm upgrade todo-api . -f custom-values.yaml -n todo-app

# Dry-run preview
helm upgrade todo-api . -f custom-values.yaml -n todo-app --dry-run --debug

# Rollback
helm rollback todo-api 1 -n todo-app
```

**See**: [helm/todo-api/README.md](./helm/todo-api/README.md) for full Helm documentation.

---

## 🔗 Connecting to MongoDB

### Connection String Format

```
mongodb://<username>:<password>@<host>:<port>/<database>?authSource=<auth-db>
```

### By Deployment Method

| Method | Host | Port | Username | Password | Database | Auth DB |
|--------|------|------|----------|----------|----------|---------|
| Docker Compose | `localhost` | `27017` | `root` | `password` | `todo` | `admin` |
| K8s Manifests | `mongodb` | `27017` | `todo-user` | `todo-pass` | `todo` | `todo` |
| Helm Chart | `<release>-mongodb` | `27017` | `todo-user` | (auto-generated) | `todo` | `todo` |

---

## 🧪 Testing the API

Once deployed using any method:

```bash
# Get the service endpoint
# For LoadBalancer (cloud):
EXTERNAL_IP=$(kubectl get svc todo-api -n todo-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# For port-forward (local):
kubectl port-forward svc/todo-api 8080:8080 -n todo-app &

# Create a todo
curl -X POST http://$EXTERNAL_IP:8080/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Kubernetes","completed":false}'

# List todos
curl http://$EXTERNAL_IP:8080/api/todos

# Get单个todo (replace <id>)
curl http://$EXTERNAL_IP:8080/api/todos/<id>

# Update todo
curl -X PUT http://$EXTERNAL_IP:8080/api/todos/<id> \
  -H "Content-Type: application/json" \
  -d '{"title":"Updated title","completed":true}'

# Delete todo
curl -X DELETE http://$EXTERNAL_IP:8080/api/todos/<id>
```

---

## 🔒 Security Best Practices

### Applied in This Project

1. **Least-privilege MongoDB user**: App uses `todo-user` with only `readWrite` on `todo` database (not root)
2. **Auto-generated passwords**: Helm chart generates strong random passwords
3. **Secrets management**: Credentials stored in Kubernetes Secrets (base64 encoded, not encrypted by default - for production use external secrets manager)
4. **Resource limits**: Prevent resource exhaustion attacks
5. **Non-root containers**: MongoDB runs as UID 999 (non-root)

### Before Production

- [ ] Enable encryption-at-rest for MongoDB (storage-level or MongoDB Enterprise)
- [ ] Enable TLS/SSL for MongoDB connections
- [ ] Use external secrets manager (HashiCorp Vault, AWS Secrets Manager)
- [ ] Implement network policies to restrict pod-to-pod communication
- [ ] Set up audit logging for MongoDB
- [ ] Regular backups of MongoDB data
- [ ] Image scanning for vulnerabilities
- [ ] Use read-only root filesystem where possible

---

## 🛠️ Development

### Building the Application

```bash
# Clone the repository
git clone <repository-url>
cd pov-kubernetes/helm-development/todo-api

# Build JAR with Maven
./mvnw clean package

# Build Docker image
docker build -t mrnanda/to-do-api:2.0 .

# Push to registry (optional)
docker push mrnanda/to-do-api:2.0
```

### Running Locally (without Docker)

```bash
# Ensure MongoDB is running (e.g., via Docker Compose)
docker-compose up -d mongodb

# Run Spring Boot app
./mvnw spring-boot:run

# Or run the JAR
java -jar target/todo-1.0.0.jar
```

---

## 📚 Learning Resources

### What This Project Teaches

- **Spring Boot**: REST controllers, Spring Data MongoDB, dependency injection
- **Docker**: Multi-stage builds, Docker Compose networking and volumes
- **Kubernetes**: Pods, StatefulSets, Services, PersistentVolumeClaims, ConfigMaps, Secrets
- **Helm**: Chart structure, dependencies, templating, values management, hooks
- **Security**: Least privilege, secret management, resource constraints

### Key Concepts Demonstrated

1. **Init Containers**: MongoDB StatefulSet uses init container to set up scripts
2. **Custom User Creation**: `customUsers` in cloudpirates MongoDB chart
3. **Secret References**: Using `valueFrom.secretKeyRef` in deployments
4. **ConfigMap for Properties**: Spring Boot properties externalized
5. **Liveness/Readiness Probes**: Health checking strategies
6. **Persistent Storage**: StatefulSet with PVC for database data
7. **Helm Dependencies**: Managing subcharts with `helm dependency build`

---

## 🐛 Troubleshooting

### Common Issues

**1. Pods stuck in Pending**
```bash
kubectl describe pod <pod-name> -n todo-app
kubectl get pvc -n todo-app
# Ensure storage class exists: kubectl get storageclass
```

**2. Authentication Failed (MongoDB)**
```bash
# Verify custom user exists
kubectl exec -it mongodb-0 -n todo-app -- mongosh admin -u admin -p <password> --eval "db.getSiblingDB('todo').getUser('todo-user')"
# If missing, check init script logs in MongoDB pod
```

**3. LoadBalancer IP Pending**
```bash
# On local clusters (kind/minikube), use port-forward:
kubectl port-forward svc/todo-api 8080:8080 -n todo-app
```

**4. Helm dependency download fails**
```bash
# The cloudpirates MongoDB chart is from Docker Hub OCI. Ensure you can access Docker Hub.
# You may need to login: docker login
# Then run: helm dependency build --skip-refresh
```

---

## 📄 License

[Specify your license here]

## 🙏 Acknowledgments

- Spring Boot Team
- MongoDB Inc.
- cloudpirates Helm chart maintainers
- Kubernetes community

