# Kyverno Use-Cases: Validate, Mutate, Generate

Tài liệu này mô tả các use-case hiện được hỗ trợ bởi Helm chart `templates-chart`. Mỗi policy phải được render qua Helm với values phù hợp trước khi apply. Quy trình thực hành đầy đủ, lệnh render và cleanup nằm trong [WORKSHOP_LAB.md](WORKSHOP_LAB.md).

## Cách sử dụng chart

Chạy các lệnh từ thư mục `use-cases`. Ba file values dành cho các lab chỉ bật policy cần thiết của từng bài:

| Lab | Values file | Nhóm policy |
|---|---|---|
| Lab 1 | `templates-chart/values-lab-1.yaml` | Validate |
| Lab 2 | `templates-chart/values-lab-2.yaml` | Mutate |
| Lab 3 | `templates-chart/values-lab-3.yaml` | Generate |

Không apply trực tiếp các file trong `templates-chart/templates`. Chúng là Helm template và cần được render trước:

```bash
helm template workshop-lab ./templates-chart \
  --values ./templates-chart/values-lab-1.yaml \
  --show-only templates/cpol-val-disallow-root-user.yaml
```

## 1. Validate Policies

Các policy Validate kiểm tra resource tại admission. `validationFailureAction` nhận `audit` để ghi vi phạm nhưng vẫn cho phép request, hoặc `enforce` để từ chối request.

| Use-case | Helm template | ClusterPolicy | Selector và cấu hình chính |
|---|---|---|---|
| Không chạy container bằng root | `cpol-val-disallow-root-user.yaml` | `disallow-root-user` | `policies.validate.disallowRoot`; kiểm tra `runAsNonRoot: true` |
| Bắt buộc image digest | `cpol-val-require-image-digest.yaml` | `require-image-digest` | `policies.validate.requireImageDigest`; image phải theo dạng digest |
| Bắt buộc requests và limits | `cpol-val-enforce-resources.yaml` | `enforce-resource-quotas` | `policies.validate.enforceResources`; CPU và memory requests/limits |
| Hạn chế image registry | `cpol-val-restrict-registries.yaml` | `restrict-untrusted-registries` | `policies.validate.restrictRegistries`; `approvedRegistries` |
| Bắt buộc metadata labels | `cpol-val-require-labels.yaml` | `require-labels` | `policies.validate.requireLabels`; `requiredLabels` |

### 1.1 Không chạy container bằng root

Policy kiểm tra từng container của resource được match có `securityContext.runAsNonRoot: true`. Trong `values-lab-1.yaml`, policy áp dụng cho Pod trong namespace có label:

```yaml
require-resources: "true"
```

### 1.2 Bắt buộc image digest

Policy chỉ chấp nhận image có digest, thay vì mutable tag như `latest` hoặc `v1`. Bật policy bằng `policies.validate.requireImageDigest.enabled: true` và gán label selector đã cấu hình trong values, mặc định là:

```yaml
require-image-digest: "true"
```

### 1.3 Bắt buộc requests và limits

Policy `enforce-resource-quotas` yêu cầu mọi container có đủ bốn trường: CPU/memory `requests` và CPU/memory `limits`. Selector hiện dùng cho Lab 1 là:

```yaml
require-resources: "true"
```

Tên ClusterPolicy là `enforce-resource-quotas`, không phải `require-resources`.

### 1.4 Hạn chế image registry

Policy `restrict-untrusted-registries` kiểm tra image theo danh sách `approvedRegistries`. Lab 1 chỉ áp dụng policy lên namespace có label:

```yaml
restrict-registries: "true"
```

Danh sách mặc định trong values Lab 1 là `gcr.io`, `docker.io` và `quay.io`. Dùng một registry ngoài danh sách để kiểm thử từ chối. Image thuộc `docker.io` sẽ được cho phép.

### 1.5 Bắt buộc labels

Policy `require-labels` kiểm tra các metadata label đã khai báo trong `requiredLabels`, mặc định trong values gốc là `app`, `version`, và `managed-by`. Đây là use-case validate metadata đang được chart hỗ trợ; chart không có policy yêu cầu PodDisruptionBudget.

## 2. Mutate Policies

Các policy Mutate tự bổ sung cấu hình khi resource được tạo. Các policy hiện tại match `Pod`; không có template tương ứng cho Deployment hoặc StatefulSet trực tiếp.

| Use-case | Helm template | ClusterPolicy | Selector và cấu hình chính |
|---|---|---|---|
| Thêm restrictive security context | `cpol-mut-add-security-context.yaml` | `add-security-context` | `auto-sec-context: "true"`; `securityContext` |
| Thêm resource defaults | `cpol-mut-add-default-resources.yaml` | `add-default-resources` | `auto-add-requests: "true"`; `resources` |
| Thêm Prometheus annotations | `cpol-mut-add-monitoring.yaml` | `add-monitoring-annotations` | `prometheus-monitoring: "enabled"`; `annotations` |
| Thêm Istio proxy mẫu | `cpol-mut-inject-istio.yaml` | `inject-istio-sidecar-auto` | `istio-injection: "enabled"`; `image`, `excludeNamespaces` |

### 2.1 Restrictive security context

Policy thêm `runAsNonRoot`, `runAsUser`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation` và danh sách capability cần drop. Chỉ bật nó cho namespace có `auto-sec-context: "true"`.

Lưu ý: `readOnlyRootFilesystem: true` yêu cầu workload có writable volume tại các đường dẫn ứng dụng cần ghi. Ví dụ NGINX mặc định cần cấu hình thêm writable volume; workshop dùng BusyBox cho bài kiểm thử để tránh lỗi runtime này.

### 2.2 Resource defaults

Policy thêm CPU và memory requests/limits theo `policies.mutate.addDefaultResources.resources`. Selector mặc định của Lab 2 là `auto-add-requests: "true"`. Đây là policy thay thế cho template `cpol-mut-set-memory-requests.yaml` đã deprecated.

### 2.3 Prometheus annotations

Policy thêm các annotation `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path` và `prometheus.io/scheme` cho Pod trong namespace có `prometheus-monitoring: "enabled"`.

### 2.4 Istio sidecar mẫu

Policy thêm container `istio-proxy` vào Pod trong namespace có `istio-injection: "enabled"`. Image, tag và namespace loại trừ cấu hình được trong values. Không bật đồng thời policy này với injection native của Istio trong cùng namespace.

Chart không còn template cho các use-case sau và chúng không được khuyến nghị trong workshop hiện tại: tự thêm `imagePullSecrets`, nhãn NetworkPolicy, `priorityClassName`, init container chờ database, hoặc audit logging sidecar.

## 3. Generate Policies

Các policy Generate tạo resource khi admission request tạo Namespace match selector. Nhãn phải nằm trong manifest tạo Namespace; gắn label sau khi Namespace đã được tạo sẽ không kích hoạt generate rule.

| Use-case | Helm template | ClusterPolicy | Resource tạo ra | Selector và cấu hình chính |
|---|---|---|---|---|
| Tạo default-deny NetworkPolicy | `cpol-gen-network-policy.yaml` | `generate-network-policies` | `NetworkPolicy` | `network-policy: "enabled"` |
| Clone registry Secret | `cpol-gen-clone-secrets.yaml` | `clone-registry-secrets` | `Secret` | `clone-secrets: "true"` |
| Tạo ResourceQuota | `cpol-gen-resource-quota.yaml` | `generate-resource-quotas` | `ResourceQuota` | `env: development` hoặc `env: production` |
| Tạo LimitRange | `cpol-gen-limit-range.yaml` | `generate-limit-ranges` | `LimitRange` | Match mọi Namespace trong values Lab 3 |

### 3.1 Default-deny NetworkPolicy

Policy tạo `NetworkPolicy` tên `default-deny-ingress` khi Namespace được tạo với label:

```yaml
network-policy: "enabled"
```

### 3.2 Clone registry Secret

Policy clone Secret `docker-credentials` từ namespace `default` sang Namespace mới có label `clone-secrets: "true"`. Secret nguồn phải tồn tại trước khi policy được áp dụng. Background controller của Kyverno cũng cần quyền `get` và `create` Secrets; manifest RBAC tạm thời cho Lab 3 nằm trong [WORKSHOP_LAB.md](WORKSHOP_LAB.md).

### 3.3 ResourceQuota

Policy tạo quota `dev-quota` cho Namespace có `env: development` và `prod-quota` cho Namespace có `env: production`. Mỗi quota có resource hard limits riêng trong `policies.generate.generateResourceQuota.quotas`.

### 3.4 LimitRange

Policy tạo LimitRange `default-limits` cho Namespace mới, gồm giới hạn min/max và default requests/limits cho Container và Pod. Điều chỉnh tại `policies.generate.generateLimitRange`.

Chart không có Generate policy tạo ServiceAccount hoặc RoleBinding cho Namespace mới.

## Phạm vi được hỗ trợ

Use-case này được xác định bởi các template ClusterPolicy cấu hình được qua `values.yaml` và các values theo Lab. Các template namespace-scoped hard-code, template duplicate và template deprecated không nằm trong chart nữa hoặc không được render trong lệnh `--show-only` của workshop.

## Lưu ý khi vận hành trong thực tế

- Bắt đầu validation với `audit`, xem `PolicyReport`, rồi mới chuyển sang `enforce`.
- Dùng `match.resources.namespaceSelector` cho policy áp dụng lên workload trong namespace có label.
- Dùng `match.resources.selector` cho generate policy cần lọc label của chính resource `Namespace`.
- Kiểm tra resource đã render trước khi apply: `helm template ...`.
- Chỉ xóa các ClusterPolicy, RBAC và resource test do lab đã tạo; không chạy `kubectl delete clusterpolicy --all` trên cluster dùng chung.
