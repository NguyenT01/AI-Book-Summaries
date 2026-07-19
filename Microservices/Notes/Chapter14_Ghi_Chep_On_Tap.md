# Chapter 14 — Container Hosting and Development Patterns: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| Container | Môi trường cô lập, nhẹ, chạy một ứng dụng cùng toàn bộ dependency, chia sẻ chung kernel với host |
| Containerization | Quá trình đóng gói ứng dụng + dependency thành gói portable, tái lập được, gọi là image |
| Image | Bản thiết kế (blueprint) tĩnh, bất biến, định nghĩa filesystem/config/dependency để tạo container |
| Docker | Công cụ containerization phổ biến nhất, kiến trúc client-server qua daemon `dockerd` |
| Docker daemon (`dockerd`) | Tiến trình nền quản lý vòng đời container, build/lưu image, network, volume |
| OCI (Open Container Initiative) | Chuẩn định nghĩa format image/runtime thống nhất, đảm bảo tương thích giữa Docker, Podman, containerd, CRI-O |
| Base image / Parent image | Image nền tảng làm điểm khởi đầu để build image mới (vd: `aspnet:10.0-alpine`) |
| `scratch` image | Base image trống hoàn toàn, dùng cho binary compile tĩnh (Go, .NET AOT) |
| Layer | Lớp thay đổi chồng lên nhau tạo thành image, mỗi chỉ thị Dockerfile thường tạo một layer |
| Immutability (bất biến) | Đặc tính image không thể sửa sau khi tạo — muốn đổi phải build lại |
| Dockerfile | File text mô tả từng bước build image (base image, lệnh cài đặt, copy asset, entrypoint) |
| Multi-stage build | Kỹ thuật tách stage build (nặng, có SDK) khỏi stage final (nhẹ, chỉ runtime) trong cùng Dockerfile |
| `.dockerignore` | File liệt kê thư mục/file không đưa vào image khi build |
| Container registry | Kho lưu trữ trung tâm cho image có version (Docker Hub, ACR, ECR, Harbor...) |
| Docker Compose | Công cụ định nghĩa và điều phối multi-container qua file YAML, dùng cho local/dev |
| `docker-compose.override.yml` | File ghi đè/mở rộng cấu hình theo môi trường lên trên `docker-compose.yml` gốc |
| `depends_on` | Chỉ thị Compose khai báo thứ tự khởi động container (không đảm bảo app-ready) |
| Container orchestration | Cách tiếp cận tự động hóa khởi chạy, quản lý vòng đời, scaling nhiều container |
| Kubernetes (K8s) | Nền tảng orchestration chuẩn công nghiệp cho production-scale, do CNCF duy trì |
| Cluster | Tập hợp node (worker machine) được quản lý bởi control plane trong K8s |
| Pod | Đơn vị triển khai nhỏ nhất trong K8s, bọc một/nhiều container chia sẻ network + storage |
| Deployment (K8s) | Controller quản lý Pod, đảm bảo số replica mong muốn, hỗ trợ rolling update |
| Service (K8s) | Lớp trừu tượng expose tập Pod thành network endpoint ổn định |
| Ingress (K8s) | Tập luật định tuyến traffic từ bên ngoài vào các service trong cluster |
| NodePort | Loại Service expose qua port tĩnh (30000–32767) trên host, dùng nhiều cho local dev |
| `kubectl` | CLI để tương tác với Kubernetes cluster |
| Minikube | Công cụ chạy Kubernetes cluster local, thường dùng trên Linux/macOS |
| Microsoft.NET.Build.Containers | Package cho phép publish .NET app thành container mà không cần Dockerfile |

---

## 2. So sánh Virtual Machine vs Container

| Tiêu chí | Virtual Machine | Container |
|---|---|---|
| Mức cô lập | Cao — mỗi VM có hệ điều hành riêng đầy đủ | Trung bình — chia sẻ chung kernel host |
| Kích thước | Nặng (GB), nhân bản toàn bộ OS | Nhẹ (MB), chỉ đóng gói app + dependency |
| Thời gian khởi động | Chậm (phút) | Nhanh (giây hoặc dưới giây) |
| Mật độ trên 1 host | Thấp | Cao |
| Bảo mật | Cách ly mạnh hơn, phù hợp multi-tenant nghiêm ngặt | Cách ly yếu hơn do chung kernel, cần thêm kiểm soát runtime |
| Công cụ điển hình | VMware, VirtualBox, Hyper-V | Docker, Podman, containerd |

## 3. So sánh Docker Compose vs Kubernetes

| Tiêu chí | Docker Compose | Kubernetes |
|---|---|---|
| Mục tiêu sử dụng | Local dev, orchestration quy mô nhỏ | Production-scale, hệ thống phân tán lớn |
| Mô hình cấu hình | YAML đơn giản, 1 file | Declarative API, nhiều manifest (Deployment, Service, Ingress...) |
| Self-healing | Không có sẵn | Có — tự restart/reschedule Pod khi lỗi |
| Auto-scaling | Không hỗ trợ native | Có (horizontal + vertical) |
| Service discovery/load balancing | Hạn chế | Tích hợp sẵn |
| Độ phức tạp vận hành | Thấp, dễ học | Cao, cần kinh nghiệm/đội DevOps chuyên trách |
| Khi nào dùng | Dev/test, hệ thống nhỏ, team ít kinh nghiệm hạ tầng | Hệ thống lớn cần resilience/scaling tự động, multi-cloud |

## 4. Cấu trúc Dockerfile multi-stage (giải thích từng stage)

| Stage | Base image dùng | Mục đích | Có trong image cuối? |
|---|---|---|---|
| `base` | `aspnet:...-alpine` (chỉ runtime) | Nền tối giản cho image production | Có (làm gốc cho `final`) |
| `build` | `sdk:...-alpine` (đầy đủ SDK) | Restore + build source code | Không |
| `publish` | Kế thừa `build` | Chạy `dotnet publish` tạo output tối ưu | Không |
| `final` | Kế thừa `base` | Copy đúng output đã publish, set ENTRYPOINT | Có — đây là image thực sự chạy production |

## 5. Vòng đời image → registry → orchestration

| Bước | Lệnh/hành động | Ghi chú |
|---|---|---|
| 1. Build image | `docker build -t <name>:<tag> .` | Nên pin version, tránh `latest` trong production |
| 2. Test local | `docker run ...` | Kiểm tra container chạy đúng trước khi publish |
| 3. Tag cho registry | `docker tag <local> <registry>/<name>:<tag>` | Chuẩn bị đường dẫn đầy đủ tới registry đích |
| 4. Push lên registry | `docker push <registry>/<name>:<tag>` | Cần đăng nhập registry trước (`docker login`) |
| 5. Khai báo trong Compose/K8s | image reference trong YAML | Compose dùng `image:`, K8s dùng `spec.containers[].image` |
| 6. Triển khai | `docker-compose up` hoặc `kubectl apply -f ...` | K8s cần image đã ở registry cluster truy cập được |

---

## Checklist kỹ thuật khi triển khai

- [ ] Dockerfile dùng multi-stage build, tách rõ stage build (SDK) khỏi stage final (runtime)
- [ ] Base image pin version cụ thể, không dùng `latest` cho production
- [ ] Chọn distro (Alpine/Debian) phù hợp nhu cầu ứng dụng, không mặc định chọn nhẹ nhất mù quáng
- [ ] `.dockerignore` loại bỏ `bin`, `obj`, `.git`, `node_modules`, file secret khỏi build context
- [ ] Không hardcode secret/password trong Dockerfile — dùng biến môi trường/volume/K8s Secrets
- [ ] Image đã được scan lỗ hổng (Trivy/Anchore) trước khi push lên registry production
- [ ] `docker-compose.yml` khai báo `depends_on` đúng thứ tự dependency giữa các service
- [ ] Có cơ chế retry/health check ở tầng ứng dụng, không chỉ dựa vào `depends_on` để đảm bảo service đã sẵn sàng
- [ ] Registry đã cấu hình access control phù hợp (private cho nội bộ, public chỉ cho image không nhạy cảm)
- [ ] Nếu dùng Kubernetes: đã có Deployment + Service manifest, replicas hợp lý, image đã push lên registry cluster truy cập được
- [ ] Đã cân nhắc kỹ trước khi chọn Kubernetes: quy mô hệ thống, đội ngũ vận hành, chi phí học tập có tương xứng không

---

## Anti-patterns / Lưu ý cần tránh

- **Dùng image `latest` trong production** → phá vỡ tính tái lập, deploy hôm nay và ngày mai có thể ra kết quả khác nhau dù không đổi code.
- **Build image chỉ với 1 stage duy nhất** (không multi-stage) → image production mang theo cả SDK/compiler không cần thiết, phình to kích thước, tăng surface tấn công.
- **Đưa secret/password vào Dockerfile hoặc COPY thẳng vào image** → image có thể bị chia sẻ/leak, secret lộ vĩnh viễn trong lịch sử layer.
- **Dùng `docker commit` làm quy trình build chính thức cho image chia sẻ team** → không tái lập được, không review được, dễ tạo ra "hộp đen" không ai biết chính xác image chứa gì.
- **Nhảy thẳng sang Kubernetes vì "ai cũng dùng"** mà chưa có nhu cầu thực sự về scaling/self-healing → gánh chi phí vận hành khổng lồ không tương xứng lợi ích.
- **Chỉ dựa vào `depends_on` để đảm bảo service sẵn sàng** → `depends_on` chỉ đảm bảo thứ tự *khởi động container*, không đảm bảo ứng dụng bên trong đã sẵn sàng nhận request — vẫn cần retry/health check ở tầng gọi.
- **Bỏ qua observability khi container hóa** → container ephemeral, log/metric không tập trung sẽ khiến việc debug production gần như bất khả thi.
- **Multi-tenant chạy chung host mà không tăng cường kiểm soát runtime** (AppArmor, seccomp, network policy) → rủi ro bảo mật do container chia sẻ chung kernel.

---

## Từ vựng chuyên ngành (Anh - Việt)

| Thuật ngữ | Nghĩa |
|---|---|
| Container | Container — môi trường cô lập chạy ứng dụng |
| Image | Ảnh/gói image — bản thiết kế bất biến để tạo container |
| Containerization | Container hóa |
| Configuration drift | Sự lệch cấu hình dần theo thời gian giữa các môi trường |
| Immutability | Tính bất biến |
| Base image / Parent image | Image nền/image gốc |
| Layer | Lớp (trong cấu trúc image) |
| Multi-stage build | Build nhiều giai đoạn |
| Registry | Kho lưu trữ image |
| Orchestration | Điều phối (nhiều container/service) |
| Self-healing | Tự phục hồi |
| Rolling update | Cập nhật cuốn chiếu (không downtime) |
| Replica | Bản sao (instance) của Pod/service |
| Node | Máy worker trong cluster |
| Cluster | Cụm máy được quản lý tập trung |
| Ephemeral | Tồn tại ngắn hạn, không bền vững |
| Declarative configuration | Cấu hình khai báo (mô tả trạng thái mong muốn, không mô tả từng bước) |
| Vulnerability scanning | Quét lỗ hổng bảo mật |
| Lateral movement | Di chuyển ngang (kỹ thuật tấn công lan từ service này sang service khác) |

---

## Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Container khác máy ảo (VM) ở điểm cốt lõi nào?**
→ Container chia sẻ chung kernel của host, chỉ đóng gói ứng dụng + dependency cần thiết, nên nhẹ và khởi động nhanh hơn nhiều. VM thì nhân bản toàn bộ hệ điều hành riêng cho từng instance, cách ly mạnh hơn nhưng nặng nề và tốn tài nguyên hơn hẳn.

**Q2: Tại sao image container được thiết kế bất biến (immutable)?**
→ Để đảm bảo môi trường dev, staging, production dùng chính xác cùng một artifact — loại bỏ configuration drift và câu chuyện "chạy trên máy tôi thì được". Muốn thay đổi phải build lại image mới, không sửa trực tiếp image cũ.

**Q3: Multi-stage build trong Dockerfile giải quyết vấn đề gì?**
→ Tách giai đoạn build (cần SDK/compiler, nặng) khỏi giai đoạn chạy thực tế (chỉ cần runtime, nhẹ). Image cuối cùng chỉ copy đúng phần đã publish từ stage build, không mang theo toàn bộ SDK — giảm đáng kể kích thước và bề mặt tấn công của image production.

**Q4: Khi nào nên dùng Docker Compose, khi nào nên nhảy sang Kubernetes?**
→ Docker Compose phù hợp cho local dev, test, hoặc hệ thống nhỏ với đội ngũ chưa có kinh nghiệm vận hành hạ tầng phức tạp. Kubernetes nên dùng khi thực sự cần self-healing, auto-scaling, multi-cloud, và hệ thống đủ lớn để việc đầu tư vào đội DevOps/platform engineering là xứng đáng. Kubernetes không phải mặc định — nó là một điểm đến, không phải điểm khởi đầu.

**Q5: Service trong Kubernetes giải quyết vấn đề gì mà Pod không tự làm được?**
→ Pod có bản chất ephemeral, có thể bị restart với IP khác bất cứ lúc nào. Service cung cấp một endpoint mạng ổn định, dùng selector để luôn trỏ đúng tới tập Pod hiện hành, giúp client không cần biết hay quan tâm tới IP thực tế của từng Pod.

**Q6: `depends_on` trong Docker Compose có đảm bảo service phụ thuộc đã sẵn sàng xử lý request chưa?**
→ Không. `depends_on` chỉ đảm bảo thứ tự container được *khởi động*, không đảm bảo ứng dụng bên trong đã hoàn tất khởi tạo (ví dụ database chưa migrate xong). Cần bổ sung health check hoặc cơ chế retry ở tầng gọi để xử lý đúng khoảng trễ này.

**Q7: Vì sao không nên dùng tag `latest` cho image trong production?**
→ Vì `latest` là tag di động — nội dung nó trỏ tới có thể thay đổi bất cứ lúc nào mà không có cảnh báo, phá vỡ tính tái lập của deployment. Luôn pin một version cụ thể để đảm bảo mọi lần deploy đều dùng đúng một artifact đã kiểm thử.

**Q8: Container chia sẻ chung kernel host mang lại rủi ro bảo mật gì, và giảm thiểu ra sao?**
→ Nếu một container bị khai thác lỗ hổng kernel, về lý thuyết có thể ảnh hưởng tới các container khác trên cùng host — rủi ro cao hơn trong môi trường multi-tenant. Giảm thiểu bằng cách dùng base image cập nhật, scan lỗ hổng thường xuyên (Trivy, Anchore), áp dụng kiểm soát runtime (AppArmor, seccomp, PodSecurityPolicies), và thiết lập network policy ngăn di chuyển ngang giữa service.

---

## Bài tập thực hành đề xuất

1. Viết Dockerfile multi-stage cho một service ASP.NET Core bất kỳ trong hệ thống HealthCare (ví dụ Patients API), tự kiểm chứng kích thước image cuối cùng nhỏ hơn đáng kể so với khi build 1 stage duy nhất.
2. Tạo `docker-compose.yml` điều phối tối thiểu 3 service (Patients API, Appointments API, Redis), khai báo `depends_on` đúng thứ tự, chạy thử bằng `docker-compose up` và quan sát log khởi động.
3. Thử nghiệm container hóa một image database có seed sẵn dữ liệu (theo cách Dockerfile + script, không dùng `docker commit`), push lên Docker Hub cá nhân (tài khoản free), rồi pull lại từ máy khác để xác nhận tính tái lập.
4. Cài Minikube, viết Deployment + Service manifest cho một service, deploy bằng `kubectl apply`, sau đó thử xóa thủ công một Pod (`kubectl delete pod ...`) và quan sát Kubernetes tự động tạo lại Pod mới — minh họa trực quan cho self-healing.
5. So sánh thời gian và độ phức tạp thao tác giữa việc scale service lên 3 instance bằng Docker Compose (thủ công sửa file, restart) và bằng Kubernetes (`kubectl scale deployment ... --replicas=3`), rút ra nhận xét cá nhân về khi nào đáng đầu tư sang K8s.

---

*Ghi chú: Chương tiếp theo sẽ mở rộng sang các mô hình phát triển hướng cloud (Cloud Development Strategies) — dự kiến bao gồm Serverless Microservices Development và Observability/Monitoring, theo đúng mục lục Part 4 đã xác nhận (Container Hosting → Serverless Microservices Development → Observability and Monitoring with Modern Tools → Wrapping It All Up).*
