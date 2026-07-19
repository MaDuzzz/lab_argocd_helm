# Deployment vs StatefulSet

Hướng dẫn lựa chọn giữa Deployment và StatefulSet, và cách sử dụng.

## 📊 So sánh nhanh

| Đặc tính         | Deployment                | StatefulSet                      |
| ---------------- | ------------------------- | -------------------------------- |
| **Pod Identity** | Random name               | Stable (my-app-0, my-app-1, ...) |
| **Hostname**     | Random                    | Stable (DNS resolvable)          |
| **Scaling**      | Parallel                  | Sequential (ordered)             |
| **Storage**      | Chung hoặc riêng (manual) | Persistent cho mỗi pod (auto)    |
| **Restart**      | Pod bất kỳ                | Giữ identity                     |
| **Use Cases**    | Web app, API, stateless   | DB, cache, queue, stateful       |

## 🎯 Khi nào dùng Deployment?

✅ **Stateless applications:**

- Web servers (Nginx, Apache)
- API servers (Node.js, Python, Go)
- Microservices
- Load-balanced apps

**Example:**

```bash
deployment:
  enabled: true
  replicas: 3  # Bất kỳ pod nào cũng được

helm install my-app ./my-app-chart \
  --set deployment.enabled=true \
  --set deployment.replicas=3
```

## 🗄️ Khi nào dùng StatefulSet?

✅ **Stateful applications:**

- **Databases**: PostgreSQL, MySQL, MongoDB
- **Cache**: Redis, Memcached
- **Message Queues**: RabbitMQ, Kafka
- **Search Engines**: Elasticsearch
- **Consensus Systems**: etcd, ZooKeeper
- **Distributed Systems**: cần stable identity

**Example:**

```bash
statefulSet:
  enabled: true
  replicas: 3  # Pod sẽ là: app-0, app-1, app-2

helm install my-db ./my-app-chart \
  --set statefulSet.enabled=true \
  --set deployment.enabled=false \
  -f values-statefulset-example.yaml
```

## 🔧 Setup StatefulSet (Step-by-step)

### Step 1: Disable Deployment, Enable StatefulSet

```yaml
# values.yaml hoặc values-statefulset-example.yaml
deployment:
  enabled: false

statefulSet:
  enabled: true
  replicas: 3
  serviceName: my-app-stateful
  podManagementPolicy: OrderedReady # Chờ pod ready mới tạo pod kế tiếp
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: fast-ssd
```

### Step 2: Configure persistent storage via volumeClaimTemplates

**Tự động tạo PVC cho mỗi pod:**

- Pod 0: `my-app-statefulset-data-0`
- Pod 1: `my-app-statefulset-data-1`
- Pod 2: `my-app-statefulset-data-2`

Mỗi PVC có storage riêng, khi pod die, PVC vẫn tồn tại.

### Step 3: Mount volume trong container

```yaml
pod:
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /data # App dùng /data để lưu state
```

### Step 4: Enable PDB (Pod Disruption Budget)

Quan trọng cho StatefulSet - bảo vệ ít nhất 1 pod luôn running:

```yaml
pdb:
  enabled: true
  minAvailable: 1 # Luôn có ít nhất 1 pod available
```

### Step 5: Set Pod Anti-Affinity

Spread pods across nodes - tránh single node failure:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - my-app-chart
          topologyKey: kubernetes.io/hostname
```

## 🚀 Deploy

```bash
# Deploy StatefulSet
helm install my-db ./my-app-chart \
  -f values-statefulset-example.yaml \
  --namespace production \
  --create-namespace

# Verify pods created in order
kubectl get pods -n production -w
# Output:
# my-db-0      0/1   Pending   0    5s
# (waiting for my-db-0 ready...)
# my-db-0      1/1   Running   0    10s
# my-db-1      0/1   Pending   0    12s
# (waiting for my-db-1 ready...)
# my-db-1      1/1   Running   0    20s
# my-db-2      0/1   Pending   0    22s
# ...
```

## 📝 Pod Naming & DNS

StatefulSet pods có stable hostname:

```bash
# Pod name format: release-name-index
my-db-0, my-db-1, my-db-2

# DNS name (resolvable từ cluster):
my-db-0.my-app-stateful.production.svc.cluster.local
my-db-1.my-app-stateful.production.svc.cluster.local
my-db-2.my-app-stateful.production.svc.cluster.local

# Shorthand trong same namespace:
my-db-0.my-app-stateful
my-db-1.my-app-stateful
my-db-2.my-app-stateful

# Headless service (no load balancing):
my-app-stateful.production.svc.cluster.local
# -> resolves to all pod IPs
```

### Automatic Headless Service Creation

Service template (`service.yaml`) tự động tạo **Headless Service** (clusterIP: None) khi StatefulSet enabled:

```yaml
# StatefulSet config - service tự động tạo
statefulSet:
  enabled: true
  serviceName: my-app-stateful
  headlessService:
    enabled: true # Tự động tạo headless service
```

Không cần tạo service riêng - `service.yaml` template xử lý cả Deployment service lẫn StatefulSet headless service.

## 💾 Persistent Volume Claims

StatefulSet tự động tạo PVC:

```bash
# View PVCs
kubectl get pvc -n production
# NAME                        STATUS   VOLUME                 CAPACITY
# data-my-db-0               Bound    pvc-xxx                20Gi
# data-my-db-1               Bound    pvc-yyy                20Gi
# data-my-db-2               Bound    pvc-zzz                20Gi

# View PVs
kubectl get pv
```

**Important:** Khi delete StatefulSet, PVCs KHÔNG bị delete (safety feature).  
Cần manually delete nếu muốn:

```bash
kubectl delete pvc -l app.kubernetes.io/name=my-app-chart -n production
```

## 🔄 Rolling Updates

StatefulSet update theo thứ tự (từ cuối):

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 0 # Update từ pod index 0 trở lên
    # partition: 2 -> chỉ update my-db-2, my-db-1, my-db-0 không update
```

### Canary deployment với partition:

```bash
# Step 1: Set partition=2 (chỉ update my-db-2)
helm upgrade my-db ./my-app-chart \
  --set statefulSet.updateStrategy.rollingUpdate.partition=2

# Step 2: Kiểm tra my-db-2 hoạt động ok
kubectl logs my-db-2

# Step 3: Set partition=1 (update my-db-1 và my-db-2)
helm upgrade my-db ./my-app-chart \
  --set statefulSet.updateStrategy.rollingUpdate.partition=1

# Step 4: Set partition=0 (update tất cả)
helm upgrade my-db ./my-app-chart \
  --set statefulSet.updateStrategy.rollingUpdate.partition=0
```

## 🐛 Debugging

```bash
# Check StatefulSet status
kubectl get statefulset -n production
kubectl describe statefulset my-db -n production

# Check pods (thứ tự creation)
kubectl get pods -n production -L ordinal

# Check persistent volumes
kubectl get pvc,pv -n production

# View pod logs
kubectl logs my-db-0 -n production
kubectl logs my-db-1 -n production

# Connect to pod
kubectl exec -it my-db-0 -n production -- /bin/bash

# Check pod DNS (từ inside pod)
nslookup my-db-0.my-app-stateful
nslookup my-app-stateful  # <- all pods
```

## 📚 Example Use Cases

### PostgreSQL Cluster

```yaml
statefulSet:
  enabled: true
  replicas: 3
  serviceName: postgres
  podManagementPolicy: OrderedReady
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi # Large storage
        storageClassName: fast-ssd
```

### RabbitMQ Cluster

```yaml
statefulSet:
  enabled: true
  replicas: 3
  serviceName: rabbitmq
  volumeClaimTemplates:
    - metadata:
        name: rabbitmq-data
      spec:
        resources:
          requests:
            storage: 50Gi
```

## ⚠️ Common Pitfalls

❌ **DON'T:**

1. Auto-scale StatefulSet (HPA) - tricky với stateful apps
2. Delete PVC khi delete StatefulSet (data loss!)
3. Change `serviceName` - breaks pod DNS
4. Use Deployment cho stateful apps - data loss on pod restart

✅ **DO:**

1. Enable PDB - protect pods from disruption
2. Use Pod Anti-Affinity - spread across nodes
3. Set proper `partition` cho controlled rollout
4. Monitor PVC usage - prevent out-of-disk

## 🔗 References

- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [StatefulSet Basics](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
