# MERN Application — Kubernetes Deployment, HELM Chart & Jenkins CI/CD

**Project:** shopNow (Frontend + Backend)

This README provides a step-by-step guide to containerize, deploy and automate a MERN (MongoDB, Express, React, Node) application using Kubernetes, a HELM chart, and Jenkins pipeline automation. It includes example Kubernetes manifests, Helm chart structure and a Jenkins Groovy pipeline to build, push and deploy the app.

> **Assumptions & Notes**
>
> * You have two repositories (or two folders) — `frontend` and `backend` (the provided repo `https://github.com/mohanDevOps-arch/shopNow` contains both). Adjust paths/names if different.
> * Each component contains a `Dockerfile` (if not, the README includes example Dockerfiles you can adapt).
> * You have access to a Kubernetes cluster (minikube, kind, EKS, GKE, AKS, etc.) and `kubectl` configured for it.
> * You have Helm v3 installed and a Docker image registry (DockerHub, ECR, GCR, or private registry).
> * Jenkins server has Docker (or build agent with docker) and `helm` installed or available via tools.

---

## Table of contents

1. Overview
2. Repo structure (recommended)
3. Prerequisites
4. Build Docker images (local)
5. Kubernetes manifests — quick deploy (manifests for learning/testing)
6. Helm chart — structure & templates (recommended production flow)
7. MongoDB considerations (statefulset / external DB / chart dependency)
8. Jenkins Pipeline (Groovy) — CI/CD
9. How to deploy (commands)
10. Rollback & troubleshooting
11. Security & best practices
12. Challenges & solutions (example)
13. Appendix — example files (Dockerfile, manifests, templates, values)

---

## 1. Overview

This document helps you:

* Containerize FE (React) and BE (Node/Express) services
* Deploy them to Kubernetes with manifests
* Create a Helm chart to parameterize deployments
* Implement Jenkins pipeline to automate build, test, push and release

## 2. Recommended repository structure

```
shopNow/
├── charts/                # optional - top-level helm charts
├── helm/                  # single umbrella helm chart for app
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── mongodb-statefulset.yaml
│       ├── ingress.yaml
│       └── _helpers.tpl
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── ...
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── ...
├── k8s-manifests/         # optional - raw manifests for quick deploy/testing
│   ├── fe-deployment.yaml
│   ├── be-deployment.yaml
│   └── mongodb-statefulset.yaml
├── Jenkinsfile            # Declarative pipeline / Jenkinsfile
└── README.md              # this file
```

Adjust names and locations as needed.

---

## 3. Prerequisites

* Git
* Docker (or buildkit)
* kubectl configured for your cluster
* Helm v3
* A container registry and credentials (DockerHub, AWS ECR, GCP GCR, etc.)
* Jenkins server with a runner/agent that can build Docker images and execute `kubectl`/`helm` (or you can run pipeline steps on different agents)

Optional but recommended:

* cert-manager + kube-ingress-controller if you want TLS
* External DNS (if deploying to cloud with real domain)

---

## 4. Build Docker images (local quick test)

### Example frontend Dockerfile (React)

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . ./
RUN npm run build

# serve production build with lightweight web server
FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
# optionally copy a custom nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Example backend Dockerfile (Node/Express)

```dockerfile
# backend/Dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --production
COPY . .
ENV PORT=5000
EXPOSE 5000
CMD [ "node", "server.js" ]
```

Build and test locally:

```bash
# build
docker build -t <your-registry>/shopnow-frontend:1.0.0 ./frontend
docker build -t <your-registry>/shopnow-backend:1.0.0 ./backend

# run
docker run -p 8080:80 <your-registry>/shopnow-frontend:1.0.0
docker run -p 5000:5000 <your-registry>/shopnow-backend:1.0.0
```

Push to registry:

```bash
docker push <your-registry>/shopnow-frontend:1.0.0
docker push <your-registry>/shopnow-backend:1.0.0
```

---

## 5. Kubernetes manifests — quick deploy

Below are minimal manifests to test in cluster. Use them for learning and replace by Helm-managed resources for production.

### frontend-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopnow-frontend
  labels:
    app: shopnow-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shopnow-frontend
  template:
    metadata:
      labels:
        app: shopnow-frontend
    spec:
      containers:
      - name: frontend
        image: <YOUR_REGISTRY>/shopnow-frontend:1.0.0
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

### frontend-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shopnow-frontend-svc
spec:
  type: ClusterIP
  selector:
    app: shopnow-frontend
  ports:
    - port: 80
      targetPort: 80
```

### backend-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopnow-backend
  labels:
    app: shopnow-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shopnow-backend
  template:
    metadata:
      labels:
        app: shopnow-backend
    spec:
      containers:
      - name: backend
        image: <YOUR_REGISTRY>/shopnow-backend:1.0.0
        ports:
        - containerPort: 5000
        env:
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: shopnow-secrets
              key: mongo-uri
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
```

### backend-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shopnow-backend-svc
spec:
  type: ClusterIP
  selector:
    app: shopnow-backend
  ports:
    - port: 5000
      targetPort: 5000
```

### MongoDB (simple StatefulSet example)

For production use a cloud managed DB or an official stable chart (Bitnami).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: shopnow-mongodb
spec:
  serviceName: shopnow-mongodb
  replicas: 1
  selector:
    matchLabels:
      app: shopnow-mongodb
  template:
    metadata:
      labels:
        app: shopnow-mongodb
    spec:
      containers:
      - name: mongo
        image: mongo:6.0
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

Create a Kubernetes secret for MONGO_URI before applying backend:

```bash
kubectl create secret generic shopnow-secrets --from-literal=mongo-uri='mongodb://shopnow-mongodb-0.shopnow-mongodb.default.svc.cluster.local:27017/shopnow'
```

Apply manifests:

```bash
kubectl apply -f k8s-manifests/
```

---

## 6. Helm Chart — structure & templates

Create a Helm chart to parameterize images, replica counts, resource requests/limits, and external services.

```
helm/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── secret.yaml
    ├── mongodb-statefulset.yaml (optional)
    ├── ingress.yaml
    └── hpa.yaml
```

### Example `Chart.yaml`

```yaml
apiVersion: v2
name: shopnow
description: A Helm chart for shopNow MERN app
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### Example `values.yaml` (key parts)

```yaml
replicaCount: 2
image:
  registry: docker.io/youraccount
  frontend:
    repository: shopnow-frontend
    tag: "1.0.0"
  backend:
    repository: shopnow-backend
    tag: "1.0.0"
service:
  backend:
    port: 5000
    type: ClusterIP
  frontend:
    port: 80
    type: ClusterIP
mongodb:
  enabled: true
  persistence:
    size: 5Gi
ingress:
  enabled: false
  hosts:
    - host: shopnow.example.com
      paths:
        - /
resources: {}

# imagePullSecrets, nodeSelector, tolerations etc.
```

### Sample `templates/backend-deployment.yaml` (Helm templating)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "shopnow.fullname" . }}-backend
  labels:
    app: {{ include "shopnow.name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "shopnow.name" . }}-backend
  template:
    metadata:
      labels:
        app: {{ include "shopnow.name" . }}-backend
    spec:
      containers:
      - name: backend
        image: "{{ .Values.image.registry }}/{{ .Values.image.backend.repository }}:{{ .Values.image.backend.tag }}"
        ports:
        - containerPort: {{ .Values.service.backend.port }}
        env:
        - name: MONGO_URI
          value: {{ .Values.mongodb.uri | quote }}
```

> Note: Create helper templates in `_helpers.tpl` for `fullname` and `name`.

---

## 7. MongoDB considerations

Options:

* **Use Helm subchart**: Add Bitnami MongoDB as a dependency in `Chart.yaml`. This is the easiest for production-like local deployments.
* **Use managed DB** (recommended for production): e.g., Amazon DocumentDB / MongoDB Atlas.
* **StatefulSet + PVC**: Example included above; ensure proper storage class in your cluster.

When using an external DB, store credentials in Kubernetes `Secret` and reference via env.

---

## 8. Jenkins Pipeline (Declarative / Groovy)

Below is an example Jenkinsfile that builds both images, pushes them, and deploys the Helm chart.

> Make sure Jenkins has credentials configured: `DOCKER_REGISTRY` (username/password or token) and `KUBE_CONFIG` (or configure `kubectl` on agent). We'll reference Jenkins credentials by id.

```groovy
pipeline {
  agent any
  environment {
    REGISTRY = 'docker.io/youraccount'
    FRONTEND_IMAGE = "${REGISTRY}/shopnow-frontend"
    BACKEND_IMAGE  = "${REGISTRY}/shopnow-backend"
    HELM_RELEASE = 'shopnow'
    HELM_CHART_DIR = 'helm'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/mohanDevOps-arch/shopNow.git']]])
      }
    }

    stage('Build Frontend') {
      steps {
        dir('frontend') {
          sh 'docker build -t ${FRONTEND_IMAGE}:${GIT_COMMIT} .'
        }
      }
    }

    stage('Build Backend') {
      steps {
        dir('backend') {
          sh 'docker build -t ${BACKEND_IMAGE}:${GIT_COMMIT} .'
        }
      }
    }

    stage('Push Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDS', passwordVariable: 'DOCKER_PSW', usernameVariable: 'DOCKER_USER')]) {
          sh "echo $DOCKER_PSW | docker login -u $DOCKER_USER --password-stdin"
          sh "docker push ${FRONTEND_IMAGE}:${GIT_COMMIT}"
          sh "docker push ${BACKEND_IMAGE}:${GIT_COMMIT}"
        }
      }
    }

    stage('Helm Deploy') {
      steps {
        withCredentials([file(credentialsId: 'KUBECONFIG_FILE', variable: 'KUBECONFIG')]) {
          sh 'export KUBECONFIG=$KUBECONFIG'
          sh "helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_DIR} --set image.frontend.tag=${GIT_COMMIT} --set image.backend.tag=${GIT_COMMIT} --namespace default"
        }
      }
    }
  }
  post {
    success {
      echo 'Deployed successfully'
    }
    failure {
      echo 'Build or deployment failed'
    }
  }
}
```

**Notes**:

* Use proper `credentialsId` existing in your Jenkins (DOCKERHUB_CREDS, KUBECONFIG_FILE etc.).
* You can add unit tests or lint stages before pushing images.
* Consider tagging images with semantic tags and `latest` separately.

---

## 9. How to deploy (commands)

Local quick test (manifests):

```bash
kubectl apply -f k8s-manifests/
```

With Helm:

```bash
cd helm
helm install shopnow . -f values.yaml
# or upgrade
helm upgrade --install shopnow . -f values.yaml --set image.backend.tag=1.0.1
```

To view status:

```bash
kubectl get pods
kubectl get svc
kubectl logs deployment/shopnow-backend
```

---

## 10. Rollback & Troubleshooting

* Check events: `kubectl describe pod <pod>` and `kubectl get events --sort-by='.metadata.creationTimestamp'`
* Helm rollback: `helm rollback shopnow 1` (where `1` is revision)
* If pods crash-loop, `kubectl logs <pod>` and `kubectl describe pod <pod>` to inspect probes and envs.
* Image pull issues: check `imagePullSecrets` and `kubectl get secret`.

---

## 11. Security & best practices

* Store DB credentials in `Secrets`, not `ConfigMaps`.
* Use `imagePullSecrets` for private registries.
* Limit RBAC access and don't run containers as root.
* Use resource requests/limits and pod disruption budgets.
* Use readiness/liveness probes for proper rolling updates.
* For production, use managed DB or production-ready Helm charts (Bitnami) and backup strategies.

---

## 12. Example challenges & solutions

* **Challenge:** Backend couldn't reach MongoDB in-cluster due to wrong service FQDN.
  **Solution:** Use the StatefulSet stable hostname `shopnow-mongodb-0.shopnow-mongodb.default.svc.cluster.local:27017` or use a Headless Service for stable DNS.

* **Challenge:** Frontend build fails on CI agent lacking Node version.
  **Solution:** Use a Docker build stage or run npm build inside Docker builder image to keep agent lightweight.

* **Challenge:** Helm values not overriding child chart value.
  **Solution:** Use `--set` with proper path or update `values.yaml` and confirm `helm get values`.

---

## 13. Appendix — example templates & snippets

(Short excerpts; full files should exist in `helm/templates` and `k8s-manifests`.)

### Example `secret.yaml` (Helm)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "shopnow.fullname" . }}-secrets
type: Opaque
stringData:
  mongo-uri: {{ .Values.mongodb.uri | quote }}
```

### Example `ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "shopnow.fullname" . }}-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: {{ .Values.ingress.hosts[0].host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "shopnow.fullname" . }}-frontend-svc
                port:
                  number: {{ .Values.service.frontend.port }}
{{- end }}
```

---

## Final notes

* This README is a template and must be adjusted for actual repo contents (actual Dockerfiles, ports, health endpoints, and environment variables).
* If you want, I can:

  * generate the Helm chart templates (actual files) for you
  * create Kubernetes manifest files (full)
  * provide a tested Jenkins declarative pipeline adapted to your Jenkins credentials

Good luck — once you try the steps you will see where small adjustments are needed and I can help iterate on the exact files.

---

*End of README*
