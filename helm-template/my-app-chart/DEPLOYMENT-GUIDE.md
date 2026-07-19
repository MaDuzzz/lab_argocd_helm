# Deployment Guide: Helm Chart + Nexus + ArgoCD

Hướng dẫn chi tiết cách push Helm chart & Docker image lên Nexus, rồi deploy qua ArgoCD.

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────┐
│ Developer Laptop                        │
│  - Build Docker image                  │
│  - Package Helm chart                  │
└────────────────┬────────────────────────┘
                 │ docker push & helm push
                 ▼
┌─────────────────────────────────────────┐
│ Nexus Registry                          │
│  - Docker Registry (images)             │
│  - Helm Repository (charts)             │
└────────────────┬────────────────────────┘
                 │ ArgoCD monitors & pulls
                 ▼
┌─────────────────────────────────────────┐
│ Kubernetes Cluster                      │
│  - Pre-created docker-registry secret   │
│  - ArgoCD pulls Helm chart              │
│  - K8s pulls Docker image via secret    │
│  - Pods running                         │
└─────────────────────────────────────────┘
```

## 📋 Prerequisites

```bash
# Local machine cần có:
- Docker
- Helm 3+
- kubectl
- Nexus instance running (with Docker Registry & Helm Repository enabled)
```

## 🚀 Step 1: Build & Push Docker Image lên Nexus

### 1.1 Chuẩn bị Dockerfile

```dockerfile
# Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1
CMD ["node", "index.js"]
```

### 1.2 Build image

```bash
docker build -t nexus.example.com:8082/my-company/my-app:v1.0.0 .
```

### 1.3 Login vào Nexus Docker Registry

```bash
docker login nexus.example.com:8082 \
  --username deploy-user \
  --password <password>
```

### 1.4 Push image lên Nexus

```bash
docker push nexus.example.com:8082/my-company/my-app:v1.0.0
```

**Verify:**

```bash
curl -u deploy-user:<password> \
  http://nexus.example.com:8081/service/rest/v1/search/assets?repository=docker-registry&name=my-app
```

## 📦 Step 2: Package & Push Helm Chart lên Nexus

### 2.1 Update `Chart.yaml` & `values.yaml`

```yaml
# my-app-chart/Chart.yaml
apiVersion: v2
name: my-app-chart
version: 1.0.0
appVersion: v1.0.0
```

```yaml
# my-app-chart/values.yaml
image:
  repository: nexus.example.com:8082/my-company/my-app
  tag: v1.0.0
imagePullSecrets:
  - name: nexus-registry-secret # Pre-created secret, not credentials!
```

### 2.2 Validate Helm chart

```bash
helm lint ./my-app-chart/
helm template my-app ./my-app-chart/ --debug
```

### 2.3 Package Helm chart

```bash
helm package ./my-app-chart/
# Output: my-app-chart-1.0.0.tgz
```

### 2.4 Push lên Nexus Helm Repository

```bash
# Option 1: Dùng curl
curl -u deploy-user:<password> \
  --upload-file my-app-chart-1.0.0.tgz \
  http://nexus.example.com:8081/repository/helm-repo/
```

```bash
# Option 2: Dùng helm-push plugin (recommended)
helm plugin install https://github.com/chartmuseum/helm-push.git

helm cm-push ./my-app-chart/ \
  --username=deploy-user \
  --password=<password> \
  nexus-helm-repo
```

**Add Nexus repo locally:**

```bash
helm repo add nexus-helm \
  http://nexus.example.com:8081/repository/helm-repo/ \
  --username deploy-user \
  --password <password>

helm repo update
helm search repo nexus-helm/my-app-chart
```

## 🔐 Step 3: Setup Kubernetes Image Pull Secret (PRE-REQUISITE)

### ⚠️ CRITICAL: Create secret BEFORE deploying app

**Credentials should NEVER be hardcoded in values.yaml or committed to Git!**

### 3.1 Create docker-registry secret trong K8s

```bash
# Create the secret FIRST in your namespace
kubectl create secret docker-registry nexus-registry-secret \
  --docker-server=nexus.example.com:8082 \
  --docker-username=deploy-user \
  --docker-password=<password> \
  --docker-email=devops@example.com \
  --namespace=production

# Verify the secret exists
kubectl get secret nexus-registry-secret -n production -o yaml
```

### 3.2 Deploy Helm chart referencing the secret

```bash
# Helm chart references the pre-created secret by name
# values-example.yaml shows:
# imagePullSecrets:
#   - name: nexus-registry-secret

helm install my-app ./my-app-chart \
  --namespace production \
  --create-namespace
```

Or with --set flag:

```bash
helm install my-app ./my-app-chart \
  --namespace production \
  --set imagePullSecrets[0].name=nexus-registry-secret
```

### 3.3 (Production) Use Sealed Secrets with ArgoCD

For GitOps workflow - encrypt secrets before committing:

```bash
# Install sealed-secrets operator
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml -n kube-system

# Step 1: Create temporary unsealed secret
kubectl create secret docker-registry nexus-registry-secret \
  --docker-server=nexus.example.com:8082 \
  --docker-username=deploy-user \
  --docker-password=<password> \
  --docker-email=devops@example.com \
  --namespace=production \
  --dry-run=client -o yaml > secret.yaml

# Step 2: Seal the secret (encrypted version)
kubeseal -f secret.yaml -w sealed-secret.yaml

# Step 3: Safely commit to Git
rm secret.yaml  # Remove the unsealed version!
git add sealed-secret.yaml
git commit -m "Add sealed nexus registry secret"
git push
```

Sealed secrets controller in K8s cluster sẽ tự động unseal và create the secret.

## 🎯 Step 4: Setup ArgoCD Application

### 4.1 Add Nexus Helm Repository vào ArgoCD

```bash
argocd repo add nexus-helm \
  --type helm \
  --name nexus-helm \
  --repo http://nexus.example.com:8081/repository/helm-repo/ \
  --username deploy-user \
  --password <password>

# Verify
argocd repo list
```

### 4.2 Create ArgoCD Application

```yaml
# argocd/my-app-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  # Source từ Nexus Helm Repository
  source:
    repoURL: http://nexus.example.com:8081/repository/helm-repo/
    targetRevision: "1.0.0"
    chart: my-app-chart

    helm:
      values: |
        namespace: production
        image:
          repository: nexus.example.com:8082/my-company/my-app
          tag: v1.0.0
        imagePullSecrets:
          - name: nexus-registry-secret  # Must exist in cluster!
        deployment:
          replicas: 2
        service:
          enabled: true
        ingress:
          enabled: true
          hosts:
            - host: myapp.example.com
              paths:
                - path: /
                  pathType: Prefix
        hpa:
          enabled: true
          minReplicas: 2
          maxReplicas: 5
        pdb:
          enabled: true

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 4.3 Create the Application

```bash
# Pre-create the secret first!
kubectl create secret docker-registry nexus-registry-secret \
  --docker-server=nexus.example.com:8082 \
  --docker-username=deploy-user \
  --docker-password=<password> \
  --docker-email=devops@example.com \
  --namespace=production

# Then apply ArgoCD Application
kubectl apply -f argocd/my-app-application.yaml

# Monitor
argocd app get my-app
argocd app sync my-app
argocd app logs my-app
```

## 🔄 Step 5: CI/CD Pipeline Integration

### 5.1 GitLab CI example

```yaml
# .gitlab-ci.yml
stages:
  - build
  - push
  - deploy

# Build & Push Docker Image
build_image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t nexus.example.com:8082/my-company/my-app:$CI_COMMIT_SHA .
    - docker tag nexus.example.com:8082/my-company/my-app:$CI_COMMIT_SHA nexus.example.com:8082/my-company/my-app:latest
    - docker login -u $NEXUS_USER -p $NEXUS_PASSWORD nexus.example.com:8082
    - docker push nexus.example.com:8082/my-company/my-app:$CI_COMMIT_SHA
    - docker push nexus.example.com:8082/my-company/my-app:latest

# Update Helm chart values
update_chart:
  stage: build
  image: alpine:latest
  script:
    - apk add yq
    - yq eval ".image.tag = \"$CI_COMMIT_SHA\"" -i helm-template/my-app-chart/values.yaml

# Push Helm Chart
push_chart:
  stage: push
  image: alpine/helm:latest
  script:
    - helm repo add nexus-helm http://nexus.example.com:8081/repository/helm-repo/ --username $NEXUS_USER --password $NEXUS_PASSWORD
    - helm package helm-template/my-app-chart/
    - curl -u $NEXUS_USER:$NEXUS_PASSWORD --upload-file my-app-chart-*.tgz http://nexus.example.com:8081/repository/helm-repo/

# Trigger ArgoCD sync
trigger_argocd:
  stage: deploy
  image: argoproj/argocd:latest
  script:
    - argocd login $ARGOCD_SERVER --username $ARGOCD_USERNAME --password $ARGOCD_PASSWORD
    - argocd app sync my-app
    - argocd app wait my-app --sync
```

## ✅ Verification Steps

```bash
# 1. Check secret exists
kubectl get secret nexus-registry-secret -n production

# 2. Check pod running
kubectl get pods -n production
kubectl logs -f <pod-name> -n production

# 3. Check image pulled successfully
kubectl describe pod <pod-name> -n production | grep -A 5 "Pulling image"

# 4. Check service
kubectl get svc -n production
curl http://<service-ip>:80/health

# 5. Check ArgoCD status
argocd app get my-app
argocd app status my-app
```

## 🐛 Troubleshooting

### Image pull failed

```bash
# Check if secret exists
kubectl get secret nexus-registry-secret -n production

# Check pod events
kubectl describe pod <pod-name> -n production

# Check if pod spec references the secret
kubectl get pod <pod-name> -o yaml | grep -A 3 "imagePullSecrets"

# Test credentials manually
docker login -u deploy-user -p <password> nexus.example.com:8082
docker pull nexus.example.com:8082/my-company/my-app:v1.0.0
```

### ArgoCD sync failed

```bash
# Check ArgoCD logs
kubectl logs -f deployment/argocd-application-controller -n argocd

# Check if Helm repo accessible
helm repo update
helm search repo nexus-helm

# Check Application status
argocd app get my-app --refresh
argocd app sync my-app --force
```

### Pod not ready

```bash
# Check health probe
kubectl get pod <pod-name> -o yaml | grep -A 20 "readinessProbe\|livenessProbe"

# Test health endpoint manually
kubectl port-forward <pod-name> 8080:8080
curl http://localhost:8080/health
curl http://localhost:8080/ready
```

## 📚 Key Takeaways

✅ **NEVER hardcode credentials** in values.yaml  
✅ **Create secrets separately** before deploying Helm chart  
✅ **Reference secrets by name** in imagePullSecrets  
✅ **Use Sealed Secrets** for GitOps with ArgoCD  
✅ **Manage credentials** via K8s secrets or external secret managers

## References

- [Kubernetes Image Pull Secrets](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best-practices/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Nexus Docker Registry](https://help.sonatype.com/repomanager3/formats/docker-registry)
