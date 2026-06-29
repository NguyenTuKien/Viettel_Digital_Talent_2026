# Mini Lab: K8s (K3s), DRA & KServe cho AI Inference

## 1. Mô tả
Dự án này là một mini lab thực hành chuyên sâu nhằm kiểm chứng và ứng dụng các kiến thức về Container/GPU Orchestration. 

Mục tiêu cốt lõi của lab là giải quyết bài toán Resource Placement và Scaling cho AI Workload trong một môi trường bị giới hạn tài nguyên cục bộ trên laptop cá nhân (1 GPU 4GB VRAM). Lab sử dụng kiến trúc tối giản nhưng hiện đại:
* Hạ tầng: K3s (bản phân phối K8s nhẹ, phù hợp chạy local).
* Quản lý tài nguyên: Áp dụng tính năng Dynamic Resource Allocation (DRA) của Kubernetes để xin và chia sẻ tài nguyên linh hoạt, thay thế cho Device Plugin truyền thống.
* Model Serving: KServe cấu hình ở chế độ RawDeployment (loại bỏ các thành phần serverless như Knative/Istio) để tiết kiệm tối đa RAM hệ thống.

## 2. Setup
### 2.1. Cấu hình phần cứng & phần mềm
Hệ thống được thiết kế để chạy trực tiếp trên laptop cá nhân. Cấu hình phần cứng và phần mềm yêu cầu:
* CPU: >= 4 Cores.
* RAM hệ thống: >= 16GB (Quan trọng để tránh lỗi OOM khi load model weight).
* GPU: 1x NVIDIA GPU (4GB VRAM).
* OS: Ubuntu 22.04 LTS hoặc 24.04 LTS (Khuyên dùng Native Linux, hạn chế WSL2 để tránh lỗi mạng/driver).
* Công cụ: 
  - NVIDIA Container Toolkit & Drivers bản mới nhất.
  - kubectl, helm.
  - Cursor hoặc bất kỳ code editor nào để chỉnh sửa manifest.

### 2.2. Tính toán
* Số tham số mô hình (Model Parameters): 0.5 tỷ * 2 byte mỗi tham số = 1GB VRAM.
* Bộ kích hoạt và bộ nhớ đệm KV (Activation & KV Cache): 
    * Với Batch size = 1 và Sequence length = 2048
* Qwen 1.5-0.5B-Chat GGUF (Quantized 4-bit):
    * L (Số lớp) = 24
    * N_kv (Số Key/Value Heads) = 16 (Vì model này dùng Multi-Head Attention, số KV heads bằng số Q heads, không phải GQA)
    * D_kv (Kích thước mỗi head) = 64 (vì hidden_size = 1024 / 16 = 64)
    * Kiểu dữ liệu KV Cache = FP16 → C_B = 2 bytes

&rArr; Kích thước KV Cache = L * N_kv * D_kv * C_B * Sequence length = 2 * 24 * 16 * 64 * 2 * 2048 = 192.0 MB.
* Các activation khác (ngoài KV Cache) có thể tiêu tốn thêm tài nguyên. (ước tính 1GB)
* Phần ngữ cảnh (Overhead) cho KServe, DRA Driver, và các Pod khác: ~1GB.
* Hệ số X, Y, Z (dùng để ước tính tổng VRAM thực tế cần thiết):
    * X = 5–20% (0.05–0.2): Overhead activation trung gian — bộ nhớ tạm thời llama.cpp cần "nháp" trong mỗi lần tính toán (attention scores, MLP intermediate outputs), ngoài KV Cache. Model càng nhỏ thì X càng nhỏ.
    * Y = 10–30% (0.1–0.3): Overhead hệ thống — bộ nhớ CUDA runtime, memory allocator và memory fragmentation của framework (llama.cpp, KServe).
    * Z = 5% (0.05): VRAM bị GPU driver giữ lại — phần VRAM vật lý không thể dùng cho model (GPU 4GB thực tế chỉ còn ~3.8GB khả dụng).
* Tổng ước tính VRAM thực tế cần (theo công thức):
    * `(Model + KV + Activations) × (1+X)  +  Overhead × (1+Y)  +  GPU_total × Z`
    * `= (1 + 0.192 + ~0.1) GB × (1 + 0.05)  +  1 GB × (1 + 0.10)  +  4 GB × 0.05`
    * `= 1.292 × 1.05  +  1.0 × 1.10  +  0.20`
    * `≈ 1.36 + 1.10 + 0.20 ≈ **2.66 GB** (X=5%, Y=10% – trường hợp tối ưu)`
    * Dải đầy đủ (X: 5–20%, Y: 10–30%): **2.66 – 3.05 GB < 4 GB GPU**

## 3. Triển khai
Thực thi tuần tự các bước sau bằng command line tại local:

### Bước 1: Khởi tạo K3s & Kích hoạt DRA
* Cài đặt K3s cục bộ với flag kích hoạt tính năng: 
    ```bash
    curl -sfL https://get.k3s.io | sh -s - --kube-apiserver-arg=feature-gates=DynamicResourceAllocation=true
    ```
* Lấy kubeconfig từ `/etc/rancher/k3s/k3s.yaml` và export biến môi trường:
    ```bash
    # 0. Xóa biến config cũ
    unset KUBECONFIG
    # 1. Tạo thư mục .kube nếu chưa có
    mkdir -p ~/.kube
    # 2. Copy file cấu hình của K3s vào thư mục của bạn (cần sudo để copy file của root)
    sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
    # 3. Đổi quyền sở hữu file config vừa copy sang user của bạn
    sudo chown $(id -u):$(id -g) ~/.kube/config
    ```
* Kết quả: 
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get nodes
    NAME       STATUS   ROLES           AGE   VERSION
    ngtukien   Ready    control-plane   10m   v1.36.2+k3s1
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl api-resources | grep resource.k8s.io
    deviceclasses                                    resource.k8s.io/v1                false        DeviceClass
    resourceclaims                                   resource.k8s.io/v1                true         ResourceClaim
    resourceclaimtemplates                           resource.k8s.io/v1                true         ResourceClaimTemplate
    resourceslices                                   resource.k8s.io/v1                false        ResourceSlice
    ```
### Bước 2: Cài đặt NVIDIA DRA Driver
* Cài đặt `helm`: 
    ```bash
    curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```
* Thêm kho lưu trữ chính thức của NVIDIA (Helm) và cập nhật:
    ```bash
    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
    helm repo update
    ```
* (Tùy chọn) Kiểm tra chart DRA Driver trên kho NVIDIA:
    ```bash
    helm search repo nvidia/nvidia-dra-driver-gpu
    ```
   **Kết quả:** 
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ helm search repo nvidia/nvidia-dra-driver-gpu
    NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
    nvidia/nvidia-dra-driver-gpu    25.12.0         25.12.0         Official Helm chart for the NVIDIA DRA Driver f...
    ```
* Dùng `helm` để triển khai DRA Driver. 
    ```bash
    helm install nvidia-dra nvidia/nvidia-dra-driver-gpu \
    --namespace nvidia-dra \
    --create-namespace \
    --set kubeletPlugin.cdiRoot=/var/run/cdi \
    --set gpuResourcesEnabledOverride=true
    ```
* Đánh nhãn node để K3s nhận diện GPU:
    ```bash
    kubectl label node <tên node> nvidia.com/gpu.present=true
    ```
* Kiểm tra node để xác nhận Driver đã tự động publish các ResourceSlice mô tả thuộc tính của GPU 4GB.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get pods -n nvidia-dra
    NAME                                                READY   STATUS    RESTARTS   AGE
    nvidia-dra-driver-gpu-controller-6f867b6566-2qn77   1/1     Running   0          16m
    nvidia-dra-driver-gpu-kubelet-plugin-2tvnh          2/2     Running   0          11m
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get resourceslices
    NAME                                       NODE       DRIVER                      POOL       AGE
    ngtukien-compute-domain.nvidia.com-cwc7f   ngtukien   compute-domain.nvidia.com   ngtukien   11m
    ngtukien-gpu.nvidia.com-56wm7              ngtukien   gpu.nvidia.com              ngtukien   11m
    ```
### Bước 3: Cấu hình KServe Minimal
* Thêm `cert-manager`:
    ```bash
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.yaml
    ```
* Chỉ cài đặt KServe CRDs và Controller. 
    ```bash
    kubectl apply --server-side -f https://github.com/kserve/kserve/releases/download/v0.19.0/kserve.yaml
    ```
* Tùy chỉnh inferenceservice-config ConfigMap để thiết lập KServe chạy ở mode RawDeployment.
    ```bash
    kubectl patch configmap inferenceservice-config -n kserve --type=merge -p '{"data":{"deploy":"{\"defaultDeploymentMode\":\"RawDeployment\"}"}}'
    ```
* Kết quả:
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get pods -n kserve
    NAME                                                    READY   STATUS    RESTARTS   AGE
    kserve-controller-manager-8f5c5d979-665db               2/2     Running   0          2m45s
    kserve-localmodel-controller-manager-57cc7cb6db-vj4bb   1/1     Running   0          2m45s
    llmisvc-controller-manager-77cc4bc547-47tnq             1/1     Running   0          2m45s
    ```
### Bước 4: Cấp phát tài nguyên động
* Tạo [manifest](./manifest/01-dra/deviceclass.yaml) định nghĩa `DeviceClass` bằng biểu thức CEL để lựa chọn các thiết bị GPU do NVIDIA DRA Driver công bố.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get deviceclasses
    NAME                                        AGE
    compute-domain-daemon.nvidia.com            80m
    compute-domain-default-channel.nvidia.com   80m
    gpu.nvidia.com                              80m
    mig.nvidia.com                              80m
    nvidia-gpu                                  68s
    vfio.gpu.nvidia.com                         80m
    ```
* Sử dụng `ResourceClaimTemplate` [manifest](./manifest/01-dra/resourceclaimtemplate.yaml) để Kubernetes tự động sinh ra các `ResourceClaim` xin cấp phát GPU cho từng Pod.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get resourceclaimtemplates
    NAME           AGE
    gpu-template   8m54s
    ```
* **Lưu ý:** Trong kiến trúc DRA kết hợp KServe này, ta không cần tự tạo `ResourceClaim` thủ công. Thay vào đó, KServe (thông qua InferenceService) sẽ tự động gọi `ResourceClaimTemplate` để sinh ra các `ResourceClaim` động gắn liền với vòng đời của từng Pod. Bạn có thể kiểm chứng bằng lệnh:
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get resourceclaims
    NAME                                              STATE                AGE
    llama-qwen-predictor-57685c8575-h5bbx-gpu-5bfmc   allocated,reserved   85m
    ```
### Bước 5: Deploy Model & Expose API
* Tạo [manifest](./manifest/02-storage/persistentvolume.yaml) định nghĩa `PersistentVolume` (hostPath) để mount trực tiếp thư mục chứa model trên ổ cứng laptop vào container.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get pv
    NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
    llama-models-pv   5Gi        RWO            Retain           Bound    default/llama-models                  <unset> 
    ```
* Tạo [manifest](./manifest/02-storage/persistentvolumeclaim.yaml) định nghĩa `PersistentVolumeClaim` để xin cấp phát Storage (PV) vừa tạo.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get pvc
    NAME           STATUS   VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
    llama-models   Bound    llama-models-pv   5Gi        RWO                           <unset>                 34s
    ```
* Sử dụng container chứa model siêu nhẹ (vd: Qwen-1.5-0.5B-Chat GGUF).
    ```bash
    wget https://huggingface.co/QwenLM/Qwen-1.5-0.5B-Chat-GGUF/resolve/main/qwen1.5-0.5b-chat.Q4_K_M.gguf
    ```
* Tạo [manifest](./manifest/03-kserve/servingruntime.yaml) định nghĩa `ServingRuntime` để cấu hình runtime AI. `ServingRuntime` sử dụng image phục vụ suy luận (ví dụ: `llama.cpp`) và yêu cầu tài nguyên GPU thông qua danh sách resource claims.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get servingruntime
    NAME                DISABLED   MODELTYPE   CONTAINERS         AGE
    llama-cpp-runtime              gguf        kserve-container   27m
    ```
* Tạo [manifest](./manifest/03-kserve/inferenceservice.yaml) định nghĩa `InferenceService` để triển khai mô hình AI. KServe InferenceService kết nối `ServingRuntime`, Model Storage và tự động sinh `ResourceClaim` từ `ResourceClaimTemplate` để lấy GPU.
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get inferenceservice
    NAME         URL                                     READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
    llama-qwen   http://llama-qwen-default.example.com   True                                                                  27m
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get pods
    NAME                                    READY   STATUS    RESTARTS      AGE
    llama-qwen-predictor-57685c8575-h5bbx   1/1     Running   9 (14m ago)   30m
    ```
### Bước 6: Export API và Test
* Tạo [manifest](./manifest/04-network/cloudflare.yaml) cho pod Cloudflare tunnel để export API ra Internet (Lưu ý: cấu hình token bảo mật nằm trong file `./manifest/04-network/secret.yaml`).
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get pods
    NAME                                    READY   STATUS    RESTARTS      AGE
    cloudflared-845d7c844c-qj9zn            1/1     Running   0             4s
    llama-qwen-predictor-57685c8575-h5bbx   1/1     Running   9 (21m ago)   37m
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ kubectl get svc -l serving.kserve.io/inferenceservice=llama-qwen
    NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    llama-qwen-predictor   ClusterIP   10.43.129.55   <none>        80/TCP    51m
    ```
* Cấu hình Cloudflare:
    * Service Type: `HTTP`
    * URL: `llama-qwen-predictor.default.svc.cluster.local:80`
* Kết quả:
![Output](image.png)
* Kiểm tra tình trạng GPU hiện tại:
    ```bash
    ngtukien@NgTuKien:~/Documents/VDT_2026/15.Report$ nvidia-smi
    Sun Jun 28 09:58:59 2026       
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 580.142                Driver Version: 580.142        CUDA Version: 13.0     |
    +-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  NVIDIA GeForce RTX 3050 ...    Off |   00000000:01:00.0 Off |                  N/A |
    | N/A   51C    P8              9W /   60W |    2757MiB /   4096MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+

    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |    0   N/A  N/A            1671      G   /usr/lib/xorg/Xorg                        4MiB |
    |    0   N/A  N/A           53460      C   /app/llama-server                      2734MiB |
    +-----------------------------------------------------------------------------------------+
    ```
## 4. Nhận xét
### Dynamic Resource Allocation
* Các thành phần:
    * **ResourceDriver (DRA Driver)**: Plugin thực hiện việc giao tiếp giữa Kubernetes và phần cứng (GPU, FPGA, v.v.), chịu trách nhiệm cấp phát và thu hồi tài nguyên.
    * **ResourceSlice**: Định nghĩa một "lát cắt" (slice) tài nguyên khả dụng trên một node. Ví dụ: một GPU có thể được chia thành nhiều "slice" nhỏ hơn.
    * **DeviceClass**: Định nghĩa loại tài nguyên (ở đây là GPU NVIDIA) bằng các selector dựa trên nhãn (labels) hoặc thông tin từ `kubelet`.
    * **ResourceClaimTemplate**: Mẫu (template) để sinh ra `ResourceClaim` khi cần. Khi KServe yêu cầu tài nguyên, DRA sẽ dùng template này để tạo ra một `ResourceClaim` cụ thể.
    * **ResourceClaim**: Thực tế xin cấp phát tài nguyên cho một Pod cụ thể.
    