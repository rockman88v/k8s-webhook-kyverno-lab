# Hướng dẫn tùy chỉnh Kyverno Policy

Các policy trong chart `templates-chart` được cấu hình bằng Helm values. Không chỉnh trực tiếp file trong `templates/`; thay vào đó, tạo file values override, render bằng Helm và kiểm tra manifest trước khi apply.

Quy trình thực hành và các values được chuẩn bị theo bài nằm trong [WORKSHOP_LAB.md](WORKSHOP_LAB.md). Danh sách use-case hiện còn hỗ trợ nằm trong [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md).

## Cách dùng

Chạy từ thư mục `use-cases`:

```bash
cd /home/vagrant/k8s-webhook-kyverno-lab/use-cases
cp templates-chart/values-custom.yaml values-myorg.yaml
```

Chỉnh `values-myorg.yaml`, sau đó render để xem kết quả:

```bash
helm template my-policies ./templates-chart \
  --values ./values-myorg.yaml > rendered-policies.yaml
```

Khi manifest đúng, apply các resource cần thiết:

```bash
kubectl apply -f rendered-policies.yaml
```

Để chỉ render một policy, thêm `--show-only`:

```bash
helm template my-policies ./templates-chart \
  --values ./values-myorg.yaml \
  --show-only templates/cpol-val-disallow-root-user.yaml
```

## Cấu trúc values

```yaml
policies:
  validate:
    enabled: true
  mutate:
    enabled: true
  generate:
    enabled: true

global:
  background: true
```

`enabled` ở cấp nhóm bật hoặc tắt toàn bộ nhóm policy. Mỗi policy cũng có `enabled` riêng. File override chỉ cần chứa các giá trị khác với `templates-chart/values.yaml`.

## Validate policy

Validate policy dùng `validationFailureAction`:

- `audit`: ghi vi phạm nhưng vẫn cho phép request.
- `enforce`: từ chối request vi phạm.

### Không cho chạy root

```yaml
policies:
  validate:
    disallowRoot:
      enabled: true
      failureAction: audit
      resourceKinds: [Pod]
      namespaceSelector:
        matchLabels:
          enforce-security: "true"
```

Template: `cpol-val-disallow-root-user.yaml`. Policy kiểm tra `securityContext.runAsNonRoot: true`.

### Bắt buộc image digest

```yaml
policies:
  validate:
    requireImageDigest:
      enabled: true
      failureAction: audit
      resourceKinds: [Pod, Deployment, StatefulSet]
      namespaceSelector:
        matchLabels:
          require-image-digest: "true"
```

Template: `cpol-val-require-image-digest.yaml`. Image phải dùng digest thay vì tag mutable như `latest`.

### Bắt buộc resource requests và limits

```yaml
policies:
  validate:
    enforceResources:
      enabled: true
      failureAction: enforce
      resourceKinds: [Pod]
      namespaceSelector:
        matchLabels:
          require-resources: "true"
```

Template: `cpol-val-enforce-resources.yaml`. Policy được render tên `enforce-resource-quotas` và yêu cầu CPU/memory cho cả `requests` lẫn `limits`.

### Hạn chế registry

```yaml
policies:
  validate:
    restrictRegistries:
      enabled: true
      failureAction: enforce
      resourceKinds: [Pod]
      approvedRegistries: [registry.example.com]
      namespaceSelector:
        matchLabels:
          restrict-registries: "true"
```

Template: `cpol-val-restrict-registries.yaml`.

### Bắt buộc metadata labels

```yaml
policies:
  validate:
    requireLabels:
      enabled: true
      failureAction: audit
      resourceKinds: [Pod]
      requiredLabels: [app, team, cost-center]
      namespaceSelector:
        matchLabels:
          require-labels: "true"
```

Template: `cpol-val-require-labels.yaml`.

## Mutate policy

Mutate policy hiện áp dụng lên `Pod`. Mutation được thực hiện tại admission; chart hiện tại không dùng `mutationFailureAction` vì field này không được CRD Kyverno đang dùng hỗ trợ.

### Thêm labels mặc định

```yaml
policies:
  mutate:
    addDefaultLabels:
      enabled: true
      namespaceSelector:
        matchLabels:
          auto-label: "true"
      labels:
        managed-by: kyverno
        environment: "{{ request.namespace }}"
      additionalLabels:
        team: platform
```

Template: `cpol-mut-add-default-labels.yaml`.

### Thêm security context

```yaml
policies:
  mutate:
    addSecurityContext:
      enabled: true
      namespaceSelector:
        matchLabels:
          auto-sec-context: "true"
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
```

Template: `cpol-mut-add-security-context.yaml`. Workload có `readOnlyRootFilesystem: true` phải có writable volume ở các đường dẫn cần ghi.

### Thêm resource mặc định

```yaml
policies:
  mutate:
    addDefaultResources:
      enabled: true
      namespaceSelector:
        matchLabels:
          auto-add-requests: "true"
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
```

Template: `cpol-mut-add-default-resources.yaml`. Template `cpol-mut-set-memory-requests.yaml` đã deprecated, không dùng cho cấu hình mới.

### Thêm Prometheus annotations

```yaml
policies:
  mutate:
    addMonitoringAnnotations:
      enabled: true
      namespaceSelector:
        matchLabels:
          prometheus-monitoring: "enabled"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: /metrics
        prometheus.io/scheme: http
```

Template: `cpol-mut-add-monitoring.yaml`.

### Thêm Istio sidecar mẫu

```yaml
policies:
  mutate:
    injectIstioSidecar:
      enabled: true
      namespaceSelector:
        matchLabels:
          istio-injection: "enabled"
      image:
        repository: istio/proxyv2
        tag: "1.17.0"
      excludeNamespaces: [kyverno, kube-system, istio-system]
```

Template: `cpol-mut-inject-istio.yaml`. Không bật đồng thời template này và cơ chế sidecar injection native của Istio trong cùng namespace.

## Generate policy

Generate policy trong chart hiện trigger khi tạo `Namespace`. Nếu selector cần kiểm tra label của Namespace, label phải có trong chính manifest tạo Namespace; gắn label sau khi Namespace được tạo không kích hoạt generate rule.

### Tạo NetworkPolicy

```yaml
policies:
  generate:
    generateNetworkPolicy:
      enabled: true
      resourceKinds: [Namespace]
      namespaceSelector:
        matchLabels:
          network-policy: "enabled"
      policyName: default-deny-ingress
      podSelector: {}
      policyTypes: [Ingress]
      allowedPorts: []
```

Template: `cpol-gen-network-policy.yaml`.

### Clone registry Secret

```yaml
policies:
  generate:
    cloneRegistrySecrets:
      enabled: true
      resourceKinds: [Namespace]
      namespaceSelector:
        matchLabels:
          clone-secrets: "true"
      sourceNamespace: default
      secretName: docker-credentials
      targetSecretName: docker-credentials
```

Template: `cpol-gen-clone-secrets.yaml`. Secret nguồn phải tồn tại trước. Background controller của Kyverno cần quyền `get` và `create` Secret; manifest RBAC tạm thời có trong Lab 3.

### Tạo ResourceQuota và LimitRange

```yaml
policies:
  generate:
    generateResourceQuota:
      enabled: true
      resourceKinds: [Namespace]
      quotas:
        dev:
          ruleName: create-dev-quota
          namespaceSelector:
            matchLabels:
              env: development
          quotaName: dev-quota
          hard:
            requests.cpu: "5"
            requests.memory: 10Gi
            limits.cpu: "10"
            limits.memory: 20Gi
            pods: "100"
    generateLimitRange:
      enabled: true
      resourceKinds: [Namespace]
      namespaceSelector:
        matchLabels: {}
      limitRangeName: default-limits
```

Templates: `cpol-gen-resource-quota.yaml` và `cpol-gen-limit-range.yaml`.

## Kiểm tra và xử lý sự cố

```bash
kubectl get clusterpolicy
kubectl get policyreport -A
kubectl get namespace --show-labels
kubectl logs -n kyverno -l app.kubernetes.io/name=kyverno --tail=200
```

Nếu policy không match, kiểm tra:

- Giá trị `enabled` ở cấp nhóm và cấp policy.
- Namespace label có khớp `namespaceSelector` hay không.
- Resource kind có nằm trong `resourceKinds` không.
- Manifest Helm render đúng: `helm template ...`.
- Với Generate, Namespace có được tạo kèm label hay không.

## Khuyến nghị

- Bắt đầu với `audit`, xem `PolicyReport`, rồi mới dùng `enforce`.
- Kiểm tra bằng `helm template` và `kubectl apply --dry-run=server` trước khi apply thật.
- Quản lý file values trong Git và ghi rõ lý do cho mỗi override.
- Chỉ xóa resource do lab tạo; không dùng `kubectl delete clusterpolicy --all` trên cluster dùng chung.
