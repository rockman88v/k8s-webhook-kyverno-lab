# Kyverno Use-Cases: Validate, Mutate, Generate
## Hướng dẫn thực tế cho Workshop

---

## 1. VALIDATE POLICIES (5 Use-Cases)

### 1.1 Disallow Running as Root User
**Pain Point**: Containers chạy với root user gây rủi ro bảo mật cao. Cần enforce toàn cluster.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment, StatefulSet, DaemonSet, Job`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root-user
spec:
  validationFailureAction: enforce  # enforce: block, audit: log only
  rules:
  - name: check-run-as-non-root
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Running as root is not allowed"
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
```

---

### 1.2 Require Image Digest Instead of Tag
**Pain Point**: Image tags có thể thay đổi, không đảm bảo reproducibility. Phải dùng digest để cố định image.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment, StatefulSet, DaemonSet`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: enforce
  rules:
  - name: validate-image-digest
    match:
      resources:
        kinds:
        - Pod
      namespaceSelector:
        matchLabels:
          require-image-digest: "true"
    validate:
      message: "Image must be referenced by digest (sha256:...), not by tag"
      pattern:
        spec:
          containers:
          - image: "*/*/sha256:*"
```

---

### 1.3 Enforce Resource Requests and Limits
**Pain Point**: Pods không khai báo resource requests/limits gây khó dự đoán workload, làm cluster scheduling bị lệch.

**Scope**: Namespace-specific (dev, staging, prod có yêu cầu khác nhau)  
**Object Type**: `ClusterPolicy` (with namespace selector)  
**Kind**: `Pod, Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resources
spec:
  validationFailureAction: enforce
  rules:
  - name: validate-resources
    match:
      resources:
        kinds:
        - Pod
      namespaceSelector:
        matchLabels:
          require-resources: "true"
    validate:
      message: "CPU and memory requests/limits are required"
      pattern:
        spec:
          containers:
          - resources:
              requests:
                memory: "?*"
                cpu: "?*"
              limits:
                memory: "?*"
                cpu: "?*"
```

---

### 1.4 Restrict Container Image Registries
**Pain Point**: Devs có thể pull images từ untrusted registries, gây security breach. Cần whitelist approved registries.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-registries
spec:
  validationFailureAction: enforce
  rules:
  - name: validate-registry
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Images must come from: gcr.io, docker.io, or quay.io only"
      pattern:
        spec:
          containers:
          - image: "gcr.io/* | docker.io/* | quay.io/*"
```

---

### 1.5 Require Pod Disruption Budget (PDB)
**Pain Point**: Pods bị terminate khi cluster maintenance → downtime. Cần PDB để ensure availability.

**Scope**: Namespace-specific (chỉ critical services)  
**Object Type**: `Policy`  
**Kind**: `Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: require-pdb
  namespace: production
spec:
  validationFailureAction: audit
  rules:
  - name: validate-pdb-exists
    match:
      resources:
        kinds:
        - Deployment
        selector:
          matchLabels:
            required-pdb: "true"
    validate:
      message: "Pod Disruption Budget must exist for this deployment"
      # Validation yêu cầu PDB phải tồn tại (complex validation)
      # Có thể dùng CEL rule hoặc kiểm tra via Kyverno webhook
```

---

## 2. MUTATE POLICIES (10 Use-Cases)

### 2.1 Inject Istio Sidecar Proxy
**Pain Point**: DevOps phải manually inject Istio sidecar vào từng pod, dễ quên, inconsistent.

**Scope**: Namespace-specific (inject theo label)  
**Object Type**: `Policy` hoặc `ClusterPolicy` (với namespace selector)  
**Kind**: `Pod`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: inject-istio-sidecar
spec:
  mutationFailureAction: audit
  rules:
  - name: inject-sidecar
    match:
      resources:
        kinds:
        - Pod
      namespaceSelector:
        matchLabels:
          istio-injection: enabled
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            version: v1
        spec:
          containers:
          - name: istio-proxy
            image: istio/proxyv2:1.17.0
            ports:
            - containerPort: 15000
```

---

### 2.2 Add Security Context to All Containers
**Pain Point**: Containers mặc định chạy với permissive capabilities. Cần auto-apply restrictive security context.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-security-context
spec:
  mutationFailureAction: audit
  rules:
  - name: add-security-context
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"
            securityContext:
              runAsNonRoot: true
              runAsUser: 1000
              readOnlyRootFilesystem: true
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                - ALL
```

---

### 2.3 Add Default Labels to All Workloads
**Pain Point**: Workloads không có consistent labels → khó quản lý, monitoring, cost tracking.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  mutationFailureAction: audit
  rules:
  - name: add-managed-by-label
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            managed-by: kyverno
            environment: "{{ request.namespace }}"
            created-by: "{{ serviceAccountName }}"
```

---

### 2.4 Add Registry Credentials to Pods
**Pain Point**: Pods pull từ private registries nhưng cần imagePullSecrets. Manual add dễ quên.

**Scope**: Namespace-specific (per-registry)  
**Object Type**: `Policy`  
**Kind**: `Pod, Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: add-registry-credentials
  namespace: default
spec:
  mutationFailureAction: audit
  rules:
  - name: add-imagepullsecret
    match:
      resources:
        kinds:
        - Pod
        selector:
          matchLabels:
            use-private-registry: "true"
    mutate:
      patchStrategicMerge:
        spec:
          imagePullSecrets:
          - name: docker-credentials
```

---

### 2.5 Add Network Policy Label to Pods
**Pain Point**: Pods mới không có network policy label → traffic không được regulate.

**Scope**: Namespace-specific  
**Object Type**: `Policy`  
**Kind**: `Pod, Deployment`  

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: add-network-policy-label
  namespace: production
spec:
  mutationFailureAction: audit
  rules:
  - name: add-network-label
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            network-zone: internal
            allow-ingress: "false"
```

---

### 2.6 Add Resource Requests for Memory and CPU
**Pain Point**: Pods không request resources → cluster scheduler mất hướng, OOMKill random.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment, StatefulSet`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-resources
spec:
  mutationFailureAction: audit
  rules:
  - name: set-default-requests
    match:
      resources:
        kinds:
        - Pod
      namespaceSelector:
        matchExpressions:
        - key: auto-resources
          operator: In
          values:
          - "true"
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"
            resources:
              requests:
                memory: "128Mi"
                cpu: "100m"
              limits:
                memory: "512Mi"
                cpu: "500m"
```

---

### 2.7 Add Prometheus Annotations for Monitoring
**Pain Point**: Monitoring bị miss metrics vì pods không expose metrics endpoint annotations.

**Scope**: Namespace-specific  
**Object Type**: `Policy`  
**Kind**: `Pod, Deployment`  

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: add-prometheus-annotations
  namespace: monitoring
spec:
  mutationFailureAction: audit
  rules:
  - name: add-prometheus-scrape
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "8080"
            prometheus.io/path: "/metrics"
```

---

### 2.8 Add Pod Priority Class
**Pain Point**: Critical pods bị evict trước non-critical pods khi node out-of-resources.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Pod, Deployment`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-pod-priority
spec:
  mutationFailureAction: audit
  rules:
  - name: add-priority-critical
    match:
      resources:
        kinds:
        - Pod
        selector:
          matchLabels:
            priority: critical
    mutate:
      patchStrategicMerge:
        spec:
          priorityClassName: critical-pods
```

---

### 2.9 Add Init Container for Dependency Check
**Pain Point**: Services start trước dependencies sẵn sàng → race conditions, connection errors.

**Scope**: Namespace-specific  
**Object Type**: `Policy`  
**Kind**: `Pod, Deployment`  

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: add-init-wait-for-db
  namespace: backend
spec:
  mutationFailureAction: audit
  rules:
  - name: add-init-container
    match:
      resources:
        kinds:
        - Pod
        selector:
          matchLabels:
            wait-for-db: "true"
    mutate:
      patchStrategicMerge:
        spec:
          initContainers:
          - name: wait-for-db
            image: busybox:1.35
            command: ['sh', '-c', 'until nc -z db 5432; do echo waiting for db; sleep 2; done;']
```

---

### 2.10 Add Audit Logging Sidecar
**Pain Point**: Compliance yêu cầu log mọi API calls. Cần sidecar logging cho audit trail.

**Scope**: Namespace-specific (regulated environments)  
**Object Type**: `Policy`  
**Kind**: `Pod, Deployment`  

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: add-audit-sidecar
  namespace: compliance
spec:
  mutationFailureAction: audit
  rules:
  - name: inject-audit-logger
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - name: audit-logger
            image: audit-logger:v1
            volumeMounts:
            - name: shared-logs
              mountPath: /var/log
          volumes:
          - name: shared-logs
            emptyDir: {}
```

---

## 3. GENERATE POLICIES (5 Use-Cases)

### 3.1 Auto-Create NetworkPolicy for New Namespaces
**Pain Point**: Namespaces mới không có network isolation → traffic có thể đi qua tất cả pods.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Namespace`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-network-policy
spec:
  rules:
  - name: create-default-deny-ingress
    match:
      resources:
        kinds:
        - Namespace
        selector:
          matchLabels:
            network-policy: enabled
    generate:
      kind: NetworkPolicy
      name: default-deny-ingress
      namespace: "{{ request.object.metadata.name }}"
      data:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: default-deny-ingress
        spec:
          podSelector: {}
          policyTypes:
          - Ingress
```

---

### 3.2 Clone Registry Secrets to New Namespaces
**Pain Point**: Pods ở namespaces khác không thể pull từ private registries (missing imagePullSecrets).

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Namespace`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: clone-registry-secret
spec:
  rules:
  - name: clone-docker-secret
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: Secret
      name: docker-registry-credentials
      namespace: "{{ request.object.metadata.name }}"
      clone:
        namespace: default
        name: docker-registry-credentials
```

---

### 3.3 Create RBAC ServiceAccount with Role Binding
**Pain Point**: Teams khó setup ServiceAccounts + RoleBindings mỗi khi tạo namespace mới.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Namespace`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-rbac
spec:
  rules:
  - name: create-default-sa
    match:
      resources:
        kinds:
        - Namespace
        selector:
          matchLabels:
            auto-rbac: "true"
    generate:
      kind: ServiceAccount
      name: default-app
      namespace: "{{ request.object.metadata.name }}"
      data:
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: default-app
  
  - name: create-role-binding
    match:
      resources:
        kinds:
        - Namespace
        selector:
          matchLabels:
            auto-rbac: "true"
    generate:
      kind: RoleBinding
      name: default-app-reader
      namespace: "{{ request.object.metadata.name }}"
      data:
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
          name: default-app-reader
        subjects:
        - kind: ServiceAccount
          name: default-app
        roleRef:
          kind: Role
          name: pod-reader
          apiGroup: rbac.authorization.k8s.io
```

---

### 3.4 Create Resource Quota for New Namespaces
**Pain Point**: Namespaces mới không giới hạn resource → một app có thể consume hết cluster resources.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Namespace`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-resource-quota
spec:
  rules:
  - name: create-quota
    match:
      resources:
        kinds:
        - Namespace
        selector:
          matchLabels:
            env: development
    generate:
      kind: ResourceQuota
      name: dev-quota
      namespace: "{{ request.object.metadata.name }}"
      data:
        apiVersion: v1
        kind: ResourceQuota
        metadata:
          name: dev-quota
        spec:
          hard:
            requests.cpu: "5"
            requests.memory: "10Gi"
            limits.cpu: "10"
            limits.memory: "20Gi"
            pods: "100"
```

---

### 3.5 Create LimitRange for Container Resource Defaults
**Pain Point**: Containers không có default resource limits → scheduling bị lệch, OOMKill xảy ra.

**Scope**: Cluster-wide  
**Object Type**: `ClusterPolicy`  
**Kind**: `Namespace`  

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-limit-range
spec:
  rules:
  - name: create-limit-range
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: LimitRange
      name: default-limits
      namespace: "{{ request.object.metadata.name }}"
      data:
        apiVersion: v1
        kind: LimitRange
        metadata:
          name: default-limits
        spec:
          limits:
          - max:
              cpu: "2"
              memory: "2Gi"
            min:
              cpu: "50m"
              memory: "64Mi"
            default:
              cpu: "500m"
              memory: "512Mi"
            defaultRequest:
              cpu: "100m"
              memory: "128Mi"
            type: Container
```

---

## Summary Table

| Use-Case | Type | Scope | Policy Type | Pain Point |
|----------|------|-------|-------------|-----------|
| **Validate #1** | Disallow Root | Cluster | ClusterPolicy | Security risk |
| **Validate #2** | Image Digest | Cluster | ClusterPolicy | Reproducibility |
| **Validate #3** | Resources | Namespace | ClusterPolicy+Selector | Unpredictable load |
| **Validate #4** | Registry Whitelist | Cluster | ClusterPolicy | Untrusted images |
| **Validate #5** | Require PDB | Namespace | Policy | Downtime risk |
| **Mutate #1** | Istio Sidecar | Namespace | ClusterPolicy+Label | Manual injection |
| **Mutate #2** | Security Context | Cluster | ClusterPolicy | Permissive defaults |
| **Mutate #3** | Default Labels | Cluster | ClusterPolicy | Inconsistent labeling |
| **Mutate #4** | Registry Credentials | Namespace | Policy | Missing imagePullSecrets |
| **Mutate #5** | Network Label | Namespace | Policy | No traffic control |
| **Mutate #6** | Default Resources | Cluster | ClusterPolicy | Resource exhaustion |
| **Mutate #7** | Prometheus Annotations | Namespace | Policy | Missing metrics |
| **Mutate #8** | Pod Priority | Cluster | ClusterPolicy | Unfair eviction |
| **Mutate #9** | Init Container | Namespace | Policy | Race conditions |
| **Mutate #10** | Audit Sidecar | Namespace | Policy | Compliance gap |
| **Generate #1** | NetworkPolicy | Cluster | ClusterPolicy | No isolation |
| **Generate #2** | Registry Secret | Cluster | ClusterPolicy | Access denied |
| **Generate #3** | RBAC | Cluster | ClusterPolicy | Manual setup |
| **Generate #4** | ResourceQuota | Cluster | ClusterPolicy | Resource hogging |
| **Generate #5** | LimitRange | Cluster | ClusterPolicy | OOMKill |

---

## Lưu ý cho Workshop

1. **Validate vs Mutate vs Generate**:
   - **Validate**: Kiểm tra compliance, từ chối nếu không đạt (enforce/audit mode)
   - **Mutate**: Tự động sửa/thêm cấu hình để compliance
   - **Generate**: Tự động tạo resource mới (không khi namespace tạo, resource tạo, etc.)

2. **ClusterPolicy vs Policy**:
   - **ClusterPolicy**: Cluster-wide, toàn bộ namespaces (trừ khi có namespace selector)
   - **Policy**: Namespace-specific, chỉ áp dụng trong namespace đó

3. **Failure Action**:
   - **enforce**: Block request nếu vi phạm (DENY)
   - **audit**: Log violation nhưng allow request (LOG ONLY)

4. **Performance Consideration**:
   - Tránh quá nhiều ClusterPolicy (gọi webhook cho mỗi request)
   - Dùng namespace selectors để giảm scope
   - Test với audit mode trước khi enforce

5. **Ordering**:
   - Khi có multiple policies, xử lý theo thứ tự: Mutate → Validate → Generate
   - Mutate policies chạy trước nên có thể modify spec rồi validate
