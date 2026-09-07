# Bắt đầu nhanh: Tùy chỉnh Kyverno Policy

Policy trong `templates-chart` được cấu hình bằng Helm values. Bắt đầu từ file mẫu, render manifest để kiểm tra, rồi chỉ apply những template cần thiết.

## Bước 1: Tạo file values riêng

```bash
cd /home/vagrant/k8s-webhook-kyverno-lab/use-cases
cp templates-chart/values-custom.yaml values-myorg.yaml
vi values-myorg.yaml
```

File [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) mô tả toàn bộ values được hỗ trợ. Các file `values-lab-1.yaml`, `values-lab-2.yaml` và `values-lab-3.yaml` là ví dụ đã được chuẩn bị cho workshop.

## Bước 2: Chọn policy cần bật

Ví dụ chỉ bật security context và resource defaults cho namespace đã gắn label:

```yaml
policies:
  validate:
    enabled: false
  mutate:
    enabled: true
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
  generate:
    enabled: false

global:
  background: true
```

## Bước 3: Render policy

Không apply trực tiếp file trong `templates/`. Render bằng Helm trước:

```bash
helm template my-policies ./templates-chart \
  --values ./values-myorg.yaml \
  --show-only templates/cpol-mut-add-security-context.yaml \
  --show-only templates/cpol-mut-add-default-resources.yaml \
  > my-policies.yaml
```

Kiểm tra output, đặc biệt `metadata.name`, selector và resource values:

```bash
kubectl apply --dry-run=server -f my-policies.yaml
```

## Bước 4: Apply và xác minh

```bash
kubectl apply -f my-policies.yaml
kubectl get clusterpolicy
kubectl get namespace --show-labels
```

Gắn label để policy có selector được áp dụng:

```bash
kubectl label namespace dev auto-sec-context=true auto-add-requests=true --overwrite
```

Tạo Pod thử nghiệm:

```bash
cat <<'EOF' | kubectl apply -f - -n dev
apiVersion: v1
kind: Pod
metadata:
  name: custom-policy-test
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
EOF

kubectl get pod custom-policy-test -n dev -o yaml
```

## Các tình huống thông dụng

### Chỉ bật Validate security

```yaml
policies:
  validate:
    enabled: true
    disallowRoot:
      enabled: true
      failureAction: audit
      resourceKinds: [Pod]
      namespaceSelector:
        matchLabels:
          enforce-security: "true"
    requireImageDigest:
      enabled: false
    enforceResources:
      enabled: false
    restrictRegistries:
      enabled: true
      failureAction: enforce
      resourceKinds: [Pod]
      approvedRegistries: [registry.example.com]
      namespaceSelector:
        matchLabels:
          restrict-registries: "true"
    requireLabels:
      enabled: false
  mutate:
    enabled: false
  generate:
    enabled: false
```

Render bằng `--show-only templates/cpol-val-disallow-root-user.yaml` và `--show-only templates/cpol-val-restrict-registries.yaml`.

### Tạo default-deny NetworkPolicy cho Namespace mới

```yaml
policies:
  validate:
    enabled: false
  mutate:
    enabled: false
  generate:
    enabled: true
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

Sau khi apply policy, tạo Namespace cùng label trong một request:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: protected-team
  labels:
    network-policy: "enabled"
EOF
```

Không dùng `kubectl create namespace` rồi mới gắn label, vì Generate rule sẽ không được kích hoạt.

## Kiểm tra nhanh

```bash
helm template my-policies ./templates-chart --values ./values-myorg.yaml >/dev/null
kubectl get clusterpolicy
kubectl get policyreport -A
kubectl logs -n kyverno -l app.kubernetes.io/name=kyverno --tail=200
```

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| Policy không xuất hiện | `enabled` ở cấp nhóm hoặc policy là `false` | Kiểm tra file values và output `helm template` |
| Policy không match Pod | Namespace thiếu label hoặc kind không khớp | So sánh namespace labels với `namespaceSelector` |
| Generate không tạo resource | Label Namespace được thêm sau khi tạo | Tạo Namespace kèm label trong manifest EOF |
| Clone Secret bị từ chối | Background controller thiếu quyền Secret | Áp dụng RBAC tạm thời trong Lab 3 của [WORKSHOP_LAB.md](WORKSHOP_LAB.md) |
| NGINX crash với non-root | NGINX cần ghi cache/runtime directory | Dùng writable volume và NGINX config phù hợp, hoặc dùng BusyBox khi test policy |

## Dọn thử nghiệm

```bash
kubectl delete pod custom-policy-test -n dev --ignore-not-found
kubectl delete clusterpolicy add-security-context add-default-resources --ignore-not-found
kubectl label namespace dev auto-sec-context- auto-add-requests- --ignore-not-found
```

Chỉ xóa resource bạn đã tạo. Xem [WORKSHOP_LAB.md](WORKSHOP_LAB.md) để có cleanup đầy đủ theo từng Lab.
