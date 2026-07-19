# Kubernetes 进阶技术文档

## 1. 概述 - Kubernetes是什么，为什么需要K8s
Kubernetes（K8s）是开源的容器编排平台，用于自动化部署、扩展和管理容器化应用。其核心价值在于将基础设施抽象为可编程的资源池，提供声明式API、自我修复、服务发现、滚动更新和弹性伸缩能力。在微服务架构与云原生时代，K8s解决了传统虚拟机管理中的资源碎片化、部署不一致、运维复杂等痛点，成为现代应用交付的标准底座。

```yaml
# 命名空间与资源配额示例（集群级资源治理起点）
apiVersion: v1
kind: Namespace
metadata:
  name: production-app
  labels:
    env: prod
    team: backend
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production-app
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "5"
    persistentvolumeclaims: "10"
```

---

## 2. 核心资源对象 - Pod、Service、Deployment、StatefulSet、DaemonSet、CronJob
核心资源对象构成K8s的声明式模型基础：
- **Pod**：最小调度单元，封装容器与共享网络/存储。
- **Deployment**：无状态应用控制器，支持滚动更新与回滚。
- **StatefulSet**：有状态应用控制器，保证稳定网络标识与有序部署。
- **DaemonSet**：每节点运行单一Pod副本（如日志采集、监控）。
- **CronJob**：基于时间调度的任务控制器。
- **Service**：稳定网络入口，通过Label Selector动态绑定Pod IP。

```yaml
# StatefulSet + Headless Service + PVC 典型组合
apiVersion: v1
kind: Service
metadata:
  name: postgres-hl
  namespace: db
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
  namespace: db
spec:
  serviceName: "postgres-hl"
  replicas: 3
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
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## 3. 高级调度 - Node Affinity、Pod Affinity/Anti-Affinity、Taint/Toleration
高级调度机制用于精确控制Pod落地位置，满足性能隔离、低延迟通信或硬件依赖需求：
- **Node Affinity**：基于节点标签选择目标节点。
- **Pod Affinity/Anti-Affinity**：基于同节点/同拓扑域内其他Pod的标签进行亲和或反亲和调度。
- **Taint/Toleration**：节点打污点排斥非容忍Pod，Pod设置容忍度接受调度。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-inference
  namespace: ml
spec:
  replicas: 2
  selector:
    matchLabels:
      app: inference
  template:
    metadata:
      labels:
        app: inference
    spec:
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["cn-east-1a", "cn-east-1b"]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: ["inference"]
                topologyKey: "kubernetes.io/hostname"
      containers:
        - name: model-server
          image: triton-inference-server:2.30
          resources:
            limits:
              nvidia.com/gpu: 1
```

---

## 4. 网络策略 - NetworkPolicy、Service Mesh概述
**NetworkPolicy** 基于eBPF或iptables实现L3/L4层流量控制，默认允许所有流量，需显式声明规则。  
**Service Mesh**（如Istio/Linkerd）将服务通信逻辑下沉至Sidecar代理，实现可观测性、熔断、灰度发布与mTLS。K8s原生网络策略适合基础隔离，Mesh适合复杂微服务治理。

```yaml
# 严格网络隔离策略：仅允许特定Pod访问数据库端口
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-db-access
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: api-gateway
        - namespaceSelector:
            matchLabels:
              env: prod
      ports:
        - protocol: TCP
          port: 5432
```

---

## 5. 存储管理 - PV、PVC、StorageClass、CSI驱动
K8s存储抽象解耦了底层存储实现：
- **StorageClass**：动态供给卷，定义provisioner与回收策略。
- **PV/PVC**：静态/动态卷的生命周期与绑定机制。
- **CSI**：容器存储接口标准，支持云厂商、分布式存储、本地SSD等插件。

```yaml
# 动态存储供给示例（以Ceph RBD为例）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com  # 替换为实际CSI驱动
parameters:
  type: gp3
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: web
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

---

## 6. HPA自动扩缩 - 资源指标、自定义指标、VPA
- **HPA**：基于CPU/Memory或Prometheus自定义指标（如QPS、队列长度）水平伸缩。
- **VPA**：垂直调整单Pod资源请求/限制，通常与HPA配合使用。
- 生产环境建议启用`metrics-server`或Prometheus Adapter，并设置合理`targetAverageUtilization`。

```yaml
# HPA 基于自定义指标（Prometheus Adapter 格式）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 7. RBAC权限控制 - Role、ClusterRole、RoleBinding
RBAC遵循最小权限原则。K8s权限模型由 `User/Group/ServiceAccount` → `Role/ClusterRole` → `RoleBinding/ClusterRoleBinding` 组成。生产环境应严格隔离命名空间权限，避免滥用`cluster-admin`。

```yaml
# 命名空间级只读角色与绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: log-reader
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-log-access
  namespace: monitoring
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: log-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 8. Helm包管理 - Chart结构、Values、模板语法
Helm是K8s的包管理器，采用Client-Server架构（Tiller已废弃）。Chart目录包含`Chart.yaml`、`values.yaml`、`templates/`。模板基于Go `text/template`，支持管道、条件循环与内置函数。

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp/backend
  tag: "v2.1.0"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

```go-template
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## 9. Operator模式 - CRD、Operator SDK、工作原理
Operator将运维知识编码为K8s控制器，通过`CRD`定义业务模型，Controller监听事件并收敛至期望状态。适用于数据库、中间件、AI训练等复杂有状态系统。

```yaml
# CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: redisclusters.example.com
spec:
  group: example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                storageSize:
                  type: string
  scope: Namespaced
  names:
    plural: redisclusters
    singular: rediscluster
    kind: RedisCluster
---
# 实例资源
apiVersion: example.com/v1alpha1
kind: RedisCluster
metadata:
  name: prod-redis
  namespace: cache
spec:
  replicas: 3
  storageSize: 20Gi
```

---

## 10. 故障排查 - kubectl调试命令、常见问题排查
生产调试依赖精准工具链：`kubectl debug`（Ephemeral Containers）、`kubectl logs -f --previous`、`kubectl describe`、`kubectl proxy`（访问APIServer/Dashboard）。常见陷阱：镜像拉取失败、节点压力驱逐、DNS解析延迟、资源竞争。

```yaml
# 临时调试容器（不修改原Pod规格）
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: prod
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
  ephemeralContainers:
    - name: netshoot
      image: nicolaka/netshoot:latest
      command: ["sleep", "3600"]
      securityContext:
        capabilities:
          add: ["NET_ADMIN", "NET_RAW"]
      terminationMessagePath: /dev/termination-log
```

---

## 11. 最佳实践 - 生产环境部署建议
- 始终设置`requests`与`limits`，避免资源争抢。
- 使用`liveness/readiness/startup`探针，合理配置`initialDelaySeconds`与`failureThreshold`。
- 启用`PodDisruptionBudget`保障高可用。
- 敏感数据使用`Secret`或外部密钥管理（Vault/KMS），避免明文。
- 镜像打固定Tag，禁用`:latest`，启用ImagePullPolicy。
- 节点隔离工作负载，划分`system-critical`与`tenant`命名空间。

```yaml
# 生产级加固Deployment + PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: prod
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
        - name: web
          image: registry.internal/web:v1.4.2
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
            limits: { cpu: "500m", memory: "512Mi" }
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            failureThreshold: 3
          startupProbe:
            httpGet: { path: /start, port: 8080 }
            failureThreshold: 30
            periodSeconds: 5
```

---

## 12. 常见问题FAQ

**Q1：HPA扩缩频繁震荡怎么办？**  
A：调整`behavior.scaleUp.scaleDown.stabilizationWindowSeconds`，启用`metrics`聚合窗口，避免瞬时峰值触发伸缩。结合VPA预热资源请求。

**Q2：Pod一直处于`Pending`状态？**  
A：检查节点资源不足、污点未容忍、PVC未绑定、调度器日志。使用`kubectl describe pod <name>`查看Events，确认`Insufficient cpu/memory`或`Unschedulable`原因。

**Q3：StatefulSet扩容后Pod顺序错乱？**  
A：确保`serviceName`指向Headless Service，`volumeClaimTemplates`命名唯一，控制器未设置`podManagementPolicy: OrderedReady`以外的策略。

**Q4：NetworkPolicy生效后所有流量中断？**  
A：K8s默认允许所有进出流量。添加策略后需显式放行`kube-dns`（`coredns`）及`cni`插件所需流量（如Calico的`nodeagent`）。

```yaml
# 放行DNS与CNI必需流量（补充示例）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-cni
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
    - Ingress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Q5：Helm升级失败如何回滚？**  
A：使用`helm rollback <release-name> <revision>`，配合`--atomic`标志确保安装/升级失败时自动回退。生产环境建议开启`history-max`保留记录。
