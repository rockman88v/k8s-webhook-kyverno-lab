# Workshop Kyverno: Phòng thực hành

Tài liệu này dùng chart `templates-chart` trong cùng thư mục. Mỗi bài có một file values riêng, chỉ bật các policy cần cho bài đó. Các lệnh `helm template` bên dưới phải được chạy từ thư mục `use-cases`.

## Chuẩn bị

### Điều kiện tiên quyết

- Kubernetes 1.16 trở lên
- `kubectl` đã trỏ tới cluster cần thực hành
- Helm 3 trở lên
- Kiến thức cơ bản về manifest Kubernetes và Kyverno

### Cài Kyverno

```bash
kubectl create namespace kyverno-system
kubectl create namespace kyverno-test
kubectl create namespace dev
kubectl create namespace prod

helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno --namespace kyverno-system

kubectl get pods -n kyverno-system
kubectl get crd | grep kyverno
```

Chờ các pod Kyverno ở trạng thái `Running` trước khi áp dụng policy.

## Lab 1: Policy VALIDATE

### Mục tiêu

Học cách kiểm tra và ngăn cấu hình không phù hợp bằng validation policy.

### Kịch bản

Trong bài này, namespace có label `require-resources=true` sẽ được kiểm tra container chạy non-root và khai báo resources. Namespace có label `restrict-registries=true` chỉ được dùng image từ `gcr.io`, `docker.io` hoặc `quay.io`.

### Bước 1.1: Tạo label cho namespace

```bash
kubectl label namespace kyverno-test require-resources=true restrict-registries=true --overwrite
kubectl label namespace prod require-resources=true restrict-registries=true env=production --overwrite
```

Các label này khớp với selector trong `templates-chart/values-lab-1.yaml`.

### Bước 1.2: Render và apply đúng các policy của Lab 1

Ba file template dưới đây là toàn bộ policy được dùng trong bài này:

- `templates-chart/templates/cpol-val-disallow-root-user.yaml`
- `templates-chart/templates/cpol-val-enforce-resources.yaml`
- `templates-chart/templates/cpol-val-restrict-registries.yaml`

```bash
helm template workshop-lab ./templates-chart \
  -f ./templates-chart/values-lab-1.yaml \
  --show-only templates/cpol-val-disallow-root-user.yaml \
  --show-only templates/cpol-val-enforce-resources.yaml \
  --show-only templates/cpol-val-restrict-registries.yaml \
  | kubectl apply -f -
```

`failureAction` của ba policy đang là `audit`, `audit`, `enforce` tương ứng. Policy registry sẽ từ chối image không được phép ngay từ đầu.

### Bước 1.3: Kiểm tra policy non-root

```bash
cat <<'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-root-pod
spec:
  containers:
    - name: app
      image: nginx:latest
      securityContext:
        runAsNonRoot: false
EOF
```

Ở chế độ `audit`, pod vẫn có thể được tạo nhưng sẽ có vi phạm trong `PolicyReport`. Để thử chế độ chặn, patch policy rồi chạy lại:

```bash
kubectl patch clusterpolicy disallow-root-user \
  --type merge -p '{"spec":{"validationFailureAction":"enforce"}}'
kubectl delete pod test-root-pod -n kyverno-test --ignore-not-found
```

### Bước 1.4: Tạo pod hợp lệ

```bash
cat <<'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant-pod
spec:
  containers:
    - name: app
      image: nginx:latest
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      resources:
        requests:
          cpu: 10m
          memory: 16Mi
        limits:
          cpu: 100m
          memory: 128Mi
EOF
```

### Bước 1.5: Kiểm tra registry không được phép

```bash
cat <<'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-untrusted-registry
spec:
  containers:
    - name: app
      image: registry.example.invalid/untrusted/app:v1
EOF
```

Kết quả mong đợi: request bị từ chối vì registry không nằm trong danh sách được phép. `docker.io/untrusted/app:v1` không phải ca kiểm thử hợp lệ vì `docker.io` đang được cho phép.

### Bước 1.6: Kiểm tra resources

```bash
cat <<'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-no-resources
spec:
  containers:
    - name: app
      image: nginx:latest
EOF
```

Trong `audit`, pod được tạo và vi phạm được ghi nhận; khi chuyển `enforce-resource-quotas` sang `enforce`, request tương tự sẽ bị từ chối.

### Bước 1.7: Xem vi phạm

```bash
kubectl get policyreport -A
kubectl describe policyreport -n kyverno-test
kubectl logs -n kyverno-system -l app.kubernetes.io/name=kyverno --tail=100
```

Kyverno tự tạo `PolicyReport`; không cần thêm manifest PolicyReport trong chart.

### Cleanup Lab 1

```bash
kubectl delete pod test-root-pod test-compliant-pod test-untrusted-registry test-no-resources \
  -n kyverno-test --ignore-not-found
kubectl delete clusterpolicy disallow-root-user enforce-resource-quotas restrict-untrusted-registries \
  --ignore-not-found
kubectl label namespace kyverno-test require-resources- restrict-registries- --ignore-not-found
kubectl label namespace prod require-resources- restrict-registries- --ignore-not-found
```

## Lab 2: Policy MUTATE

### Mục tiêu

Học cách tự động bổ sung cấu hình chuẩn cho Pod.

### Bước 2.1: Gắn label cho namespace

```bash
kubectl label namespace kyverno-test \
  auto-sec-context=true \
  auto-add-requests=true \
  prometheus-monitoring=enabled \
  istio-injection=enabled --overwrite
kubectl label namespace dev auto-add-requests=true --overwrite
```

Các selector này khớp với `templates-chart/values-lab-2.yaml`.

### Bước 2.2: Render và apply đúng các policy của Lab 2

```bash
helm template workshop-lab ./templates-chart \
  -f ./templates-chart/values-lab-2.yaml \
  --show-only templates/cpol-mut-add-security-context.yaml \
  --show-only templates/cpol-mut-add-default-resources.yaml \
  --show-only templates/cpol-mut-add-monitoring.yaml \
  --show-only templates/cpol-mut-inject-istio.yaml \
  | kubectl apply -f -
```

### Bước 2.3: Kiểm tra security context và resources

```bash
cat <<'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-mutation-sec-context
spec:
  containers:
    - name: app
      image: nginx:latest
EOF

kubectl get pod test-mutation-sec-context -n kyverno-test -o yaml
```

Pod phải nhận `runAsNonRoot`, `runAsUser`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation` và `capabilities.drop`. Pod trong namespace `dev` cũng nhận resources mặc định nếu được tạo sau khi policy đã apply.

### Bước 2.4: Kiểm tra annotation Prometheus

```bash
cat <<'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-mutation-monitoring
spec:
  containers:
    - name: app
      image: docker.io/library/nginx:latest
EOF

kubectl get pod test-mutation-monitoring -n kyverno-test -o jsonpath='{.metadata.annotations}'; echo
```

Annotation `prometheus.io/scrape`, `port`, `path` và `scheme` được thêm tự động.

### Bước 2.5: Kiểm tra Istio sidecar

```bash
kubectl get pod test-mutation-sec-context -n kyverno-test \
  -o jsonpath='{.spec.containers[*].name}'; echo
```

Kết quả có thể gồm container `istio-proxy`. Template này chỉ là ví dụ Kyverno; trong cluster có Istio thật, cần cân nhắc dùng cơ chế injection của Istio và không bật đồng thời hai cơ chế.

### Cleanup Lab 2

```bash
kubectl delete pod test-mutation-sec-context test-mutation-monitoring \
  -n kyverno-test --ignore-not-found
kubectl delete deployment app-no-resources -n dev --ignore-not-found
kubectl delete clusterpolicy add-security-context add-default-resources \
  add-monitoring-annotations inject-istio-sidecar-auto --ignore-not-found
kubectl label namespace kyverno-test auto-sec-context- auto-add-requests- \
  prometheus-monitoring- istio-injection- --ignore-not-found
kubectl label namespace dev auto-add-requests- --ignore-not-found
```

## Lab 3: Policy GENERATE

### Mục tiêu

Học cách tự động tạo resource khi namespace mới được tạo.

### Bước 3.1: Tạo secret nguồn

Policy clone secret cần secret nguồn tồn tại trước khi namespace đích được tạo.

```bash
kubectl apply -f - -n default <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: docker-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6e319
EOF
```

Secret trên chỉ phục vụ minh họa. Trong môi trường thật, hãy tạo dữ liệu bằng `kubectl create secret docker-registry` thay vì đưa credential vào shell history.

### Bước 3.2: Render và apply đúng các policy của Lab 3

```bash
helm template workshop-lab ./templates-chart \
  -f ./templates-chart/values-lab-3.yaml \
  --show-only templates/cpol-gen-network-policy.yaml \
  --show-only templates/cpol-gen-resource-quota.yaml \
  --show-only templates/cpol-gen-limit-range.yaml \
  --show-only templates/cpol-gen-clone-secrets.yaml \
  | kubectl apply -f -
```

### Bước 3.3: Tạo namespace và kiểm tra NetworkPolicy

```bash
kubectl create namespace test-gen-ns
kubectl label namespace test-gen-ns network-policy=enabled clone-secrets=true
kubectl get networkpolicy -n test-gen-ns
kubectl describe networkpolicy default-deny-ingress -n test-gen-ns
```

Kỳ vọng: có NetworkPolicy `default-deny-ingress` và secret `docker-credentials` được clone.

### Bước 3.4: Kiểm tra ResourceQuota

```bash
kubectl create namespace test-dev-quota
kubectl label namespace test-dev-quota env=development
kubectl get resourcequota -n test-dev-quota
kubectl describe resourcequota dev-quota -n test-dev-quota
```

### Bước 3.5: Kiểm tra LimitRange

```bash
kubectl create namespace test-limits
kubectl get limitrange -n test-limits
kubectl describe limitrange default-limits -n test-limits
```

### Bước 3.6: Kiểm tra tác động của resource được generate

```bash
cat <<'EOF' | kubectl apply -f - -n test-dev-quota
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: app
      image: docker.io/library/nginx:latest
EOF

kubectl get pod test-pod -n test-dev-quota -o jsonpath='{.spec.containers[0].resources}'; echo
```

`LimitRange` sẽ cung cấp resources mặc định cho Pod mới.

### Cleanup Lab 3

```bash
kubectl delete namespace test-gen-ns test-dev-quota test-limits --ignore-not-found
kubectl delete clusterpolicy generate-network-policies generate-resource-quotas \
  generate-limit-ranges clone-registry-secrets --ignore-not-found
kubectl delete secret docker-credentials -n default --ignore-not-found
```

## Lab 4: Quản lý policy

```bash
kubectl get clusterpolicy -o wide
kubectl get policy -A
kubectl describe clusterpolicy disallow-root-user
kubectl get policyreport -A
kubectl get policyreport -n kyverno-test -o yaml
```

Chuyển policy từ audit sang enforce:

```bash
kubectl patch clusterpolicy disallow-root-user \
  --type merge -p '{"spec":{"validationFailureAction":"enforce"}}'
```

Khi hoàn tất, xóa đúng các policy đã render ở lab tương ứng. Không dùng `kubectl delete clusterpolicy --all` nếu cluster đang có policy của workload khác.

## Lab 5: Xử lý sự cố và thực hành tốt

### Policy không được áp dụng

```bash
kubectl get pods -n kyverno-system
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
kubectl get namespace kyverno-test --show-labels
kubectl get clusterpolicy -o yaml
```

Kiểm tra namespace có đúng label theo values của lab hay không. Nếu policy đã được render nhưng không match, selector là nơi cần kiểm tra đầu tiên.

### Theo dõi violation và hiệu năng

```bash
kubectl get policyreport -A
kubectl top pod -n kyverno-system
kubectl logs -n kyverno-system -l app.kubernetes.io/name=kyverno --tail=200
```

Nên bắt đầu với `audit`, theo dõi PolicyReport, sau đó mới chuyển sang `enforce`. Chỉ áp dụng policy lên namespace có selector rõ ràng và loại trừ namespace hệ thống khi cần.

## Dọn toàn bộ môi trường sau workshop

```bash
kubectl delete namespace kyverno-test dev prod --ignore-not-found
helm uninstall kyverno -n kyverno-system
kubectl delete namespace kyverno-system --ignore-not-found
```

## Bạn đã học được

- Validation policy để phát hiện hoặc chặn cấu hình không phù hợp.
- Mutation policy để tự động chuẩn hóa Pod.
- Generate policy để tạo resource theo sự kiện namespace.
- Cách dùng selector và values riêng để giới hạn phạm vi policy.
- Sự khác nhau giữa `audit` và `enforce`.
- Cách xem PolicyReport và dọn đúng tài nguyên của từng lab.
