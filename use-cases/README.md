# Kyverno Workshop: Use Cases và Thực hành

Thư mục này chứa policy Helm chart và tài liệu thực hành Kyverno. Các policy được render từ `templates-chart` bằng Helm values trước khi apply vào cluster.

## Tài liệu

| Tài liệu | Mục đích |
|---|---|
| [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) | Danh sách use-case Validate, Mutate và Generate còn được chart hỗ trợ |
| [WORKSHOP_LAB.md](WORKSHOP_LAB.md) | Bài thực hành từng bước, lệnh render/apply và cleanup theo Lab |
| [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) | Tùy chỉnh policy qua Helm values |
| [QUICKSTART_CUSTOMIZATION.md](QUICKSTART_CUSTOMIZATION.md) | Quy trình ngắn để tạo values override và kiểm tra policy |
| [../kyverno-installation/README.md](../kyverno-installation/README.md) | Cài Kyverno release `kyverno` vào namespace `kyverno` |

## Bắt đầu

1. Cài Kyverno theo [hướng dẫn cài đặt](../kyverno-installation/README.md).
2. Đọc [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) để hiểu policy có trong chart.
3. Thực hiện [WORKSHOP_LAB.md](WORKSHOP_LAB.md).
4. Dùng [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) khi cần tạo policy values riêng.

## Cấu trúc

```text
use-cases/
├── README.md
├── WORKSHOP_USE_CASES.md
├── WORKSHOP_LAB.md
├── CUSTOMIZATION_GUIDE.md
├── QUICKSTART_CUSTOMIZATION.md
└── templates-chart/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-lab-1.yaml
    ├── values-lab-2.yaml
    ├── values-lab-3.yaml
    ├── values-custom.yaml
    └── templates/
```

## Render policy

Chạy từ thư mục `use-cases`. Không apply trực tiếp các file trong `templates-chart/templates` vì đó là Helm template.

```bash
helm template workshop-lab ./templates-chart \
  --values ./templates-chart/values-lab-1.yaml \
  --show-only templates/cpol-val-disallow-root-user.yaml \
  > policy.yaml

kubectl apply --dry-run=server -f policy.yaml
kubectl apply -f policy.yaml
```

Các file `values-lab-*.yaml` chỉ bật policy cần thiết cho từng Lab. Chỉ xóa policy và resource test do Lab tạo; không chạy `kubectl delete clusterpolicy --all` trên cluster dùng chung.
