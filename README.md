# Mini Lab: K8s (K3s), DRA & KServe cho AI Inference

## 1. Mô tả
Dự án này là một mini lab thực hành chuyên sâu nhằm kiểm chứng và ứng dụng các kiến thức về Container/GPU Orchestration. 

Mục tiêu cốt lõi của lab là giải quyết bài toán Resource Placement và Scaling cho AI Workload trong một môi trường bị giới hạn tài nguyên cục bộ trên laptop cá nhân (1 GPU 4GB VRAM). Lab sử dụng kiến trúc tối giản nhưng hiện đại:
* Hạ tầng: K3s (bản phân phối K8s nhẹ, phù hợp chạy local).
* Quản lý tài nguyên: Áp dụng tính năng Dynamic Resource Allocation (DRA) của Kubernetes để xin và chia sẻ tài nguyên linh hoạt, thay thế cho Device Plugin truyền thống.
* Model Serving: KServe cấu hình ở chế độ RawDeployment (loại bỏ các thành phần serverless như Knative/Istio) để tiết kiệm tối đa RAM hệ thống.

## 2. Setup
Hệ thống được thiết kế để chạy trực tiếp trên laptop cá nhân. Cấu hình phần cứng và phần mềm yêu cầu:
* CPU: >= 4 Cores.
* RAM hệ thống: >= 16GB (Quan trọng để tránh lỗi OOM khi load model weight).
* GPU: 1x NVIDIA GPU (4GB VRAM).
* OS: Ubuntu 22.04 LTS hoặc 24.04 LTS (Khuyên dùng Native Linux, hạn chế WSL2 để tránh lỗi mạng/driver).
* Công cụ: 
  - NVIDIA Container Toolkit & Drivers bản mới nhất.
  - kubectl, helm.
  - Cursor hoặc bất kỳ code editor nào để chỉnh sửa manifest.

## 3. Triển khai
Thực thi tuần tự các bước sau bằng command line tại local:

* Bước 1: Khởi tạo K3s & Kích hoạt DRA
  Cài đặt K3s cục bộ với flag kích hoạt tính năng: 
  --kube-apiserver-arg=feature-gates=DynamicResourceAllocation=true.

* Bước 2: Cài đặt NVIDIA DRA Driver
  Dùng helm hoặc kubectl apply để triển khai DRA Driver. Kiểm tra node để xác nhận Driver đã tự động publish các ResourceSlice mô tả thuộc tính của GPU 4GB.

* Bước 3: Cấu hình KServe Minimal
  Chỉ cài đặt KServe CRDs và Controller. Tùy chỉnh inferenceservice-config ConfigMap để thiết lập KServe chạy ở mode RawDeployment.

* Bước 4: Cấp phát tài nguyên động
  Tạo manifest định nghĩa DeviceClass bằng biểu thức CEL để phân loại thiết bị.
  Sử dụng ResourceClaimTemplate để Kubernetes tự động sinh ra các ResourceClaim xin cấp phát GPU cho từng Pod.

* Bước 5: Deploy Model & Expose API
  Sử dụng container chứa model siêu nhẹ (vd: Qwen-1.5-0.5B-Chat GGUF).
  Chạy lệnh `kubectl apply -f InferenceService.yaml` để deploy.
  Cấu hình Ingress định tuyến API endpoint về tên miền local (ví dụ: ai.ngtukien.id.vn) hoặc dùng NodePort/Port-forward để test trực tiếp.

## 4. Nhận xét
* Về tính năng DRA: Dynamic Resource Allocation tách biệt hoàn toàn vòng đời của thiết bị (ResourceClaim) ra khỏi Pod, giúp việc cấp phát tài nguyên AI linh hoạt và chủ động hơn nhiều so với Node Affinity hay Taints/Tolerations.
* Về tối ưu tài nguyên: KServe RawDeployment kết hợp với K3s là chiến lược phù hợp nhất cho môi trường laptop có 4GB VRAM. Nó giải quyết được bài toán hao phí tài nguyên nền ("High CPU/memory usage") thường gặp khi triển khai hệ thống AI.
* Tính ứng dụng: Mặc dù được thiết kế chạy trên 1 node (laptop), mô hình này tuân thủ nguyên tắc "Application-Infrastructure Separation" và có thể áp dụng thẳng lên môi trường thực tế nhiều Node/GPU mà không cần thay đổi kiến trúc cốt lõi.