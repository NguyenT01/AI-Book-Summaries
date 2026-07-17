# Chapter 3 — Synchronous Communication: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Synchronous Communication** | Service gọi service khác, chờ phản hồi ngay mới tiếp tục — giống gọi hàm nhưng qua mạng |
| **Blocking behavior** | Bên gọi bị "đứng hình" cho tới khi có phản hồi |
| **Cumulative Latency** | Độ trễ cộng dồn khi request đi qua chuỗi nhiều service (network hop) |
| **Retry Pattern** | Thử lại request khi gặp lỗi tạm thời (transient failure) thay vì fail ngay |
| **Circuit Breaker Pattern** | Ngắt tạm thời việc gọi 1 service khi nó liên tục lỗi, tránh nghẽn tài nguyên |
| **Transient Failure** | Lỗi tạm thời, thường tự hết (network chập chờn, service restart) |
| **Socket Exhaustion** | Hết cổng kết nối khả dụng do `HttpClient` không được quản lý vòng đời đúng cách |
| **Protobuf** | Protocol Buffers — định dạng nhị phân dùng trong gRPC, nhanh và nhỏ gọn hơn JSON |
| **IDL** | Interface Definition Language — file `.proto` định nghĩa contract dịch vụ gRPC |
| **Multiplexing** | HTTP/2 cho phép nhiều request/response chạy đồng thời trên 1 kết nối TCP |
| **Stub/Proxy** | Đối tượng phía client cung cấp cùng method như service từ xa, cho phép gọi như hàm cục bộ |

---

## 2. Ba mối quan tâm bắt buộc khi thiết kế giao tiếp đồng bộ

| Mối quan tâm | Vấn đề | Giải pháp |
|---|---|---|
| **Performance** | Độ trễ cộng dồn qua nhiều network hop | Giảm số hop, dùng gRPC khi cần throughput cao, timeout hợp lý |
| **Resilience** | Lỗi tạm thời, service quá tải | Retry Pattern (cẩn trọng với write operation) + Circuit Breaker Pattern |
| **Tracing/Monitoring** | Request qua nhiều service, khó debug | Centralized logging + distributed tracing (OpenTelemetry, Azure Monitor) |

**Lưu ý quan trọng khi retry**: KHÔNG retry mù quáng với thao tác ghi dữ liệu (POST/PUT) — request có thể đã tới đích và xử lý thành công, chỉ mất response trên đường về. Retry có thể gây duplicate operation (đặt lịch 2 lần, trừ tiền 2 lần).

---

## 3. So sánh SOAP vs REST

| Tiêu chí | SOAP | REST |
|---|---|---|
| Định dạng | XML only | JSON, XML, HTML, plain text |
| Độ dài message | Verbose (cồng kềnh) | Gọn nhẹ |
| Bảo mật/Transaction | WS-Security, hỗ trợ rollback | Không có sẵn, cần tự xây |
| Hiệu năng parse | Nặng (XML) | Nhẹ (JSON) |
| Use case | Enterprise, ngành quản lý chặt (tài chính, y tế) | Web/mobile client, hệ thống hiện đại nói chung |
| Trạng thái hiện tại | Ít dùng cho service mới | Chuẩn phổ biến nhất hiện nay |

---

## 4. HTTP Headers — phân loại theo mục đích

| Nhóm | Header ví dụ | Mục đích |
|---|---|---|
| Auth | `Authorization` | Mang JWT/API key/OAuth token |
| Content negotiation | `Content-Type`, `Accept` | Khai báo/yêu cầu định dạng payload |
| Tracing | `X-Request-ID`, `traceparent`, `baggage` | Lan truyền trace ID qua các service |
| Cache/Compression | `Cache-Control`, `ETag`, `Content-Encoding` | Tối ưu hiệu năng |
| Custom | Tự định nghĩa | Trao đổi context-specific data |

## HTTP Status Code — 5 nhóm

| Nhóm | Ý nghĩa |
|---|---|
| 1xx | Thông tin giao thức |
| 2xx | Thành công |
| 3xx | Redirect |
| 4xx | Lỗi client (400 bad request, 404 not found...) |
| 5xx | Lỗi server |

---

## 5. So sánh REST vs gRPC — bảng ôn nhanh cho phỏng vấn

| Tiêu chí | REST | gRPC |
|---|---|---|
| Đối tượng gọi | Browser, mobile, client bên ngoài | Service-tới-service nội bộ |
| Định dạng | JSON (text, dễ đọc) | Protobuf (binary, nhỏ gọn) |
| Giao thức | HTTP/1.1 | HTTP/2 (multiplexing) |
| Contract | Lỏng lẻo, tự quy ước | Chặt chẽ, contract-first (.proto) |
| Streaming | Không hỗ trợ tự nhiên | Server/Client/Bidirectional streaming |
| Đa ngôn ngữ | Tốt | Rất tốt, tự sinh code |
| Tốc độ serialize | Chậm hơn | Nhanh hơn (binary) |

---

## 6. Checklist kỹ thuật khi implement REST API trong .NET

- [ ] Value object (VD `TimeSlot`, `Location`) có đánh dấu `[Owned]` cho EF Core chưa?
- [ ] Controller có dùng `[ApiController]` + `ControllerBase` để tự động validation/binding không?
- [ ] Database context và các service khác có được inject qua constructor (Dependency Injection) không, tránh tự `new` thủ công?
- [ ] Endpoint có trả đúng status code theo ngữ nghĩa không (200/201/204/400/404)?
- [ ] Concurrency exception (`DbUpdateConcurrencyException`) có được xử lý khi PUT/update không?

## 7. Checklist khi gọi service khác qua HttpClient

- [ ] Có đăng ký qua `AddHttpClient<T>()` (typed client) thay vì tự `new HttpClient()` không? — tránh **socket exhaustion**.
- [ ] Base address có được cấu hình qua `appsettings.json` thay vì hardcode không?
- [ ] Có gọi `EnsureSuccessStatusCode()` trước khi deserialize response không?
- [ ] Nếu gọi nhiều service trong 1 endpoint, có đánh giá được tổng độ trễ cộng dồn chưa?

## 8. Checklist khi implement gRPC

- [ ] File `.proto` đã định nghĩa đầy đủ `service` (RPC method) và `message` (schema) chưa?
- [ ] `.csproj` đã cấu hình đúng `GrpcServices="Server"` (bên cung cấp) hoặc `GrpcServices="Client"` (bên gọi) chưa?
- [ ] gRPC endpoint address có được cấu hình động qua `appsettings.json` + `IConfiguration` không (không hardcode)?
- [ ] Lỗi có được raise đúng chuẩn qua `RpcException` với `StatusCode` phù hợp không (VD `StatusCode.NotFound`)?

---

## 9. Anti-patterns cần tránh

- [ ] Chuỗi gọi đồng bộ quá dài (A → B → C → D trong 1 request) → cumulative latency cao, cascading failure risk.
- [ ] Tự `new HttpClient()` trong từng method thay vì dùng `AddHttpClient` → socket exhaustion.
- [ ] Retry vô điều kiện cho cả thao tác ghi dữ liệu → rủi ro duplicate operation.
- [ ] Không có Circuit Breaker khi gọi service hay lỗi → hao tổn tài nguyên khi service đích đang down.
- [ ] Hardcode URL endpoint (REST hay gRPC) thay vì đọc từ configuration → không portable giữa các environment.
- [ ] Không có centralized tracing → không debug được request lỗi ở service nào trong chuỗi.

---

## 10. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Synchronous Communication | Giao tiếp đồng bộ — gọi và chờ phản hồi ngay |
| Blocking call | Cuộc gọi khiến bên gọi bị chặn/chờ |
| Transient Failure | Lỗi tạm thời, thường tự khắc phục |
| Retry Pattern | Mẫu thiết kế thử lại khi gặp lỗi tạm thời |
| Circuit Breaker Pattern | Mẫu thiết kế ngắt mạch khi service lỗi liên tục |
| Cascading Failure | Lỗi dây chuyền lan qua nhiều service |
| Cumulative Latency | Độ trễ cộng dồn qua nhiều network hop |
| Network Hop | Một lần truyền dữ liệu qua mạng giữa 2 điểm |
| Idempotency | Tính chất một thao tác thực hiện nhiều lần cho cùng kết quả (quan trọng khi retry) |
| Typed Client | HttpClient được đăng ký kèm theo class riêng, quản lý vòng đời tự động |
| Socket Exhaustion | Cạn kiệt cổng kết nối do quản lý HttpClient sai cách |
| Protocol Buffers (Protobuf) | Định dạng dữ liệu nhị phân dùng trong gRPC |
| Contract-first | Thiết kế bắt đầu từ hợp đồng dữ liệu (.proto) trước khi code |
| Streaming (Server/Client/Bidirectional) | Các mô hình truyền dữ liệu liên tục trong gRPC |
| Stub/Proxy | Đại diện phía client gọi method như thể service ở cục bộ |
| Distributed Tracing | Theo dõi một request xuyên suốt nhiều service |

---

## 11. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Giao tiếp đồng bộ có nhược điểm gì trong kiến trúc microservices?**
→ Tạo tight coupling giữa các service, tăng độ trễ cộng dồn khi chuỗi gọi dài, và có nguy cơ cascading failure — nếu 1 service chậm/down, toàn bộ chuỗi request phụ thuộc nó cũng bị ảnh hưởng.

**Q2: Phân biệt Retry Pattern và Circuit Breaker Pattern.**
→ Retry: thử lại request khi gặp lỗi tạm thời, hy vọng lỗi tự hết. Circuit Breaker: sau một ngưỡng lỗi liên tục, chủ động ngừng gọi service đó trong một khoảng thời gian để tránh lãng phí tài nguyên và cho service đích thời gian hồi phục. Hai pattern thường dùng kết hợp.

**Q3: Vì sao cần cẩn trọng khi Retry với thao tác ghi dữ liệu (POST/PUT)?**
→ Vì request có thể đã tới đích và được xử lý thành công, chỉ là response bị mất trên đường về do lỗi tạm thời. Nếu retry mù quáng, thao tác có thể bị thực hiện trùng lặp (duplicate). Cần đảm bảo idempotency (VD dùng idempotency key) trước khi áp dụng retry cho write operation.

**Q4: Vì sao nên dùng `AddHttpClient<T>()` thay vì tự khởi tạo `new HttpClient()`?**
→ Tự khởi tạo và không dispose đúng cách dẫn tới socket exhaustion — cạn kiệt cổng kết nối khả dụng khiến service không gọi ra ngoài được nữa. `AddHttpClient` quản lý vòng đời kết nối, tái sử dụng connection pool an toàn, đồng thời cho phép mock dễ dàng khi unit test.

**Q5: So sánh REST và gRPC — khi nào dùng cái nào?**
→ REST phù hợp cho client bên ngoài (browser, mobile) nhờ tính linh hoạt, dễ đọc (JSON), dễ debug. gRPC phù hợp cho giao tiếp nội bộ service-tới-service cần hiệu năng cao — nhờ Protobuf (nhị phân, nhỏ gọn) và HTTP/2 (multiplexing), cùng contract chặt chẽ giúp giảm lỗi sai lệch định dạng dữ liệu giữa các team.

**Q6: gRPC dùng HTTP/2 mang lại lợi thế gì so với HTTP/1.1 mà REST truyền thống thường dùng?**
→ HTTP/2 hỗ trợ multiplexing — nhiều request/response chạy đồng thời trên một kết nối TCP duy nhất, giảm độ trễ và tăng throughput. Nó cũng dùng binary framing giúp parse nhanh hơn so với text-based của HTTP/1.1 + JSON.

**Q7: Cumulative Latency là gì và làm sao giảm thiểu nó?**
→ Là độ trễ cộng dồn khi một request phải đi qua nhiều network hop (nhiều service gọi lẫn nhau tuần tự). Giảm thiểu bằng cách: giảm số hop trong chuỗi gọi, cân nhắc gọi song song thay vì tuần tự khi các cuộc gọi độc lập nhau, dùng gRPC cho các đường gọi nội bộ cần tốc độ, và luôn có timeout hợp lý.

**Q8: Tại sao SOAP vẫn được dùng trong một số ngành dù REST phổ biến hơn?**
→ Vì SOAP có contract chính thức (formal contract) đảm bảo cấu trúc message nghiêm ngặt và type validation, cùng hỗ trợ sẵn WS-Security và transaction rollback — phù hợp với ngành quản lý chặt như tài chính, y tế, nơi tính chuẩn hóa và bảo mật quan trọng hơn tốc độ phát triển.

---

## 12. Bài tập thực hành đề xuất

1. Dựng thử 2 Web API project (.NET) đơn giản: `PatientsApi` và `AppointmentsApi`. Cho `AppointmentsApi` gọi `PatientsApi` qua `HttpClient` (đăng ký đúng chuẩn `AddHttpClient`), đo thời gian phản hồi khi gọi tuần tự 2 service so với gọi song song bằng `Task.WhenAll`.
2. Viết một `.proto` file định nghĩa service `PrescriptionService` với 2 RPC method: `GetPrescriptionsByPatient` và `AddPrescription`. Implement server side.
3. Thêm thử một Retry Policy đơn giản (dùng thư viện Polly) cho lời gọi `PatientsApiClient.GetPatientAsync`, giới hạn tối đa 3 lần retry với exponential backoff.
4. Liệt kê trong hệ thống bạn đang làm: đâu là những chuỗi gọi đồng bộ có nguy cơ cumulative latency cao nhất? Đề xuất cách rút ngắn chuỗi hoặc chuyển một phần sang bất đồng bộ.

---

*Ghi chú: Chương tiếp theo (Chapter 4) sẽ khám phá giao tiếp bất đồng bộ (asynchronous communication) giữa các microservice — cách xử lý các tiến trình dài hạn không cần phản hồi ngay lập tức.*
