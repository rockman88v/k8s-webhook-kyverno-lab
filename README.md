# Kyverno Webhook Lab

Repository này cung cấp workshop thực hành Kyverno, gồm cài đặt Kyverno, Helm chart cho policy và các bài Lab Validate, Mutate, Generate.

## Bắt đầu

1. Cài Kyverno release `kyverno` vào namespace `kyverno` theo [kyverno-installation/README.md](kyverno-installation/README.md).
2. Đọc [use-cases/WORKSHOP_USE_CASES.md](use-cases/WORKSHOP_USE_CASES.md) để xem các policy được chart hỗ trợ.
3. Thực hành theo [use-cases/WORKSHOP_LAB.md](use-cases/WORKSHOP_LAB.md).
4. Tạo values override theo [use-cases/CUSTOMIZATION_GUIDE.md](use-cases/CUSTOMIZATION_GUIDE.md) hoặc [use-cases/QUICKSTART_CUSTOMIZATION.md](use-cases/QUICKSTART_CUSTOMIZATION.md).

## Nội dung

| Thành phần | Mô tả |
|---|---|
| [kyverno-installation/](kyverno-installation/) | Cài Kyverno bằng Helm repository hoặc chart `kyverno-3.9.0.tgz` đã tải sẵn |
| [use-cases/](use-cases/) | Helm chart policy và tài liệu workshop |
| [use-cases/templates-chart/](use-cases/templates-chart/) | Chart render policy Kyverno bằng Helm values |

## Policy được hỗ trợ

- Validate: không chạy root, image digest, resource requests/limits, registry allowlist, labels bắt buộc.
- Mutate: labels mặc định, restrictive security context, resource defaults, Prometheus annotations, Istio proxy mẫu.
- Generate: NetworkPolicy, clone registry Secret, ResourceQuota và LimitRange khi tạo Namespace.

Danh sách template, tên ClusterPolicy, selector và values key thực tế có trong [use-cases/WORKSHOP_USE_CASES.md](use-cases/WORKSHOP_USE_CASES.md).

## Cách áp dụng policy

Không apply trực tiếp file trong `use-cases/templates-chart/templates`; render Helm trước:

```bash
cd use-cases
helm template workshop-lab ./templates-chart \
  --values ./templates-chart/values-lab-1.yaml \
  --show-only templates/cpol-val-disallow-root-user.yaml \
  > policy.yaml

kubectl apply --dry-run=server -f policy.yaml
kubectl apply -f policy.yaml
```

## Yêu cầu

- Kubernetes đáp ứng yêu cầu của Kyverno chart (chart offline `3.9.0` yêu cầu Kubernetes `>=1.25`).
- `kubectl` đã cấu hình cluster.
- Helm 3 trở lên.
- Quyền quản trị cần thiết để tạo ClusterPolicy, RBAC và resource workshop.

## Dọn dẹp

Dùng cleanup trong từng Lab để xóa policy và resource test tương ứng. Khi hoàn tất workshop, gỡ Kyverno:

```bash
helm uninstall kyverno --namespace kyverno
kubectl delete namespace kyverno --ignore-not-found
```
