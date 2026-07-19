# my-app-chart - Standard Helm Template

Standard production-ready Helm chart template cho microservices deployment trên Kubernetes.

## 📋 Tính năng chính

- ✅ **Deployment** - cho stateless apps
- ✅ **StatefulSet** - cho stateful apps (DB, cache, etc) với persistent storage
- ✅ **Service** - hỗ trợ ClusterIP, NodePort, LoadBalancer (cho Deployment)
- ✅ **Headless Service** - tự động tạo cho StatefulSet DNS resolution
- ✅ **Ingress** - nginx/traefik compatible
- ✅ **ConfigMap & Secret** - support key-value và file config
- ✅ **PVC** - Persistent Volume cho data storage
- ✅ **HPA** - Auto-scaling with custom behavior
- ✅ **PDB** - Pod Disruption Budget để protect pods
- ✅ **Pod Anti-Affinity** - spread pods across nodes/zones
- ✅ **Image Pull Secrets** - hỗ trợ private registries (Nexus, ECR, etc.)
- ✅ **graceful Shutdown** - preStop hook

## 🚀 Cách sử dụng

### 1. Cơ bản - Deploy với default values

```bash
helm install my-app ./my-app-chart \
  --namespace default \
  --create-namespace
```

### 2. Specify image từ Nexus

```bash
helm install my-app ./my-app-chart \
  --set image.repository=nexus.example.com:8082/my-app \
  --set image.tag=v1.0.0
```

### 3. Enable Service & Ingress

```bash
helm install my-app ./my-app-chart \
  --set service.enabled=true \
  --set service.type=ClusterIP \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=myapp.example.com
```

### 4. Pull image từ Private Registry (Nexus)

#### Best Practice: Pre-create docker-registry secret

```bash
# Step 1: Create secret FIRST (BEFORE deploying Helm chart)
kubectl create secret docker-registry nexus-registry-secret \
  --docker-server=nexus.example.com:8082 \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=devops@example.com \
  -n production

# Step 2: Deploy with reference to secret
helm install my-app ./my-app-chart \
  --namespace production \
  --set imagePullSecrets[0].name=nexus-registry-secret
```

#### values.yaml configuration

```yaml
# values.yaml
image:
  repository: nexus.example.com:8082/my-company/my-app
  tag: v1.0.0

# Reference to pre-created secret (NOT credentials!)
imagePullSecrets:
  - name: nexus-registry-secret
```

#### For ArgoCD (Sealed Secrets)

```bash
# Create and seal the secret
kubectl create secret docker-registry nexus-registry-secret \
  --docker-server=nexus.example.com:8082 \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=devops@example.com \
  --namespace=production \
  --dry-run=client -o yaml | kubeseal -o yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to Git (safe!)
git add sealed-secret.yaml
```

### 5. Enable HPA (Auto-scaling)

```bash
helm install my-app ./my-app-chart \
  --set hpa.enabled=true \
  --set hpa.minReplicas=2 \
  --set hpa.maxReplicas=5 \
  --set hpa.targetCPUUtilizationPercentage=70
```

### 6. Enable PDB (Pod Disruption Budget)

```bash
helm install my-app ./my-app-chart \
  --set pdb.enabled=true \
  --set pdb.minAvailable=1
```

### 7. ConfigMap với file config (nginx)

```yaml
# values.yaml
configMap:
  enabled: true
  name: app-config
  files:
    nginx.conf: |
      server {
        listen 80;
        server_name _;
        location / {
          proxy_pass http://localhost:8080;
        }
      }
```

Sau đó mount vào deployment:

```yaml
# values.yaml
pod:
  containers:
    - name: app
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: nginx-config
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: default.conf
```

### 8. StatefulSet cho Stateful Apps (DB, Cache, etc)

Dùng StatefulSet khi ứng dụng cần:

- Persistent identity (stable pod name: db-0, db-1, db-2)
- Persistent storage riêng cho mỗi pod
- Sequential deployment/scaling

```bash
# Deploy as StatefulSet (không dùng Deployment)
helm install my-db ./my-app-chart \
  --set deployment.enabled=false \
  --set statefulSet.enabled=true \
  --set statefulSet.replicas=3 \
  -f values-statefulset-example.yaml
```

Pod sẽ được tạo theo thứ tự:

```
my-db-0  -> /data volume từ PVC (20Gi)
my-db-1  -> /data volume từ PVC (20Gi)
my-db-2  -> /data volume từ PVC (20Gi)
```

DNS names:

```
my-db-0.my-app-stateful.production.svc.cluster.local
my-db-1.my-app-stateful.production.svc.cluster.local
my-db-2.my-app-stateful.production.svc.cluster.local
```

**Use cases:**

- PostgreSQL, MySQL clusters
- MongoDB replica sets
- Redis clusters
- Elasticsearch nodes
- RabbitMQ clusters

Chi tiết tại [STATEFULSET-GUIDE.md](STATEFULSET-GUIDE.md)

## 📦 ArgoCD Integration

Để deploy qua ArgoCD từ Nexus Helm Repository:

### 1. Config Nexus Helm Repository trong ArgoCD

```bash
argocd repo add nexus-helm \
  --type helm \
  --name nexus-helm \
  --repo http://nexus.example.com:8081/repository/helm-repo/ \
  --username <username> \
  --password <password>
```

### 2. Create ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://nexus.example.com:8081/repository/helm-repo/
    chart: my-app-chart
    targetRevision: 1.0.0
    helm:
      values: |
        image:
          repository: nexus.example.com:8082/my-app
          tag: v1.0.0
        imagePullSecrets:
          - name: nexus-registry-secret
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 📝 Values Configuration

| Key                                | Description                            | Default              |
| ---------------------------------- | -------------------------------------- | -------------------- |
| `image.repository`                 | Docker image repository                | `my-registry/my-app` |
| `image.tag`                        | Image tag                              | `latest`             |
| `image.pullPolicy`                 | Pull policy                            | `IfNotPresent`       |
| `imagePullSecrets`                 | List of image pull secrets             | `[]`                 |
| `deployment.enabled`               | Enable Deployment                      | `true`               |
| `deployment.replicas`              | Number of replicas                     | `1`                  |
| `statefulSet.enabled`              | Enable StatefulSet                     | `false`              |
| `statefulSet.replicas`             | Number of StatefulSet replicas         | `1`                  |
| `statefulSet.serviceName`          | Headless service name                  | `my-app`             |
| `statefulSet.volumeClaimTemplates` | Persistent storage per pod             | See values.yaml      |
| `service.enabled`                  | Create service                         | `true`               |
| `service.type`                     | Service type (ClusterIP/NodePort)      | `ClusterIP`          |
| `ingress.enabled`                  | Create ingress                         | `false`              |
| `configMap.enabled`                | Create configmap                       | `false`              |
| `secret.enabled`                   | Create secret                          | `false`              |
| `pvc.enabled`                      | Create PVC                             | `false`              |
| `hpa.enabled`                      | Enable auto-scaling                    | `false`              |
| `pdb.enabled`                      | Enable PDB (important for StatefulSet) | `false`              |

## 🔐 Best Practices

1. **Image Pull Secrets** - CRITICAL:
   - **NEVER hardcode credentials** in values.yaml
   - **CREATE secret BEFORE** deploying Helm chart
   - Reference secret by name only: `imagePullSecrets: [name: nexus-registry-secret]`
   - Use Sealed Secrets or External Secrets Operator for GitOps
   - Example:

     ```bash
     # Create first
     kubectl create secret docker-registry nexus-registry-secret \
       --docker-server=nexus.example.com:8082 \
       --docker-username=deploy-user \
       --docker-password=<password>

     # Then deploy Helm chart
     helm install my-app ./my-app-chart \
       --set imagePullSecrets[0].name=nexus-registry-secret
     ```

2. **Resource Limits**:
   - Luôn set `requests` và `limits` để tránh pod bị evict
   - Adjust theo actual app requirements

3. **Health Checks**:
   - Adjust probe parameters theo app startup time
   - `startupProbe` cho app khởi động lâu
   - `readinessProbe` để loại bỏ pod từ load balancer
   - `livenessProbe` để restart unhealthy pod

4. **Anti-Affinity**:
   - Enable `podAntiAffinity` để tránh single point of failure
   - `topologySpreadConstraints` cho even distribution

5. **Graceful Shutdown**:
   - `preStop` hook để complete in-flight requests
   - `terminationGracePeriodSeconds` để đủ time

6. **Never in values.yaml**:
   - ❌ Passwords, API keys, tokens
   - ❌ Docker registry credentials
   - ❌ Database passwords
   - ✅ Use Kubernetes Secrets instead

## 🛠️ Troubleshooting

### Image pull failed

```bash
# Check image pull secret
kubectl get secret nexus-registry-secret -o yaml

# Check pod events
kubectl describe pod <pod-name>

# Check image availability
docker pull nexus.example.com:8082/my-app:v1.0.0
```

### HPA not scaling

```bash
# Check HPA status
kubectl get hpa
kubectl describe hpa my-app

# Check metrics server
kubectl get deployment metrics-server -n kube-system
```

### Pod not starting

```bash
# Check pod logs
kubectl logs <pod-name>

# Check resource requests
kubectl describe pod <pod-name> | grep -A 5 "Requests"

# Check node capacity
kubectl describe nodes
```

## 📚 References

- [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Helm Charts](https://helm.sh/docs/topics/charts/)
- [Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
