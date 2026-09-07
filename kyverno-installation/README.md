# Cài đặt Kyverno

Tài liệu này hướng dẫn cài Kyverno với Helm release tên `kyverno`, cài trong namespace `kyverno` và sử dụng file cấu hình [values-kyverno-custom.yaml](values-kyverno-custom.yaml).

## Điều kiện tiên quyết

- Kubernetes cluster và `kubectl` đã được cấu hình.
- Helm 3 trở lên.
- Quyền tạo namespace, Helm release, CRD và các resource Kyverno cần thiết.

Tất cả lệnh dưới đây được chạy trong thư mục `kyverno-installation`.

```bash
cd /home/vagrant/k8s-webhook-kyverno-lab/kyverno-installation
```

## Cách 1: Cài từ Helm repository

Thêm và cập nhật Helm repository chính thức của Kyverno, sau đó cài chart với custom values:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm upgrade --install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --values values-kyverno-custom.yaml
```

Dùng cách này khi cluster có thể truy cập Helm repository và bạn muốn cài phiên bản chart mới nhất từ repository. Để cài đúng phiên bản `3.9.0`, thêm `--version 3.9.0` vào lệnh trên.

## Cách 2: Cài từ chart đã tải sẵn

Thư mục này đã có chart [kyverno-3.9.0.tgz](kyverno-3.9.0.tgz). Cách này phù hợp cho môi trường không truy cập Internet hoặc cần cài đúng phiên bản chart đã được kiểm tra. Giải nén chart trước; archive này tạo thư mục `kyverno/`.

```bash
tar -xzf kyverno-3.9.0.tgz

helm upgrade --install kyverno ./kyverno \
  --namespace kyverno \
  --create-namespace \
  --values values-kyverno-custom.yaml
```

## Kiểm tra sau cài đặt

```bash
helm status kyverno --namespace kyverno
kubectl get pods --namespace kyverno
kubectl get crd | grep kyverno
```

Chỉ tiếp tục các lab sau khi các pod Kyverno cần thiết đã ở trạng thái `Running` hoặc `Completed`.

## Nâng cấp cấu hình

Sau khi thay đổi `values-kyverno-custom.yaml`, chạy lại đúng nguồn chart đã chọn:

```bash
helm upgrade kyverno ./kyverno \
  --namespace kyverno \
  --values values-kyverno-custom.yaml
```

## Gỡ cài đặt

Lệnh sau gỡ Helm release `kyverno` trong namespace `kyverno`:

```bash
helm uninstall kyverno --namespace kyverno
kubectl delete namespace kyverno --ignore-not-found
```

Lệnh `helm uninstall` không tự xóa namespace; chỉ xóa namespace khi namespace này chỉ dùng cho Kyverno.
