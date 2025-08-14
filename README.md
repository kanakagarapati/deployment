# MERN on Kubernetes with Helm & Jenkins

This guide walks you through **building, deploying, and operating** a MERN (MongoDB, Express.js, React.js, Node.js) app on **Kubernetes** using **Helm** and **Jenkins**, with **frontend on port 80** and **backend on port 3000**. It includes **best practices**, **troubleshooting**, and both **local (Minikube)** and **CI/CD** flows.

> **Docker Hub namespace:** `garapatimanoj`  
> **Images:** `garapatimanoj/learner-fe`, `garapatimanoj/learner-be`  
> **K8s namespace (dev):** `learner-dev`  
> **Ingress host (dev):** `learner.local` (or `learner.test` if `.local` is blocked on macOS)  
> **MongoDB URI:** Store in a Kubernetes **Secret**. **Do not** commit the URI to Git.


---

## 0) Prerequisites

- macOS with **Docker Desktop** and **kubectl**, **helm**, **git**
- **Minikube** (for local dev):  
  ```bash
  brew install minikube kubectl helm
  ```
- **Jenkins** (optional for CI/CD) with Docker installed on the agent, and credentials:
  - `dockerhub-creds` → Docker Hub username/password
  - `kubeconfig` → Kubeconfig secret file for your cluster (dev/prod)

---

## 1) Repository Layout

Either keep FE/BE repos separate and add a deployment repo, or place everything in one repo. Example:

```
/deployment
  /charts/learner-report        # Helm chart
  /environments
    values-dev.yaml
    values-prod.yaml
  Jenkinsfile

/frontend
  Dockerfile
  nginx.conf

/backend
  Dockerfile
  k8s-healthcheck.js (optional)
```

---

## 2) Dockerfiles

### 2.1 Frontend (`/frontend/Dockerfile`)
```dockerfile
# ---- build stage ----
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ENV REACT_APP_API_BASE_URL=/api
RUN npm run build

# ---- run stage ----
FROM nginx:alpine
# For strict non-root, see section 9.2 to serve on 8080
RUN adduser -D -H -u 10001 appuser
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/build /usr/share/nginx/html
USER 10001
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**`/frontend/nginx.conf`**
```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri /index.html;
  }
  # NOTE: /api is routed by Kubernetes Ingress to the backend service.
}
```

### 2.2 Backend (`/backend/Dockerfile`)
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
ENV NODE_ENV=production
EXPOSE 3000
# Run as non-root
RUN adduser -D -H -u 10002 appuser
USER 10002
CMD ["node", "server.js"]
```

**Backend `server.js` health route (example):**
```js
// Add a fast health endpoint that doesn't touch DB
app.get('/health', (_req, res) => res.status(200).send('ok'));
```

---

## 3) Build Images

Choose a single tag for both images (e.g., `dev` or `dev-<build-number>`).

```bash
# Backend
cd /path/to/learnerReportCS_backend
docker build -t docker.io/garapatimanoj/learner-be:dev .
docker push   docker.io/garapatimanoj/learner-be:dev

# Frontend
cd /path/to/learnerReportCS_frontend
docker build -t docker.io/garapatimanoj/learner-fe:dev .
docker push   docker.io/garapatimanoj/learner-fe:dev
```

> **Alternative (no push):** build into Minikube's Docker daemon
> ```bash
> eval $(minikube docker-env)
> docker build -t docker.io/garapatimanoj/learner-be:dev /path/to/learnerReportCS_backend
> docker build -t docker.io/garapatimanoj/learner-fe:dev /path/to/learnerReportCS_frontend
> # Use --set image.pullPolicy=IfNotPresent when installing (section 6)
> ```

---

## 4) Create Cluster, Namespace & Secret (Local Dev)

```bash
# Start Minikube
minikube start --cpus=4 --memory=8192

# Enable NGINX ingress
minikube addons enable ingress

# Create namespace
kubectl create namespace learner-dev

# Create Mongo secret (DO NOT COMMIT URI)
kubectl -n learner-dev create secret generic mongo-uri \
  --from-literal=MONGO_URI='<your-mongo-uri>'
```

> Replace `<your-mongo-uri>` with your real connection string.  
> Example format: `mongodb+srv://USERNAME:PASSWORD@cluster-host.mongodb.net/`

---

## 5) Helm Chart

**Location:** `/deployment/charts/learner-report`

- `Chart.yaml`
- `values.yaml`
- `templates/` (deployments, services, ingress, HPA, PDB, network policy)

**`/deployment/environments/values-dev.yaml` (example)**
```yaml
namespace: learner-dev
frontend: { tag: "dev" }
backend:  { tag: "dev" }
ingress:  { host: learner.local }
```

Install / upgrade:
```bash
cd /path/to/deployment
helm lint charts/learner-report

helm upgrade --install learner-report charts/learner-report \
  -n learner-dev --create-namespace \
  -f environments/values-dev.yaml
```

Check:
```bash
kubectl -n learner-dev get pods,svc,ingress
```

---

## 6) Accessing the App (Ingress)

Map the host to your Minikube IP:
```bash
IP=$(minikube ip); echo $IP
# If `.local` is blocked on macOS, use learner.test instead
echo "$IP  learner.local" | sudo tee -a /etc/hosts
open http://learner.local/
```

**Curl via NodePort (Host header):**
```bash
PORT=$(kubectl -n ingress-nginx get svc ingress-nginx-controller \
  -o=jsonpath='{.spec.ports[?(@.port==80)].nodePort}')

curl -I --resolve learner.local:${PORT}:${IP} http://learner.local:${PORT}/
curl -I --resolve learner.local:${PORT}:${IP} http://learner.local:${PORT}/api/health
```

**Bypass Ingress (service port-forward):**
```bash
# Frontend
kubectl -n learner-dev port-forward svc/learner-report-frontend 8080:80 &
open http://localhost:8080/

# Backend
kubectl -n learner-dev port-forward svc/learner-report-backend 3000:3000 &
curl -i http://localhost:3000/health
```

---

## 7) Jenkins CI/CD (Optional)

**`/deployment/Jenkinsfile`** (single-repo example):
```groovy
pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }
  environment {
    REGISTRY = 'docker.io'
    NAMESPACE = 'learner-dev'
    FRONTEND_IMAGE = 'garapatimanoj/learner-fe'
    BACKEND_IMAGE = 'garapatimanoj/learner-be'
    FE_TAG = "dev-${env.BUILD_NUMBER}"
    BE_TAG = "dev-${env.BUILD_NUMBER}"
    CHART_PATH = 'deployment/charts/learner-report'
  }
  stages {
    stage('Checkout'){ steps{ checkout scm } }

    stage('Build & Push FE'){
      steps{
        dir('frontend'){
          script{
            docker.withRegistry("https://${env.REGISTRY}", 'dockerhub-creds'){
              def fe = docker.build("${FRONTEND_IMAGE}:${FE_TAG}")
              fe.push()
            }
          }
        }
      }
    }

    stage('Build & Push BE'){
      steps{
        dir('backend'){
          script{
            docker.withRegistry("https://${env.REGISTRY}", 'dockerhub-creds'){
              def be = docker.build("${BACKEND_IMAGE}:${BE_TAG}")
              be.push()
            }
          }
        }
      }
    }

    stage('Deploy (Helm)'){
      steps{
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]){
          sh '''
            export KUBECONFIG="$KUBECONFIG_FILE"
            kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
            helm upgrade --install learner-report ${CHART_PATH} \
              -n ${NAMESPACE} -f deployment/environments/values-dev.yaml \
              --set frontend.image=${FRONTEND_IMAGE},frontend.tag=${FE_TAG} \
              --set backend.image=${BACKEND_IMAGE},backend.tag=${BE_TAG}
          '''
        }
      }
    }
  }
}
```

> Create `dockerhub-creds` (username/password) and `kubeconfig` credentials in Jenkins.  
> For multi-repo (separate FE/BE), add extra `checkout` steps or use a multibranch pipeline.

---

## 8) Operations

**Update image tags:**
```bash
helm upgrade learner-report deployment/charts/learner-report \
  -n learner-dev -f deployment/environments/values-dev.yaml \
  --set frontend.tag=dev-2 --set backend.tag=dev-2
```

**Rollout & status:**
```bash
kubectl -n learner-dev rollout status deploy/learner-report-frontend
kubectl -n learner-dev rollout status deploy/learner-report-backend
kubectl -n learner-dev get pods,svc,ingress
```

**Rollback:**
```bash
helm history learner-report -n learner-dev
helm rollback learner-report <REVISION> -n learner-dev
```

---

## 9) Troubleshooting (Real-World Issues)

### 9.1 `ImagePullBackOff` / `ErrImagePull`
- **Cause:** Image tag doesn’t exist on Docker Hub, or repo is private without imagePullSecret.
- **Fix:**
  ```bash
  # See which image is referenced
  kubectl -n learner-dev get deploy learner-report-frontend -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

  # Build & push that exact tag
  docker build -t docker.io/garapatimanoj/learner-fe:dev .
  docker push   docker.io/garapatimanoj/learner-fe:dev
  helm upgrade learner-report deployment/charts/learner-report -n learner-dev -f deployment/environments/values-dev.yaml
  ```
- **Local alternative:** `eval $(minikube docker-env)` then build images locally and set `--set image.pullPolicy=IfNotPresent`.

### 9.2 `container has runAsNonRoot and image will run as root`
- **Cause:** Pod securityContext enforces non-root but the image runs as root (common with NGINX on port 80).
- **Quick unblock (dev only):**
  ```bash
  kubectl -n learner-dev patch deploy/learner-report-frontend --type='json' -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/securityContext","value":{"runAsNonRoot":false}}
  ]'
  ```
- **Proper fix (recommended):** serve FE on **8080** as non-root, while Service exposes **80**:
  - In `nginx.conf`: `listen 8080;` and `pid /tmp/nginx.pid;`
  - In Dockerfile: `EXPOSE 8080` and keep `USER 10001`
  - In Helm values: `frontend.containerPort=8080` (Service still `port: 80` → `targetPort: 8080`)

### 9.3 `CreateContainerConfigError`
- **Cause:** Missing Secret or key (e.g., `mongo-uri` / `MONGO_URI`) in the **same namespace**.
- **Fix:**
  ```bash
  kubectl -n learner-dev create secret generic mongo-uri \
    --from-literal=MONGO_URI='<your-mongo-uri>' \
    --dry-run=client -o yaml | kubectl apply -f -

  kubectl -n learner-dev rollout restart deploy/learner-report-backend
  ```

### 9.4 Ingress timeouts / 404
- Ensure **ingress-nginx-controller** is Running:
  ```bash
  kubectl -n ingress-nginx get pods
  ```
- Ensure `/etc/hosts` maps host to `minikube ip`.
- Verify endpoints are **not empty**:
  ```bash
  kubectl -n learner-dev get endpoints
  ```

---

## 10) Production Hardening Checklist

- Enable **TLS** on Ingress and use a real domain.
- Use **HPA** (CPU/memory or custom metrics) and tune **requests/limits**.
- Add **PodDisruptionBudget** to keep at least one pod available.
- Use **image digests** (`@sha256:...`) and a private registry + `imagePullSecrets`.
- Ensure **readiness** and **liveness** probes are fast and reliable.
- Centralized logging (e.g., EFK/ELK) and monitoring (Prometheus/Grafana).
- Separate **dev/prod values** (`values-prod.yaml`) and namespaces.
- Database credentials: manage via **Secrets** or an external secret manager (Azure Key Vault, etc.).

---

## 11) Quick Commands Reference

```bash
# Build & push (FE/BE)
docker build -t docker.io/garapatimanoj/learner-fe:dev ./frontend && docker push docker.io/garapatimanoj/learner-fe:dev
docker build -t docker.io/garapatimanoj/learner-be:dev ./backend  && docker push docker.io/garapatimanoj/learner-be:dev

# Minikube
minikube start --cpus=4 --memory=8192
minikube addons enable ingress

# Namespace & Secret
kubectl create ns learner-dev
kubectl -n learner-dev create secret generic mongo-uri --from-literal=MONGO_URI='<your-mongo-uri>'

# Helm install/upgrade
helm upgrade --install learner-report deployment/charts/learner-report \
  -n learner-dev -f deployment/environments/values-dev.yaml

# Access
IP=$(minikube ip); echo "$IP  learner.local" | sudo tee -a /etc/hosts
open http://learner.local/

# Status
kubectl -n learner-dev get pods,svc,ingress
kubectl -n learner-dev get endpoints
```

---

**That’s it!** You now have a repeatable, production-like workflow to build, deploy, and operate your MERN app on Kubernetes with Helm and Jenkins. Keep secrets out of Git, prefer non-root containers, and use separate values for dev/prod.

