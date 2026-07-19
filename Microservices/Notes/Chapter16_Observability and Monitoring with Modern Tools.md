# Chapter 16 — Observability and Monitoring with Modern Tools: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| Monitoring | Cho biết CÓ vấn đề đang xảy ra (ví dụ đèn cảnh báo bật sáng) |
| Observability | Cho biết TẠI SAO vấn đề xảy ra, thông qua suy luận từ logs/metrics/traces |
| Log | Bản ghi rời rạc, có timestamp, mô tả một sự kiện cụ thể |
| Structured logging | Log dưới dạng dữ liệu có thuộc tính (key-value), truy vấn/filter/join được thay vì chỉ là văn bản |
| Metric | Con số đo lường sức khỏe hệ thống theo thời gian (latency, error rate, throughput...) |
| Trace | Hành trình đầu-cuối của một request xuyên qua nhiều service |
| Span | Một đoạn công việc do một service thực hiện trong một khoảng thời gian, nhiều span ghép thành một trace |
| Tag | Siêu dữ liệu gắn với span, dùng để filter/query (HTTP method, DB host, session ID...) |
| Correlation ID | Mã định danh duy nhất gắn với một luồng nghiệp vụ, dùng để nối log/metric/trace của cùng một request |
| Log level | Mức độ nghiêm trọng của log: Information, Debug, Warning, Error, Critical |
| ILogger\<T\> | Interface logging built-in của .NET, T = tên class phát sinh log, phục vụ categorization |
| Log aggregation | Thu thập log từ nhiều service về một nền tảng tập trung để tra cứu |
| Sink | Khái niệm của Serilog tương đương "provider" — đích xuất log (Console, File, Seq...) |
| OpenTelemetry (OTel) | Chuẩn mở để sinh và thu thập telemetry (trace + metric), không khóa cứng vào một vendor |
| Sidecar pattern | Tiến trình/container phụ trợ chạy cùng vòng đời với service chính, đảm nhiệm cross-cutting concerns |
| Service mesh | Tầng mạng gồm data plane (sidecar proxy) + control plane (chính sách tập trung), quản lý giao tiếp service-to-service |
| Data plane | Các sidecar proxy (Envoy) xử lý traffic thực tế |
| Control plane | Thành phần trung tâm cấu hình chính sách cho các proxy |
| mTLS | Mutual TLS — mã hóa hai chiều giữa các service, thường do mesh tự động enforce |
| Dapr | Distributed Application Runtime — sidecar nhẹ phơi bày API HTTP/gRPC thống nhất cho cross-cutting concerns |
| .NET Aspire | Lớp orchestration + observability ở development-time cho ứng dụng .NET phân tán |
| AppHost (Aspire) | Project mô tả toàn bộ app model: service, hạ tầng, dependency, là điểm chạy duy nhất cục bộ |
| ServiceDefaults (Aspire) | Thư viện dùng chung cấu hình OTel, health check, resilience, discovery nhất quán cho mọi service |

---

## 2. So sánh Jaeger vs Prometheus

| Tiêu chí | Jaeger | Prometheus |
|---|---|---|
| Loại dữ liệu | Distributed traces | Time-series metrics |
| Câu hỏi trả lời | Request này chậm/lỗi ở bước nào? | Service này có khỏe mạnh không? |
| Nguồn gốc | Uber, nay thuộc CNCF | CNCF |
| Theo dõi request cá nhân | Có | Không |
| Alerting + dashboard | Hạn chế | Mạnh, tích hợp Grafana |
| Vai trò trong hệ thống | Bổ trợ, không thay thế Prometheus | Bổ trợ, không thay thế Jaeger |

## 3. So sánh các thư viện logging .NET

| Thư viện | Điểm mạnh | Điểm yếu |
|---|---|---|
| Serilog | Structured logging, nhiều sink, tích hợp OTel/cloud mượt | Cần thêm package cho từng sink |
| NLog | Hiệu năng tốt, nhiều target | Structured logging/OTel chưa mượt bằng Serilog |
| Log4Net | Ổn định, lâu đời | Ergonomics cũ, chậm cập nhật .NET mới |
| elmah.io | Trải nghiệm hosted tốt | Kém linh hoạt khi cần nhiều sink tự host |

## 4. So sánh Sidecar / Service Mesh / Dapr / Aspire

| Tiêu chí | Sidecar thuần | Service Mesh (Istio/Linkerd) | Dapr | .NET Aspire |
|---|---|---|---|---|
| Phạm vi | 1 mối quan tâm cụ thể (proxy, logging) | Toàn bộ network layer (data + control plane) | Building block cross-cutting qua API | Development-time orchestration |
| Đa ngôn ngữ | Tùy triển khai | Có (transparent với app) | Có (HTTP/gRPC) | Chủ yếu hệ sinh thái .NET |
| Dùng cho production | Có | Có | Có | Không (chỉ dev-time) |
| Chi phí thêm | 1 container/service | Data plane + control plane | 1 container/service | Không (chỉ chạy local) |
| Ví dụ | Envoy đơn lẻ | Istio, Linkerd, Consul Connect, OSM | Dapr runtime | AppHost + ServiceDefaults |

---

## Checklist kỹ thuật khi triển khai

### Logging
- [ ] Xác định service nào cần log chi tiết (Information) vs chỉ log lỗi (Error)
- [ ] Không log credential, payment info, PII — chỉ log ID tra cứu được
- [ ] Dùng log level nhất quán trên toàn team, tránh gán sai mức nghiêm trọng
- [ ] Dùng enum cho eventId thay vì hard-code số
- [ ] Cài Serilog + ít nhất 1 sink phù hợp môi trường (Console cho dev, File + Seq/Application Insights cho production)
- [ ] Gắn correlation ID vào mọi log entry liên quan tới một luồng nghiệp vụ

### Distributed Tracing
- [ ] Cài OTel packages (Instrumentation.AspNetCore, Extensions.Hosting)
- [ ] Cấu hình OTLP exporter trỏ đúng endpoint (Jaeger/Collector)
- [ ] Bật `RecordException = true` cho AspNetCoreInstrumentation để bắt exception vào trace
- [ ] Phân biệt rõ khi nào cần Jaeger (trace) và khi nào cần Prometheus (metric) — không dùng nhầm vai trò

### Sidecar / Service Mesh
- [ ] Đánh giá số lượng service đủ lớn để việc thêm mesh đáng giá hay chưa
- [ ] Nếu dùng mesh: xác định rõ ai (mesh hay code) chịu trách nhiệm retry/mTLS/traffic shaping, tránh chồng chéo
- [ ] Tính toán overhead CPU/RAM khi mỗi service có thêm 1 sidecar container

### Dapr
- [ ] Chạy `dapr init` trước khi bắt đầu, kiểm tra Redis/Zipkin container đã lên
- [ ] Định nghĩa component YAML cho state store và pub/sub, tách biệt code khỏi backend cụ thể
- [ ] Cấu hình samplingRate và endpoint tracing trong config.yaml
- [ ] Không PHI/PII trong span — dùng surrogate key

### .NET Aspire
- [ ] Dùng AppHost làm điểm chạy duy nhất cho local dev, không tự chạy từng project riêng lẻ
- [ ] Gọi `AddServiceDefaults()` và `MapDefaultEndpoints()` ở mọi microservice trong hệ thống
- [ ] Hiểu rõ "run" (Aspire) khác với "publish" (triển khai production thật)

---

## Anti-patterns / Lưu ý cần tránh

- **Log quá chatty (verbose logging)**: chôn vùi sự kiện quan trọng dưới hàng ngàn dòng nhiễu, tăng chi phí lưu trữ vô ích.
- **Log thông tin nhạy cảm**: vi phạm GDPR/HIPAA/PCI DSS, rủi ro pháp lý và bảo mật nghiêm trọng nếu log bị rò rỉ.
- **Gán sai log level**: log lỗi nghiêm trọng ở mức Information khiến hệ thống cảnh báo mất tác dụng.
- **Dùng Jaeger để thay Prometheus hoặc ngược lại**: hai công cụ phục vụ hai câu hỏi khác nhau, không thể thay thế nhau.
- **Thêm sidecar/service mesh cho hệ thống nhỏ, tĩnh**: overhead vận hành và độ trễ vượt xa lợi ích chuẩn hóa mang lại.
- **Dùng .NET Aspire như một giải pháp production**: Aspire là công cụ development-time, không phải orchestrator production.
- **Không gắn correlation ID xuyên service**: khiến log/metric/trace của cùng một request bị tách rời, mất khả năng ghép lại thành câu chuyện.
- **Trộn lẫn trách nhiệm giữa Dapr và service mesh**: nếu dùng cả hai, cần rõ ràng lớp nào sở hữu retry/mTLS để tránh xung đột hành vi.

---

## Từ vựng chuyên ngành (Anh - Việt)

| Thuật ngữ | Nghĩa |
|---|---|
| Observability | Khả năng quan sát/suy luận trạng thái nội bộ hệ thống |
| Structured logging | Ghi log dạng dữ liệu có cấu trúc |
| Log aggregation | Gom log tập trung |
| Distributed tracing | Truy vết phân tán |
| Span | Đoạn công việc trong một trace |
| Sidecar pattern | Mẫu hình tiến trình phụ trợ đi kèm |
| Service mesh | Lưới dịch vụ / tầng mạng quản lý giao tiếp service |
| Data plane | Tầng dữ liệu (xử lý traffic thực tế) |
| Control plane | Tầng điều khiển (cấu hình chính sách) |
| mTLS (Mutual TLS) | Mã hóa TLS hai chiều |
| Canary release | Triển khai thử nghiệm dần trên một phần traffic |
| Throughput | Thông lượng xử lý |
| Latency | Độ trễ |
| Sampling rate | Tỷ lệ lấy mẫu (trace) |
| Zero-trust networking | Mô hình mạng không tin tưởng mặc định, luôn xác thực |
| PII (Personally Identifiable Information) | Thông tin định danh cá nhân |
| Readiness probe | Kiểm tra dịch vụ đã sẵn sàng nhận traffic |
| Liveness probe | Kiểm tra tiến trình còn sống |

---

## Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Phân biệt Monitoring và Observability?**
→ Monitoring cho biết CÓ vấn đề (dashboard đỏ, alert bắn ra). Observability cho biết TẠI SAO có vấn đề, dựa trên khả năng suy luận trạng thái nội bộ từ dữ liệu thu được (logs, metrics, traces) mà không cần thêm code debug và deploy lại. Một hệ thống có monitoring tốt vẫn có thể thiếu observability nếu dữ liệu thu thập không đủ chi tiết hoặc không liên kết được với nhau.

**Q2: Vì sao cần cả ba tín hiệu logs, metrics, traces — không thể chỉ dùng một?**
→ Mỗi tín hiệu trả lời một câu hỏi khác nhau: log cho chi tiết một sự kiện cụ thể, metric cho xu hướng tổng thể theo thời gian, trace cho hành trình xuyên nhiều service. Ví dụ: metric báo latency tăng, nhưng chỉ trace mới chỉ ra chính xác service nào gây nghẽn, và log ở service đó mới cho biết nguyên nhân cụ thể (timeout, exception nào).

**Q3: Correlation ID giải quyết vấn đề gì trong hệ thống phân tán?**
→ Khi một request đi qua nhiều service, mỗi service tự sinh log riêng biệt. Không có correlation ID, không có cách nào chắc chắn liên kết các dòng log đó lại thành đúng một luồng nghiệp vụ, đặc biệt khi có traffic đồng thời từ nhiều request khác nhau. Correlation ID là mã định danh duy nhất được truyền xuyên suốt (qua header) để làm chìa khóa lọc.

**Q4: Serilog khác gì so với ILogger built-in của .NET?**
→ ILogger là tầng trừu tượng logging built-in, output mặc định là văn bản. Serilog là một implementation cắm phía sau ILogger, biến mỗi dòng log thành "event có thuộc tính" (structured), cho phép filter/join/pivot dữ liệu log như một dataset thay vì đọc từng dòng văn bản, đồng thời hỗ trợ nhiều sink khác nhau mà không đổi code nghiệp vụ.

**Q5: Jaeger và Prometheus có thể thay thế nhau không? Vì sao?**
→ Không. Jaeger chuyên về distributed tracing — theo dấu một request cụ thể xuyên qua nhiều service, không thu thập metric. Prometheus chuyên về time-series metrics — theo dõi các con số hệ thống theo thời gian, không theo dõi được từng request riêng lẻ. Chúng bổ trợ nhau: Prometheus phát hiện "có vấn đề", Jaeger giải thích "vấn đề ở đâu".

**Q6: Sidecar pattern giải quyết vấn đề gì, và cái giá phải trả là gì?**
→ Giải quyết vấn đề mỗi service phải tự nhúng logic cross-cutting (logging, retry, mTLS...) gây phình code và thiếu nhất quán. Sidecar tách các trách nhiệm đó ra một tiến trình đồng hành, chuẩn hóa trên toàn hệ thống. Cái giá: tăng gấp đôi số container cần vận hành, thêm độ trễ do traffic phải qua proxy, và tăng độ khó debug vì lỗi có thể nằm ở service, proxy, hoặc control plane.

**Q7: Khi nào nên chọn Dapr thay vì một service mesh đầy đủ (Istio)?**
→ Chọn Dapr khi cần các building block cross-cutting (state, pub/sub, service invocation) qua API thống nhất, đặc biệt trong môi trường đa ngôn ngữ, mà không cần kiểm soát traffic-layer sâu như mesh cung cấp. Chọn mesh đầy đủ khi cần chính sách network tập trung, mTLS toàn diện, và định tuyến nâng cao (canary) trên quy mô lớn.

**Q8: .NET Aspire có dùng được cho production không?**
→ Không. Aspire là công cụ orchestration + observability ở development-time, giúp dựng và quan sát toàn bộ topology cục bộ bằng một lệnh chạy duy nhất. Khi triển khai thật, hệ thống vẫn cần publish sang nền tảng production thực sự (AKS, Azure Container Apps, Kubernetes...) — "run" và "publish" là hai khái niệm tách biệt trong Aspire.

---

## Bài tập thực hành đề xuất

1. Thêm Serilog vào service Appointments hiện có, cấu hình 2 sink (Console + File), sau đó chạy Seq bằng Docker và verify log xuất hiện đúng trên dashboard Seq.
2. Cấu hình OpenTelemetry cho cả hai service Appointments và Patients, trỏ về cùng một Jaeger instance, sau đó gọi Appointments → Patients và quan sát trace liên service trên giao diện Jaeger.
3. Viết một enum `LogEvents` đầy đủ cho tối thiểu 5 loại thao tác trong service Appointments (create, retrieve, update, delete, failure), áp dụng vào toàn bộ controller.
4. Dựng thử một sidecar Dapr cho service Appointments với state store Redis, publish thử một event `appointment.created` và verify nó xuất hiện trên Zipkin.
5. Tạo một project .NET Aspire tối giản với 2 service (Appointments, Patients) + 1 PostgreSQL server có 2 database riêng, verify toàn bộ resource "Running" trên dashboard Aspire.

---

*Ghi chú: Chương tiếp theo (Chapter 17) dự kiến là chương tổng kết cuối sách — "Wrapping It All Up" — sẽ ôn lại hành trình từ Chapter 1 tới Chapter 16.*
