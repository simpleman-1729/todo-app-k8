1.Seperate k8 manifests
2.ConfigMaps and secret added 
3.Added Docker file user for non root user 
4.Added kustomizationfile for easy apply
5.Ingress and Added the todo.local 127.0.0.0 in host file for ingress

===== .\.env =====

POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=todo
DB_HOST=db
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=todo

===== .\docker-compose.yaml =====

services:

  db:
    image: postgres:14
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  backend:
    build: ./backend
    env_file:
      - .env
    ports:
      - "3000:3000"
    depends_on:
     db:
      condition: service_healthy

  frontend:
    build: ./frontend
    ports:
      - "8080:80"
    depends_on:
      - backend

volumes:
  postgres_data:

===== .\backend\Dockerfile =====

FROM node:18-alpine

WORKDIR /app

RUN addgroup -S nodegroup && adduser -S nodeuser -G nodegroup

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

RUN chown -R nodeuser:nodegroup /app

USER nodeuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
CMD wget -qO- http://localhost:3000/api/health || exit 1

CMD ["node","server.js"]

===== .\frontend\Dockerfile =====

FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY nginx.conf /etc/nginx/conf.d/default.conf


===== .\frontend\nginx.conf =====

server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri /index.html;
    }

    location /api {
        proxy_pass http://backend:3000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

===== .\k8s\kustomization.yaml =====

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Namespace to apply everything
namespace: todo

# All resource files (order matters for dependencies)
resources:
  - namespace/namespace.yaml

  # Configs
  - config/configmap.yaml
  - config/secret.yaml
  - config/postgres-init.yaml   

  # Database
  - database/service.yaml
  - database/statefulset.yaml

  # Backend
  - backend/service.yaml
  - backend/deployment.yaml
  - backend/hpa.yaml

  # Frontend
  - frontend/service.yaml
  - frontend/deployment.yaml

  # Ingress
  - ingress/ingress.yaml

===== .\k8s\backend\deployment.yaml =====

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: todo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: backend:latest
        imagePullPolicy: IfNotPresent
        resources:         
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "300m"
        ports:
          - containerPort: 3000
        env:
          - name: DB_HOST
            value: postgres
          - name: DB_USER
            valueFrom:
              configMapKeyRef:
                name: postgres-config
                key: DB_USER
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: POSTGRES_PASSWORD
          - name: DB_NAME
            valueFrom:
              configMapKeyRef:
                name: postgres-config
                key: DB_NAME
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5   

===== .\k8s\backend\hpa.yaml =====

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

===== .\k8s\backend\service.yaml =====

apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: todo
spec:
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000

===== .\k8s\config\configmap.yaml =====

apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: todo
data:
  POSTGRES_DB: todo
  POSTGRES_USER: postgres
  DB_HOST: postgres
  DB_NAME: todo
  DB_USER: postgres

===== .\k8s\config\postgres-init.yaml =====

apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
  namespace: todo
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS todos (
      id SERIAL PRIMARY KEY,
      task TEXT NOT NULL
    );

===== .\k8s\config\secret.yaml =====

apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: todo
type: Opaque
data:
  POSTGRES_PASSWORD: cG9zdGdyZXM=

===== .\k8s\database\service.yaml =====

apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: todo
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
  clusterIP: None


===== .\k8s\database\statefulset.yaml =====

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: todo
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
          - containerPort: 5432

        env:
          - name: POSTGRES_DB
            valueFrom:
              configMapKeyRef:
                name: postgres-config
                key: POSTGRES_DB
          - name: POSTGRES_USER
            valueFrom:
              configMapKeyRef:
                name: postgres-config
                key: POSTGRES_USER
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: POSTGRES_PASSWORD

        volumeMounts:
          - name: postgres-data
            mountPath: /var/lib/postgresql/data
          - name: init-script
            mountPath: /docker-entrypoint-initdb.d

      volumes:  
      - name: init-script
        configMap:
          name: postgres-init

  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: hostpath   
      resources:
        requests:
          storage: 1Gi

===== .\k8s\frontend\deployment.yaml =====

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: todo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: frontend:latest
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "300m"
        ports:
          - containerPort: 80

===== .\k8s\frontend\service.yaml =====

apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: todo
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80

===== .\k8s\ingress\ingress.yaml =====

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo-ingress
  namespace: todo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: todo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80

===== .\k8s\namespace\namespace.yaml =====

apiVersion: v1
kind: Namespace
metadata:
  name: todo