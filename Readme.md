📝 Todo App – Docker + Kubernetes (Local Setup)

A full-stack 3 Tier Application deployed using Docker Compose and Kubernetes (local cluster) with production-style practices.

---

🚀 Tech Stack

* Frontend: Nginx (Static HTML, JS)
* Backend: Node.js (Express)
* Database: PostgreSQL
* Containerization: Docker
* Orchestration: Kubernetes (K8s)
* Configuration Management: ConfigMaps & Secrets
* Scaling: Horizontal Pod Autoscaler (HPA)
* Ingress: NGINX Ingress Controller

---

## 📁 Project Structure

```
.
├── backend/
├── frontend/
├── database/
├── docker-compose.yaml
├── .env
└── k8s/
    ├── namespace/
    ├── config/
    ├── database/
    ├── backend/
    ├── frontend/
    └── ingress/
```

---

⚙️ Features Implemented

✅ Dockerized full-stack app
✅ Non-root user in backend container (security best practice)
✅ PostgreSQL with persistent storage
✅ Kubernetes manifests split by components
✅ ConfigMaps & Secrets for configuration
✅ StatefulSet for database
✅ Deployments for frontend & backend
✅ Liveness & Readiness probes
✅ Horizontal Pod Autoscaler (HPA)
✅ Ingress with custom domain (`todo.local`)
✅ Kustomize for easy deployment

---

🐳 Run with Docker Compose

1. Start Application

```bash
docker-compose up --build
```

2. Access App

```
http://localhost:8080
```

---

☸️ Kubernetes Deployment (Local)

> Deployed with  Docker Desktop single node Kubernetes cluster.

1. Create Cluster 

2. Enable Ingress

3. Apply Kustomize
kubectl apply -k k8s/
---

🌐 Access Application via Ingress

Update Hosts File

Add this line:

```
Add to host path

C:\Windows\System32\drivers\etc\hosts

127.0.0.1 todo.local
```

Then open:
```
http://todo.local
```
---

🔐 Environment Variables

Defined in .env.example 

```
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=todo
DB_HOST=postgres
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=todo
```
---
🧠 Kubernetes Architecture

Database:
StatefulSet
* Persistent Volume (1Gi)
* Init SQL via ConfigMap

Backend:
* Deployment (replicas: 2)
* Connected via Service
* Auto-scaled using HPA

Frontend:
* Deployment (replicas: 2)
* Served via Nginx

Networking:
* Services for internal communication
* Ingress for external access
---
📊 Scaling (HPA)

Backend automatically scales based on CPU:

* Min Pods: 2
* Max Pods: 5
* Target CPU: 70%

---
❤️ Health Checks

* Backend exposes:
```
/api/health
```
Used for:
* Liveness Probe
* Readiness Probe
---

🧪 API Endpoints

| Method | Endpoint       | Description   |
| ------ | -------------- | ------------- |
| GET    | /api/health    | Health check  |
| GET    | /api/todos     | Get all todos |
| POST   | /api/todos     | Create todo   |
| DELETE | /api/todos/:id | Delete todo   |

---

🚧 Improvements (Future Scope)

* Add CI/CD (GitHub Actions)
* Use Helm charts
* Add HTTPS with cert-manager
* Add monitoring (Prometheus + Grafana)

---

👨‍💻 Author

Karthick Vignesh K

---

