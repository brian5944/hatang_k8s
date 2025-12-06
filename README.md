# Thiết kế cụm Kubernetes 4 Node – Không sử dụng MetalLB (Mặc định)

## 1. Mục tiêu tài liệu

- Áp dụng cho mô hình triển khai:
  
  **Internet → Cloudflare → Nginx Proxy Manager (NPM) → Kubernetes (4 node)**

- Giai đoạn đầu **KHÔNG sử dụng MetalLB** để tránh rủi ro về:
  - ARP bị chặn
  - IP–MAC binding
  - DHCP snooping
  - Policy hạn chế quảng bá IP ảo (VIP)

- Tài liệu cũng bao gồm **phân tích và lộ trình nâng cấp lên MetalLB** khi hạ tầng và policy của khách hàng cho phép.

---

## 2. Sơ đồ mô hình (Không sử dụng MetalLB)

```text
Internet
  |
  v
Cloudflare (DNS + TLS)
  |
  v
Nginx Proxy Manager (NPM)
  |
  |  HTTP/HTTPS
  v
Upstream (Round-robin)
  |-- 10.10.10.11:30080  (K8s Node 1)
  |-- 10.10.10.12:30080  (K8s Node 2)
  |-- 10.10.10.13:30080  (K8s Node 3)
  |-- 10.10.10.14:30080  (K8s Node 4)
                |
                v
          ingress-nginx (NodePort)
                |
                +--> Service / Pod: App 1
                +--> Service / Pod: App 2
```

> Trong mô hình này, **NPM đóng vai trò Load Balancer L7**, phân phối request tới nhiều node K8s thông qua **NodePort của ingress-nginx**.

---

## 3. Phân bố vai trò 4 Node Kubernetes

### 3.1. Mô hình khuyến nghị

- Node 1–3: `control-plane + worker`
- Node 4: `worker chính`

Điều này giúp:
- Đảm bảo **HA cho control-plane**
- Có worker riêng cho workload nặng, DB, queue, v.v.

### 3.2. Cấu hình tối thiểu mỗi VM (gợi ý)

| Thành phần | Tối thiểu | Khuyến nghị |
|------------|-----------|-------------|
| vCPU       | 4         | 6–8         |
| RAM        | 16 GB     | 24–32 GB    |
| Disk       | 100 GB    | 200 GB SSD  |
| OS         | Ubuntu 22.04 / 24.04 | |

---

## 4. Network & Port

### 4.1. IP

- 4 Node cùng 1 subnet LAN, ví dụ:
  - `10.10.10.11`
  - `10.10.10.12`
  - `10.10.10.13`
  - `10.10.10.14`

### 4.2. Port nội bộ K8s

| Port | Mục đích |
|------|----------|
| 6443 | Kubernetes API |
| 2379–2380 | etcd |
| 10250–10259 | kubelet, controller |
|

### 4.3. Port từ NPM → K8s

| Port | Mục đích |
|------|----------|
| 30080 | NodePort HTTP của ingress-nginx |
| 30443 | (Tuỳ chọn) NodePort HTTPS |

---

## 5. Cấu hình ingress-nginx (NodePort Mode)

```yaml
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/component: controller
```

- ingress-nginx deployment:
  - 2–3 replicas
  - Spread trên nhiều node

---

## 6. Cấu hình Nginx Proxy Manager

### 6.1. Upstream

| Host | Port |
|------|------|
| 10.10.10.11 | 30080 |
| 10.10.10.12 | 30080 |
| 10.10.10.13 | 30080 |
| 10.10.10.14 | 30080 |

### 6.2. Proxy Host (ví dụ)

- Domain: `app.customer.com`
- Forward scheme: `http`
- Forward IP: Danh sách node như trên
- Forward Port: `30080`

### 6.3. TLS

- TLS terminate tại:
  - Cloudflare
  - NPM (Let’s Encrypt hoặc cert nội bộ khách)
- Trong Kubernetes sử dụng **HTTP nội bộ**.

---

## 7. Ưu & nhược điểm phương án KHÔNG MetalLB

### ✅ Ưu điểm

- Không phụ thuộc ARP L2
- Không bị ảnh hưởng bởi:
  - IP–MAC binding
  - DHCP snooping
  - ARP inspection
- Phù hợp môi trường:
  - Hạ tầng khách hàng không rõ ràng
  - Network policy bảo mật cao
- Hoạt động ổn định ngay cả khi:
  - Hạ tầng chỉ cho phép L3 routing

### ❌ Nhược điểm

- NPM phải quản lý **danh sách node**
- Khi:
  - Thêm node
  - Thay IP node
  → phải update lại upstream trên NPM

---

## 7A. Storage cho Kubernetes – Assumption & Option (Bắt buộc xác nhận trước khi triển khai)

Do hệ thống Kubernetes sẽ triển khai các thành phần **stateful** như:

- Database (PostgreSQL, MySQL, MongoDB…)
- Object Storage (MinIO)
- Monitoring & Logging (Prometheus, Loki)
- CI/CD (Jenkins, ArgoCD…)

→ Do đó **bắt buộc phải có Persistent Storage dùng chung cho các node** hoặc một phương án thay thế tương đương.

Vì hiện tại **chưa nắm rõ hạ tầng storage của khách hàng**, tài liệu này đưa ra **các Option theo dạng giả định (Assumption)** để hai bên thống nhất trước khi triển khai.

---

### Option 1 – Khách hàng đã có NFS / NAS / SAN (Khuyến nghị cho UAT/Production)

**Yêu cầu từ khách hàng:**

- Cấp **1 endpoint storage dùng chung** (NFS hoặc iSCSI)
- Mount được từ **cả 4 node Kubernetes**
- Dung lượng đề xuất: **tối thiểu 500 GB – 2 TB** (tuỳ quy mô hệ thống)
- Độ ổn định: uptime cao, network ổn định

**Cách sử dụng trong Kubernetes:**

- Tạo `StorageClass` dựa trên NFS / iSCSI
- Tất cả các `PersistentVolumeClaim` (PVC) của DB, MinIO, Monitoring sẽ dùng `StorageClass` này

✅ Ưu điểm:

- Dữ liệu tách rời khỏi node
- Pod reschedule không mất dữ liệu
- Dễ sao lưu (backup), dễ khôi phục
- Phù hợp môi trường UAT / Production

❗ Lưu ý:

- Storage trở thành **Single Point of Failure** nếu không có RAID/HA phía storage
- Network giữa K8s ↔ Storage phải ổn định

---

### Option 2 – Không có storage dùng chung (Local Storage Only – Chỉ dành cho DEV/LAB)

**Đặc điểm:**

- Mỗi Pod sử dụng disk local của node (Local PV)
- Có thể dùng:
  - `local-path-provisioner`
  - hoặc Local Persistent Volume

⚠️ Nhược điểm nghiêm trọng:

- Pod restart / reschedule → **mất dữ liệu**
- Node chết → **mất toàn bộ volume**
- Không phù hợp cho:
  - Database
  - MinIO
  - Logging
  - CI/CD

❗ Chỉ khuyến nghị cho:

- DEV
- LAB
- Test ngắn hạn

---

### Option 3 – Dùng Distributed Storage trong K8s (Longhorn / Rook-Ceph)

**Đặc điểm:**

- Storage dựng trực tiếp trên 4 node
- Dữ liệu được nhân bản (replica) giữa các node

✅ Ưu điểm:

- Không cần NAS/SAN bên ngoài
- Tự HA trong cluster
- Pod migrate không mất dữ liệu

❌ Nhược điểm:

- Tốn nhiều:
  - CPU
  - RAM
  - Disk
- Vận hành phức tạp hơn
- Yêu cầu:
  - SSD chất lượng tốt
  - Network nội bộ ổn định, độ trễ thấp

---

### Kết luận về Storage

- Với hệ thống khách hàng:
  - ✅ **Khuyến nghị Option 1 – NFS/NAS/SAN có sẵn**
- Trong trường hợp khách **chưa có storage dùng chung**:
  - Có thể tạm dùng **Option 2** cho DEV
- Nếu khách muốn **all-in trong Kubernetes, không phụ thuộc storage ngoài**:
  - Áp dụng **Option 3**, kèm đánh giá lại tài nguyên hạ tầng

---

# PHẦN 2 – MetalLB & Lộ trình nâng cấp

## 8. Khi nào nên cân nhắc dùng MetalLB?

Khi khách hàng xác nhận:

- Cho phép dùng **VIP dưới dạng IP ảo trong LAN**
- Không bật các policy chặn:
  - Gratuitous ARP
  - L2 VIP broadcasting
- Có thể cấp **1 dải IP LAN riêng** cho Kubernetes VIP

---

## 9. Mô hình khi nâng cấp lên MetalLB

```text
Internet
  |
  v
Cloudflare (DNS + TLS)
  |
  v
Nginx Proxy Manager (NPM)
  |
  |  HTTP/HTTPS
  v
MetalLB VIP (vd: 10.10.10.51)
  |
  v
ingress-nginx (Service type: LoadBalancer)
  |
  +--> Service / Pod: App 1
  +--> Service / Pod: App 2
```

- ingress-nginx chuyển sang:
  - `Service type: LoadBalancer`
- MetalLB cấp IP ví dụ:
  - `10.10.10.51`

NPM lúc này chỉ cần trỏ tới **1 IP duy nhất**.

---

## 10. Resource bổ sung khi dùng MetalLB

### 10.1. Dải IP VIP

Ví dụ:

- Node IP: `10.10.10.11–14`
- MetalLB pool: `10.10.10.50–10.10.10.60`

### 10.2. Cấu hình MetalLB

- Mode: `Layer2`
- IPAddressPool + L2Advertisement

### 10.3. Thay đổi Service ingress-nginx

```yaml
spec:
  type: LoadBalancer
```

---

## 11. Ưu & nhược điểm khi dùng MetalLB

### ✅ Ưu điểm

- NPM chỉ cần **1 IP duy nhất**
- Không cần quản lý danh sách node thủ công
- VIP tự failover giữa các node
- Kiến trúc gọn, dễ vận hành, dễ scale

### ❌ Nhược điểm

- Phụ thuộc **Layer 2 & ARP trong LAN khách hàng**
- Có thể bị chặn nếu:
  - Network security quá chặt
  - Switch bật IP–MAC binding
- Khó troubleshoot nếu team network không hỗ trợ tốt

---

## 12. Lộ trình nâng cấp đề xuất

### Giai đoạn 1 – An toàn (Mặc định)

- Dùng:
  - NodePort + NPM LB
- Không dùng VIP
- Phù hợp triển khai nhanh, bàn giao sớm

### Giai đoạn 2 – Tối ưu (Tuỳ chọn)

Khi khách đồng ý về network:

1. Cấp dải IP VIP
2. Cài MetalLB
3. Chuyển ingress-nginx từ NodePort → LoadBalancer
4. Update NPM:
   - Từ nhiều upstream node
   - Sang 1 VIP duy nhất

---

## 13. Kết luận

- Trong bối cảnh **không kiểm soát được hạ tầng khách**, phương án **KHÔNG dùng MetalLB là lựa chọn an toàn và thực tế nhất**.
- MetalLB chỉ nên kích hoạt khi:
  - Network team xác nhận cho phép VIP
  - Có dải IP LAN riêng
- Thiết kế hiện tại đã sẵn sàng để **nâng cấp mềm sang MetalLB bất cứ lúc nào mà không ảnh hưởng application**.

---

**Tài liệu phục vụ cho khâu: kế hoạch triển khai, proposal kỹ thuật, và bàn giao vận hành.**
