# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

This is a **Spring Boot 3.2.5 Todo REST API** with MongoDB backend, demonstrating multiple deployment strategies (Docker Compose, Kubernetes manifests, Helm chart). The project emphasizes Kubernetes best practices and security (least-privilege access, resource limits, health probes).

**Tech Stack:** Java 21, Spring Boot, Spring Data MongoDB, Lombok, Maven, Docker, Kubernetes, Helm.

---

## Common Development Commands

### Building & Running

```bash
# Build the application JAR
./mvnw clean package

# Run locally with Spring Boot dev tools
./mvnw spring-boot:run

# Run the built JAR (requires MongoDB running)
java -jar target/todo-1.0.0.jar

# Build Docker image
docker build -t mrnanda/to-do-api:2.0 .

# Run with Docker Compose (spins up MongoDB + app)
docker-compose up -d
docker-compose logs -f todo-api
docker-compose down
```

### Testing

```bash
# Run all tests
./mvnw test

# Run a single test class
./mvnw test -Dtest=TodoApplicationTests

# Run a single test method
./mvnw test -Dtest=TodoApplicationTests#contextLoads

# Run tests with verbose output
./mvnw test -X
```

### Database & API Testing

```bash
# Access MongoDB shell in Docker Compose
docker-compose exec mongodb mongosh todo -u todo-user -p todo-pass

# Test API endpoints (after deployment)
curl http://localhost:8080/api/todos
curl -X POST http://localhost:8080/api/todos -H "Content-Type: application/json" -d '{"title":"Test","completed":false}'
```

### Kubernetes Deployment

```bash
# Using raw manifests
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-mongodb-secret.yaml
kubectl apply -f k8s/02-mongodb-configmap.yaml
kubectl apply -f k8s/03-mongodb-service.yaml
kubectl apply -f k8s/04-mongodb-statefulset.yaml
kubectl wait --for=condition=ready pod/mongodb-0 -n todo-app --timeout=120s
kubectl apply -f k8s/05-todo-api-secret.yaml
kubectl apply -f k8s/06-todo-api-configmap.yaml
kubectl apply -f k8s/07-todo-api-deployment.yaml
kubectl apply -f k8s/08-todo-api-service.yaml

# Using Helm (recommended)
cd helm/todo-api
helm dependency build
helm install todo-api . --namespace todo-app --create-namespace
helm status todo-api -n todo-app
helm upgrade todo-api . -n todo-app
helm rollback todo-api 1 -n todo-app
```

---

## Architecture & Code Structure

### High-Level Architecture

The application follows standard Spring Boot layered architecture:

```
Controller (REST API) → Service (Business Logic) → Repository (MongoDB)
```

**Key Patterns:**
- `@RestController` with `/api/todos` endpoints
- `@Service` layer handles CRUD operations
- `MongoRepository` interface provides data access
- `@Document` model class with Jackson/Lombok
- Configuration externalized via environment variables (K8s) or `application.properties` (local)

### Important: Configuration Sources

**Local development:** `src/main/resources/application.properties` contains hardcoded MongoDB credentials for Docker Compose.

**Kubernetes:** The application reads MongoDB connection via environment variables (SPRING_DATA_MONGODB_*). The `application.properties` in the JAR uses env var substitution via ConfigMap (see `helm/todo-api/templates/configmap.yaml` and `k8s/06-todo-api-configmap.yaml`).

When modifying database configuration, remember to update both:
1. `application.properties` (for local Docker Compose)
2. K8s ConfigMap/Helm values (for cluster deployment)

---

## Key File Locations

**Application Code:**
- `src/main/java/com/pavan/todo/` - All source code
- `TodoApplication.java` - Main Spring Boot entry point
- `controllers/TodoController.java` - REST endpoints (GET, POST, PUT, DELETE)
- `services/TodoService.java` - Business logic
- `models/Todo.java` - MongoDB document model
- `repositories/TodoRepository.java` - Spring Data interface

**Configuration:**
- `pom.xml` - Maven dependencies, Java 21, Spring Boot 3.2.5
- `src/main/resources/application.properties` - Local DB config
- `helm/todo-api/values.yaml` - Helm configuration (image, resources, probes, MongoDB settings)
- `k8s/06-todo-api-configmap.yaml` - K8s ConfigMap for app properties

**Deployment:**
- `Dockerfile` - Container image (Temurin JDK 21 base)
- `compose.yaml` - Docker Compose for local dev (MongoDB + app)
- `k8s/` - Raw Kubernetes manifests (namespace, secrets, statefulset, deployment, service)
- `helm/todo-api/` - Helm chart with MongoDB dependency

---

## Testing Approach

- JUnit 5 with `@SpringBootTest` for integration tests
- Default test class: `TodoApplicationTests` (contextLoads)
- No repository-specific unit tests currently; MongoDB is tested via integration
- To add tests, place them in `src/test/java/com/pavan/todo/`

---

## Deployment Patterns

Three deployment methods exist:

1. **Docker Compose** - Simplest for local development; uses root MongoDB user with password `password`
2. **K8s Manifests** - Direct resource application; includes manual secret creation and init-based user setup in MongoDB StatefulSet
3. **Helm** - Production-ready; uses cloudpirates/mongodb chart with auto-generated passwords and custom user creation via `values.yaml`

**Helm is the recommended approach** for repeatable installations and upgrades. The chart handles MongoDB as a dependency and securely generates credentials.

---

## Database Configuration Nuances

- MongoDB connection uses Spring's relaxed binding: environment variables like `SPRING_DATA_MONGODB_HOST` map to properties
- The Docker Compose setup uses `root/password` credentials in `application.properties`
- In K8s/Helm, credentials come from Kubernetes Secrets referenced in the deployment ConfigMap
- The `TodoRepository` extends `MongoRepository<Todo, String>` - no custom queries needed for basic CRUD
- The `Todo` model uses `@Indexed(unique = true)` on title; MongoDB enforces uniqueness

---

## Environment Differences

**Local (Docker Compose):**
- MongoDB host: `localhost` (actually service name `mongodb` in compose network)
- DB credentials: root/password
- Auth DB: `admin`
- Application port: `8080`

**Kubernetes:**
- MongoDB host: `mongodb` (K8s service DNS)
- DB credentials: `todo-user`/`todo-pass` (K8s secret)
- Auth DB: `todo`
- Application port: `8080`

---

## How to Make Changes Safely

1. **Code changes**: Edit files under `src/main/java/`, rebuild with `./mvnw package`, restart deployment
2. **Configuration changes**: Update appropriate config (Helm values or K8s ConfigMap), then `helm upgrade` or `kubectl apply`
3. **Adding dependencies**: Add to `pom.xml`, run `./mvnw clean package`
4. **Changing API**: Update `TodoController`, ensure backward compatibility or version the endpoint
5. **Schema changes**: Modify `Todo.java`, MongoDB will adapt (no migrations needed)

---

## Troubleshooting Tips

- **App can't connect to MongoDB**: Check environment variables in deployment (`kubectl describe pod <pod>` or `helm get values`)
- **Pods pending**: Usually storage class issue (`kubectl get storageclass`)
- **404 on /api/todos**: Check pod logs (`kubectl logs -f <pod>`), ensure app started successfully (MongoDB connection)
- **Helm dependency issues**: The MongoDB chart comes from Docker Hub OCI; you may need `docker login` for rate limits

---

## Notes

- The project uses **Lombok** (`@Getter`, `@Setter`) to reduce boilerplate; IDE must have Lombok plugin installed
- **CORS is enabled** globally (`@CrossOrigin` on controller)
- **Resource limits** are set to 250-500m CPU and 256-512Mi memory; adjust if needed in Helm values or K8s manifest
- The MongoDB StatefulSet uses an **init container** with custom user creation script (K8s manifests only; Helm chart uses `customUsers` feature)