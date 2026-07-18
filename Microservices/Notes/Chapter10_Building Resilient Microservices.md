# Chapter 10 — Building Resilient Microservices: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Resilience** | Khả năng hệ thống phục hồi từ lỗi và tiếp tục hoạt động, dù ở trạng thái suy giảm |
| **Circuit Breaker** | Pattern ngắt cuộc gọi tới service lỗi liên tục, tránh retry vô hạn/DoS vô tình |
| **Retry Policy** | Chiến lược thử lại cuộc gọi thất bại, thường kèm exponential backoff |
| **Distributed Cache** | Cache tập trung, mọi service trong hệ thống có thể truy cập (VD: Redis) |
| **Dead-Letter Queue (DLQ)** | Queue chứa message không xử lý được sau nhiều lần retry, dùng làm lưới an toàn |
| **Outbox Pattern** | Lưu message vào bảng outbox cùng transaction với business data, đảm bảo atomicity |
| **Dual Write Problem** | Rủi ro ghi DB và message broker riêng biệt — 1 thành công, 1 thất bại → mất nhất quán |
| **Delegating Handler** | Interceptor HTTP client trong .NET, dùng để tự động đính kèm header (API key, JWT) |
| **TTL (Time To Live)** | Thời gian cache entry còn hiệu lực trước khi hết hạn |

---

## 2. Ba loại lỗi trong giao tiếp Đồng bộ

| Loại lỗi | Nguyên nhân | Giải pháp |
|---|---|---|
| **Timeout** | Request mất nhiều thời gian hơn timeout cấu hình | Timeout hợp lý + retry + circuit breaker |
| **Service Unavailability** | Service bảo trì, DB không truy cập được | Cache fallback + circuit breaker |
| **Poor Authentication Handling** | Thiếu/sai API key, token hết hạn | Delegating handler tự động đính kèm + auto-refresh token |

## 3. Ba loại lỗi trong giao tiếp Bất đồng bộ

| Loại lỗi | Nguyên nhân | Giải pháp |
|---|---|---|
| **Message Loss** | Broker fail, non-durable queue, consumer crash trước khi ACK | Durable queue + đúng ACK pattern + outbox trước khi enqueue |
| **Timeout/Duplicate Processing** | Network fail khiến ACK bị mất, retry gây xử lý trùng | Idempotent handler + deduplication token (message ID, correlation ID) |
| **Dual Write Problem** | Ghi DB và publish message riêng biệt, 1 thành công 1 thất bại | Outbox Pattern |

---

## 4. Retry Policy với Polly — checklist cấu hình

- [ ] Đã cài `Polly` và `Microsoft.Extensions.Http.Polly` chưa?
- [ ] `GetRetryPolicy()` có handle transient HTTP error, `!IsSuccessStatusCode`, và `HttpRequestException` chưa?
- [ ] Có dùng exponential backoff (`Math.Pow(2, retryAttempt)`) thay vì fixed interval không?
- [ ] Số lần retry có giới hạn hợp lý (VD 5 lần) chưa, tránh concurrency/resource contention?
- [ ] Đã đăng ký policy qua `.AddPolicyHandler(GetRetryPolicy())` trên `AddHttpClient<T>` chưa?

## 5. Circuit Breaker — Ba trạng thái

| Trạng thái | Ý nghĩa |
|---|---|
| **Closed** | Bình thường, mọi cuộc gọi đi qua |
| **Open** | Sau N lỗi liên tiếp, ngắt mạch — fail nhanh, không gọi thật tới service |
| **(Half-Open ngầm định)** | Sau thời gian timeout, cho phép thử lại; nếu thành công → Closed, nếu vẫn lỗi → Open lại |

**Cấu hình mẫu**: `CircuitBreakerAsync(5, TimeSpan.FromSeconds(30))` — mở circuit sau 5 lỗi liên tiếp, ngắt 30 giây.

**Lưu ý thứ tự**: Circuit Breaker Policy nên đặt SAU Retry Policy trong chain (`.AddPolicyHandler(GetRetryPolicy()).AddPolicyHandler(GetCircuitBreakerPolicy())`) — retry hoạt động trong phạm vi circuit breaker giám sát.

---

## 6. Checklist triển khai Distributed Cache với Redis

- [ ] Đã cài `Microsoft.Extensions.Caching.StackExchangeRedis` chưa?
- [ ] Đã đăng ký `AddStackExchangeRedisCache` với connection string đúng chưa?
- [ ] Cache key có đủ cụ thể để tránh đụng độ không (VD `appointment-details:{id}`)?
- [ ] Đã set TTL hợp lý theo tần suất thay đổi dữ liệu chưa (dữ liệu ít đổi → TTL dài hơn)?
- [ ] Có wrapper `ICacheProvider`/`CacheProvider` để tái sử dụng logic get/set/clear không?
- [ ] Đã cân nhắc trade-off: cache giúp giảm tải nhưng có thể phục vụ dữ liệu stale khi source service down?

---

## 7. Hai phương pháp Authentication cho Service-to-Service

| Phương pháp | Đặc điểm | Khi nào dùng |
|---|---|---|
| **API Key** | Gửi qua HTTP header (VD `x-api-key`), đơn giản | Third-party/external service |
| **JWT (JSON Web Token)** | Token gọn nhẹ, URL-safe, cấp sau login, stateless | Service nội bộ (phổ biến hơn) |

**Hai lỗi cần xử lý riêng**:
- **401 Expired JWT**: cần logic tự động lấy token mới và retry request ban đầu.
- **403 Forbidden**: token hợp lệ nhưng thiếu quyền — không thể tự khắc phục bằng retry.

---

## 8. Checklist Retry Policy cho Message Consumer (MassTransit)

- [ ] Đã cấu hình `cfg.UseMessageRetry(r => r.Interval(N, TimeSpan.FromSeconds(X)))` cho consumer chưa?
- [ ] Đã hiểu rõ: sau khi hết số lần retry, message tự động chuyển sang `_error` queue (DLQ) chưa?
- [ ] Có consumer/job riêng để log hoặc reprocess message trong error queue không?
- [ ] Đã kiểm tra hành vi DLQ khác biệt giữa các broker chưa? (RabbitMQ tự động, Azure Service Bus dùng `$DeadLetterQueue`, Amazon SQS cần cấu hình thủ công)

## 9. Checklist triển khai Outbox Pattern

- [ ] Đã tạo entity `OutboxMessage` (Id, EventType, Payload, OccurredOnUtc, Processed, ProcessedOnUtc) chưa?
- [ ] Đã thêm `DbSet<OutboxMessage>` vào DbContext chưa?
- [ ] Thao tác lưu business data + thêm outbox message có nằm trong CÙNG 1 transaction không?
- [ ] Đã implement `OutboxProcessor` (BackgroundService) quét message chưa xử lý theo batch (VD `Take(20)`) chưa?
- [ ] Publish thất bại trong `OutboxProcessor` có được xử lý (retry/log) mà không đánh dấu `Processed = true` sai không?
- [ ] Đã đăng ký `AddHostedService<OutboxProcessor>()` chưa?

---

## 10. So sánh DLQ giữa các Message Broker

| Broker | Cách MassTransit xử lý DLQ |
|---|---|
| **RabbitMQ** | Tự động tạo `_error` queue |
| **Azure Service Bus** | Có DLQ native (`$DeadLetterQueue` subqueue), MassTransit cũng mô phỏng `_error` queue |
| **Amazon SQS** | Phải cấu hình DLQ thủ công trong AWS; visibility timeout/max receive count do AWS quản lý, không phải MassTransit |

---

## 11. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Resilience | Khả năng phục hồi của hệ thống trước lỗi |
| Transient Fault | Lỗi tạm thời, thường tự khắc phục |
| Circuit Breaker | Cầu dao ngắt mạch — ngăn gọi tiếp service đang lỗi |
| Exponential Backoff | Tăng dần khoảng cách giữa các lần retry theo cấp số nhân |
| Bulkhead Isolation | Cô lập tài nguyên để lỗi ở 1 phần không lan sang phần khác |
| Distributed Cache | Cache tập trung dùng chung bởi nhiều service |
| Stale Data | Dữ liệu lỗi thời, không còn phản ánh trạng thái mới nhất |
| Dead-Letter Queue (DLQ) | Hàng đợi chứa message xử lý thất bại sau nhiều lần retry |
| Delegating Handler | Interceptor cho HttpClient, dùng để enrich request (auth header...) |
| Idempotency | Tính chất xử lý N lần vẫn cho kết quả giống 1 lần |
| Deduplication Token | Token (message ID/correlation ID) dùng phát hiện message trùng |
| Outbox Pattern | Pattern đảm bảo atomicity giữa ghi DB và publish message |
| Dual Write Problem | Vấn đề ghi 2 hệ thống riêng biệt (DB + broker) không atomic |
| Graceful Degradation | Suy giảm nhẹ nhàng — hệ thống vẫn hoạt động một phần khi lỗi |

---

## 12. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Vì sao resiliency ở tầng ứng dụng vẫn cần thiết dù đã dùng cloud platform có tính năng resiliency hạ tầng?**
→ Vì các tính năng hạ tầng (Azure Application Gateway, AWS API Gateway...) chỉ cung cấp retry/timeout ở tầng networking — không bảo vệ khỏi lỗi logic hoặc lỗi tạm thời xảy ra bên trong logic ứng dụng hoặc giữa các service coupling chặt. Pattern tầng ứng dụng (Polly, circuit breaker, outbox...) mang lại kiểm soát chi tiết hơn về cách service hành xử khi gặp lỗi.

**Q2: Circuit Breaker khác Retry Policy ở điểm nào, và tại sao cần dùng cả hai?**
→ Retry Policy thử lại request khi gặp lỗi tạm thời, hy vọng thành công ở lần sau. Circuit Breaker giám sát tần suất lỗi và chủ động NGẮT các cuộc gọi tiếp theo khi service liên tục lỗi, tránh vòng lặp retry vô hạn có thể gây quá tải thêm cho service đã yếu (vô tình tạo DoS). Kết hợp cả hai: Retry xử lý lỗi ngắn hạn, Circuit Breaker bảo vệ hệ thống khỏi lỗi kéo dài.

**Q3: Outbox Pattern giải quyết Dual Write Problem như thế nào?**
→ Dual Write Problem xảy ra khi ghi database và publish message là 2 thao tác riêng biệt — nếu 1 thành công còn 1 thất bại, hệ thống rơi vào trạng thái không nhất quán. Outbox Pattern lưu message vào bảng outbox trong CÙNG transaction với business data (đảm bảo atomic ở tầng DB), sau đó một background worker (OutboxProcessor) đọc và publish message riêng biệt — nếu publish thất bại, message vẫn còn trong outbox để retry sau, không bị mất.

**Q4: Vì sao cần idempotency trong message consumer khi có retry policy?**
→ Vì retry có thể khiến cùng 1 message được xử lý nhiều lần (do mất ACK, network jitter khiến message được gửi lại). Nếu handler không idempotent, việc xử lý trùng có thể gây hậu quả nghiêm trọng (VD gửi email 2 lần, trừ tiền 2 lần). Giải pháp: dùng deduplication token (message ID/correlation ID), lưu ID đã xử lý trong DB/cache, kiểm tra trước khi xử lý.

**Q5: Distributed Cache mang lại lợi ích resiliency gì, và rủi ro là gì?**
→ Lợi ích: dùng làm fallback data source khi service gốc down, giảm tải database, tăng tốc phản hồi cho query lặp lại, tạo "ảo giác" hệ thống vẫn hoạt động bình thường cho người dùng cuối. Rủi ro: dữ liệu có thể trở nên stale (lỗi thời) nếu source service offline trong lúc dữ liệu gốc đã thay đổi — cần cân bằng giữa TTL hợp lý và độ phức tạp của cơ chế invalidate cache.

**Q6: Sự khác biệt giữa 401 Unauthorized và 403 Forbidden, và cách xử lý mỗi loại?**
→ 401 Unauthorized: xác thực thất bại (thiếu/sai token, hoặc token hết hạn) — có thể tự động khắc phục bằng cách lấy token mới và retry request. 403 Forbidden: xác thực THÀNH CÔNG (hệ thống nhận diện đúng client) nhưng client không có quyền truy cập resource — không thể tự khắc phục bằng retry, cần xem xét lại authorization/permission.

**Q7: Vì sao hành vi Dead-Letter Queue khác nhau tùy theo message broker (RabbitMQ, Azure Service Bus, Amazon SQS)?**
→ Vì MassTransit chỉ cung cấp abstraction thống nhất ở tầng API, nhưng hành vi thực tế phụ thuộc vào khả năng native của từng broker. RabbitMQ tự động tạo `_error` queue; Azure Service Bus có DLQ native (`$DeadLetterQueue`) mà MassTransit map vào; Amazon SQS không hỗ trợ nhiều pattern nâng cao, DLQ phải cấu hình thủ công trong AWS console, và các tham số như visibility timeout/max receive count do AWS quản lý chứ không phải MassTransit.

**Q8: Delegating Handler trong .NET dùng để làm gì trong ngữ cảnh resiliency/authentication?**
→ Là 1 interceptor cho phép enrich HTTP request trước khi nó được gửi đi — ví dụ tự động đính kèm API key header hoặc JWT Bearer token vào mọi request từ 1 HttpClient cụ thể. Điều này tập trung hóa logic authentication, giảm rủi ro developer quên đính kèm header cần thiết ở một số cuộc gọi, và giảm khả năng lỗi 401 do thiếu header.

---

## 13. Bài tập thực hành đề xuất

1. Cấu hình Polly retry policy + circuit breaker cho 1 HttpClient gọi service giả lập trả lỗi ngẫu nhiên 50% thời gian, quan sát log khi circuit breaker chuyển từ Closed → Open → Closed.
2. Implement Redis distributed cache cho 1 endpoint GET, đo thời gian phản hồi giữa cache hit và cache miss.
3. Viết Delegating Handler tự động refresh JWT khi nhận response 401, retry request 1 lần với token mới.
4. Implement Outbox Pattern đầy đủ: tạo `OutboxMessage` entity, sửa 1 endpoint POST để ghi outbox trong cùng transaction, viết `OutboxProcessor` BackgroundService xử lý message pending.
5. Thiết lập MassTransit retry policy + kiểm tra hành vi DLQ: cố tình throw exception trong consumer, xác nhận message xuất hiện đúng trong `_error` queue sau khi hết số lần retry.

---

*Ghi chú: Chương tiếp theo sẽ implement API Gateway và BFF (Backend for Frontend) Gateway Pattern.*
