# Chapter 1 — Introducing Microservices: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng tóm tắt cốt lõi

| Kiến trúc | Deploy unit | Scale | Công nghệ | Khi nào dùng |
|---|---|---|---|---|
| **Monolith** | 1 khối duy nhất | Toàn bộ app cùng lúc | 1 stack duy nhất | Sản phẩm mới, team nhỏ, cần tốc độ ra mắt nhanh |
| **Modular Monolith** | 1 khối duy nhất, module tách biệt rõ | Toàn bộ app cùng lúc | 1 stack duy nhất | Đã có nhiều domain rõ ràng, chưa cần scale/deploy độc lập |
| **Microservices** | Mỗi service 1 unit | Từng service độc lập | Đa dạng theo từng service | Cần scale riêng lẻ, đa team, đa công nghệ, tổ chức đã trưởng thành về DevOps |

**Câu thần chú cần nhớ**: Microservices không phải "chia file nhỏ ra". Bản chất là **tự trị (autonomy)** — mỗi service tự phát triển, tự test, tự deploy, tự scale.

---

## 2. Ba cơn đau khiến người ta rời bỏ Monolith

1. **Scaling khó** — không scale được từng phần, phải nhân bản toàn bộ ứng dụng dù chỉ 1 module cần thêm tài nguyên.
2. **Deployment bottleneck** — sửa 1 dòng code cũng phải build/test/deploy lại toàn bộ hệ thống.
3. **Higher risk of code debt** — dependency chồng chéo theo thời gian, tạo ra spaghetti code, không ai dám sửa code cũ.

---

## 3. Modular Monolith — bước đệm hay bị bỏ qua

- **Định nghĩa**: 1 deploy unit, nhưng code chia thành module theo domain nghiệp vụ, module không được đụng chéo vào nhau.
- **Được**: separation of concerns, phát triển/test song song giữa các team, vẫn đơn giản khi deploy.
- **Mất**: vẫn không scale riêng từng module, vẫn 1 stack công nghệ duy nhất, redeploy toàn bộ khi có thay đổi.
- **Case study thực tế**: Shopify, GitHub công khai vận hành modular monolith ở quy mô lớn, chỉ tách microservice khi có lý do cụ thể.
- **Vai trò chiến lược**: middle ground hợp lý cho app có thể tiến hóa dần lên microservices sau này.

---

## 4. Microservices — 3 lợi ích cốt lõi (interview hay hỏi)

| Lợi ích | Giải thích | Ví dụ |
|---|---|---|
| **Independent scalability** | Scale đúng service đang chịu tải, tiết kiệm chi phí hạ tầng | Appointment + Billing quá tải mùa tiêm chủng, Customer Service giữ nguyên |
| **Technology diversity** | Mỗi service chọn stack phù hợp nhất với nhu cầu dữ liệu/nghiệp vụ | Customer dùng SQL Server, Billing dùng MongoDB |
| **Team autonomy** | Team sở hữu trọn service từ code tới vận hành, không nghẽn ở release chung | Netflix, Amazon, Uber vận hành hàng trăm team song song |

**Khái niệm liên quan cần biết**: *Conway's Law* — cấu trúc hệ thống bạn xây sẽ phản ánh cấu trúc tổ chức team của bạn.

---

## 5. Checklist ra quyết định: Monolith hay Microservices?

Tự hỏi 5 câu này trước khi quyết định (đây là câu hỏi phỏng vấn kinh điển: "Khi nào bạn KHÔNG chọn microservices?"):

- [ ] **Application complexity & size** — App có đủ lớn/phức tạp để justify chi phí vận hành phân tán chưa?
- [ ] **Team structure & expertise** — Team có kinh nghiệm vận hành distributed system chưa?
- [ ] **Scalability requirements** — Có traffic tăng đột biến theo mùa/sự kiện thực sự không?
- [ ] **Technology diversity needs** — Có module nào thật sự cần stack khác biệt, không phải "muốn thử cho vui"?
- [ ] **Organizational readiness** — Đã có Agile/DevOps/CI-CD trưởng thành chưa?

> Nếu phần lớn câu trả lời là "chưa" → ở lại Modular Monolith. Đừng microservices hóa vì trend.

---

## 6. Nguyên tắc thiết kế bắt buộc phải nhớ

1. **Database-per-service** — mỗi service sở hữu database riêng. Share DB giữa các service = coupling ẩn nguy hiểm, migration của team A có thể phá vỡ team B.
2. **Dữ liệu trùng lặp có chủ đích** — ví dụ Customer info xuất hiện ở cả Appointment lẫn Billing service, đồng bộ qua event, không join trực tiếp qua DB.
3. **Giao tiếp giữa service** — đồng bộ (REST/gRPC) hoặc bất đồng bộ (message queue/event). Chương sau sẽ đi sâu.
4. **Gateway pattern** — tổng hợp các service sau 1 entry point cho client, nhưng phải HA (nhiều instance), nếu không nó trở thành SPOF mới.

---

## 7. Vấn đề phát sinh khi phân tán hệ thống (và pattern giải quyết)

| Vấn đề | Vì sao xảy ra | Pattern/Giải pháp |
|---|---|---|
| **Data consistency** | Một workflow (vd: booking) giờ chạy qua nhiều service, không còn 1 SQL transaction | **Saga Pattern** (orchestration hoặc choreography) |
| **Distributed debugging** | Log rải rác ở N service, không biết request lỗi ở đâu | **Centralized logging + Distributed Tracing (OpenTelemetry)** |
| **Cascading failure** | Service A phụ thuộc B, B down kéo A theo | **Circuit Breaker, Retry với backoff (thư viện Polly trong .NET)** |
| **Health visibility** | Không biết instance nào đang "chết" | **Health checks endpoint + dashboard** |
| **Single point of failure ở tầng gateway** | Gateway là entry point duy nhất | **Multi-instance Gateway + Load Balancing** |

---

## 8. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa | Ghi chú |
|---|---|---|
| Bounded Context | Ranh giới ngữ nghĩa của 1 domain nghiệp vụ | Nền tảng của Domain-Driven Design |
| Single Point of Failure (SPOF) | Điểm lỗi duy nhất làm sập cả hệ thống | Microservices giảm SPOF ở tầng service nhưng có thể tạo SPOF mới ở Gateway |
| Horizontal scaling (scale out) | Thêm instance mới, load balance | Ưu tiên trong cloud-native |
| Vertical scaling (scale up) | Tăng CPU/RAM cho 1 instance | Có giới hạn vật lý |
| Eventual Consistency | Dữ liệu sẽ nhất quán sau một khoảng thời gian, không tức thời | Trade-off bắt buộc khi từ bỏ ACID transaction toàn cục |
| Distributed Transaction | Transaction trải dài qua nhiều service/DB | Khó, thường thay bằng Saga thay vì 2PC |
| Service Discovery | Cơ chế để service tìm địa chỉ của service khác | Cần khi số lượng instance động (autoscaling) |
| Observability | Khả năng "nhìn thấy" trạng thái nội bộ hệ thống qua log/metric/trace | 3 trụ cột: Logs, Metrics, Traces |
| Conway's Law | Kiến trúc hệ thống phản ánh cấu trúc tổ chức | Dùng để giải thích vì sao cắt service theo team boundary |
| Premature Distribution | Tách microservices quá sớm khi chưa cần thiết | Anti-pattern phổ biến ở startup |
| Distributed Monolith | Nhiều service nhưng vẫn coupling chặt (share DB, gọi đồng bộ theo chuỗi) | Anti-pattern — tệ hơn cả monolith vì nhận đủ nhược điểm mà không có lợi ích |

---

## 9. Anti-Patterns — checklist tự audit hệ thống đang làm

- [ ] Có đang share chung 1 database giữa nhiều service không? → Nếu có: Distributed Monolith risk.
- [ ] Có service nào luôn phải gọi đồng bộ, đúng thứ tự qua 3-4 service khác để hoàn thành 1 request không? → Cascading failure risk.
- [ ] Team hiện tại có dưới 5 người mà đã có hơn 5 service chưa? → Premature distribution risk.
- [ ] Có centralized logging/tracing chưa, hay đang phải SSH vào từng server để đọc log? → Observability gap.
- [ ] API Gateway có chạy nhiều instance, có health check, load balancer chưa? → SPOF risk.

---

## 10. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Khi nào bạn KHÔNG nên dùng microservices?**
→ Khi team nhỏ, thiếu kinh nghiệm vận hành distributed system, ứng dụng chưa đủ phức tạp để justify chi phí operational overhead (CI/CD đa pipeline, observability, service mesh...). Modular Monolith thường là lựa chọn tốt hơn ở giai đoạn này.

**Q2: Vì sao mỗi microservice nên có database riêng?**
→ Để đảm bảo loose coupling thật sự — service có thể deploy, scale, thay đổi schema độc lập mà không ảnh hưởng service khác. Share DB tạo ra coupling ẩn nguy hiểm hơn cả coupling qua code.

**Q3: Nếu bỏ transaction ACID toàn cục, làm sao đảm bảo data consistency giữa các service?**
→ Dùng Saga Pattern: chuỗi các local transaction, mỗi bước có một compensating transaction để "bù trừ" nếu bước sau thất bại. Chấp nhận eventual consistency thay vì strong consistency.

**Q4: API Gateway có phải là SPOF không?**
→ Có thể, nếu chỉ chạy 1 instance. Cách giải quyết: chạy nhiều instance, đặt sau load balancer, có health check tự động loại instance lỗi.

**Q5: Distributed Monolith là gì và vì sao nó tệ hơn cả cách tiếp cận thông thường?**
→ Là hệ thống đã tách thành nhiều service về mặt hình thức nhưng vẫn coupling chặt (share DB, gọi đồng bộ theo chuỗi bắt buộc). Nó nhận đủ nhược điểm của microservices (network latency, deploy phức tạp, khó debug) mà không có được lợi ích thực sự nào (không scale độc lập được, không deploy độc lập được).

**Q6: Conway's Law áp dụng thế nào khi thiết kế microservices?**
→ Ranh giới service nên phản ánh ranh giới team sở hữu nghiệp vụ đó. Nếu team boundary không rõ ràng, service boundary thiết kế trên giấy cũng sẽ dần bị phá vỡ trong thực tế vận hành.

---

## 11. Case study thực tế để dẫn chứng khi phỏng vấn

- **Netflix**: từ hệ thống DVD rental monolith → microservices để hỗ trợ streaming quy mô lớn, cho phép deploy hàng nghìn service độc lập, tăng khả năng phục hồi (resilience) và tốc độ phát triển tính năng.
- **Uber, Amazon, Etsy**: được nhắc đến như các case điển hình áp dụng microservices ở quy mô tổ chức lớn, nhiều team song song.
- **Shopify/GitHub** (kiến thức ngành, không phải trong sách): ví dụ về việc vận hành modular monolith thành công ở quy mô lớn, minh chứng rằng không phải ai cũng cần microservices ngay.

---

## 12. Bài tập thực hành đề xuất (để nhớ lâu)

1. Vẽ lại sơ đồ bounded context cho hệ thống phòng khám (Customer, Appointment, Doctor Calendar, Billing) — quyết định module nào gộp, module nào tách, giải thích lý do bằng 1 trong 5 tiêu chí ở mục 5.
2. Viết pseudocode cho luồng "đặt lịch khám" theo kiểu Orchestration (1 service điều phối gọi các service khác) — cố tình để lộ điểm data inconsistency nếu 1 bước giữa chừng thất bại.
3. Liệt kê 3 tình huống trong dự án hiện tại của bạn có dấu hiệu Distributed Monolith (nếu đang làm microservices) hoặc dấu hiệu nên cân nhắc modular hóa (nếu đang làm monolith).

---

*Ghi chú: Chương tiếp theo (Chapter 2) sẽ đi sâu vào Domain-Driven Design — công cụ chính thức để xác định bounded context một cách có hệ thống, thay vì cảm tính.*
