# Chapter 9 — Saga Pattern: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Saga** | Chuỗi các local transaction, mỗi cái cập nhật 1 database và sinh message/event kích hoạt bước tiếp theo |
| **Compensating Transaction** | Transaction bù trừ, đảo ngược hiệu ứng của 1 transaction trước đó khi có lỗi |
| **Choreography** | Điều phối phi tập trung — service tự trị lắng nghe/publish event, không có điểm kiểm soát trung tâm |
| **Orchestration** | Điều phối tập trung — 1 orchestrator phát command, lắng nghe event, quyết định bước tiếp theo |
| **Correlation ID** | ID gắn với 1 saga instance, đưa vào mọi message để truy vết end-to-end |
| **Watchdog / Process Monitor** | Service giám sát deadline, phát timeout event nếu message mong đợi không xuất hiện đúng hạn |
| **Transactional Outbox Pattern** | Ghi ý định publish event vào cùng transaction với business data, đảm bảo event không bị mất |
| **Saga State Machine** | Cơ chế (VD MassTransit) quản lý trạng thái tiến triển của 1 saga instance qua các bước |
| **Dead-letter Queue** | Hàng đợi chứa message thất bại sau nhiều lần retry, dùng để cảnh báo/xử lý thủ công |

---

## 2. Ba loại Transaction trong Saga

| Loại | Đặc điểm |
|---|---|
| **Reversible** | Có thể bị đảo ngược bởi 1 transaction khác có hiệu ứng ngược lại |
| **Retriable** | Đảm bảo sẽ thành công, thường đặt sau pivot transaction |
| **Pivot** | Thành/bại mang tính quyết định cho việc saga có tiếp tục không — có thể là compensating cuối hoặc retriable đầu |

---

## 3. So sánh Choreography vs Orchestration

| Tiêu chí | Choreography | Orchestration |
|---|---|---|
| Điểm kiểm soát | Không có — phi tập trung | Có — 1 orchestrator trung tâm |
| Coupling | Thấp, mỗi service tự trị | Service phải tuân theo command/event do orchestrator định nghĩa |
| Độ phức tạp khi tăng participant | Tăng nhanh — dễ thành "mớ event rối rắm" | Tăng chậm hơn — logic tập trung ở 1 nơi |
| Observability | Khó — cần distributed tracing mạnh (OpenTelemetry) | Dễ hơn — trạng thái tập trung trong state machine |
| Độ trễ | Có thể thấp hơn nhờ fan-out song song | Phụ thuộc luồng tuần tự do orchestrator điều khiển |
| Rủi ro | Race condition khi nhiều consumer phụ thuộc cùng event | Orchestrator trở thành single point of failure/bottleneck nếu thiết kế kém |
| Phù hợp khi | Luồng đơn giản, ít participant, ưu tiên latency/autonomy | Luồng phức tạp, nhiều bước, cần kiểm soát/debug rõ ràng |

---

## 4. Bốn nhược điểm cấu trúc/vận hành của Saga (chung cho cả 2 mô hình)

| Nhược điểm | Giải thích |
|---|---|
| Từ bỏ ACID → Eventual Consistency | Service downstream phải chịu đựng phân kỳ tạm thời, người dùng có thể đọc dữ liệu "đang xử lý" |
| Cần logic bổ sung | Versioning, read-model isolation, compensating flag để tránh hành động dựa trên dữ liệu lỗi thời |
| Tăng độ phức tạp tổng thể | Mỗi local transaction phải idempotent + tự hoàn tác; càng nhiều bước càng nhiều tổ hợp cần test |
| Tăng độ trễ | Mỗi bước chờ network/queue/persistence; nhiều saga đồng thời retry có thể gây "retry storm" |

---

## 5. Correlation ID — Ba mục đích thực tiễn

| Mục đích | Giải thích |
|---|---|
| Tracing & Observability | Distributed tracing (OpenTelemetry) trực quan hóa saga như 1 span chứa child span từng service |
| Idempotency & Deduplication | Consumer giữ record message đã xử lý theo ID, loại bỏ bản trùng |
| Business Auditability | Trả lời "chuyện gì đã xảy ra với transaction #X" bằng cách lưu correlation ID cùng mỗi thay đổi |

---

## 6. Hai kỹ thuật giảm thiểu rủi ro "message vắng mặt" trong Choreography

| Kỹ thuật | Cách hoạt động |
|---|---|
| **Watchdog/Process Monitor** | Đăng ký toàn bộ saga event, giữ deadline ledger theo correlation ID; nếu quá hạn, phát synthetic timeout event |
| **Transactional Outbox Pattern** | Ghi ý định publish event vào cùng DB transaction với business data; outbox dispatcher đảm bảo event luôn được gửi dù service crash |

**Lưu ý**: 2 kỹ thuật này thường được kết hợp để đạt at-least-once delivery + liveness.

---

## 7. Checklist: Compensation cũng cần được thiết kế cẩn thận

- [ ] Compensation (VD: refund) có retry policy với exponential backoff cho lỗi tạm thời chưa?
- [ ] Sau N lần retry thất bại, message có được chuyển vào dead-letter queue + cảnh báo không?
- [ ] Có "compensation outstanding flag" theo dõi SLA (VD 30 phút cho tài chính), escalate sang xử lý thủ công nếu quá hạn không?
- [ ] Compensation handler có idempotent không? (Dùng deduplication key = correlation ID + event type, lưu trong idempotency table)
- [ ] Nếu compensation gọi API bên ngoài (VD payment gateway), có dùng idempotency key ở tầng gateway không?

---

## 8. Checklist triển khai Choreography Saga (MassTransit + RabbitMQ)

- [ ] Đã tạo Shared class library chứa event model dùng chung (VD `AppointmentCreated`, `PaymentProcessed`) chưa?
- [ ] Mỗi service có reference tới Shared project chưa?
- [ ] Mỗi service đã cài `MassTransit.AspNetCore`, `MassTransit.RabbitMQ` chưa?
- [ ] Event có mang đủ dữ liệu cần thiết (không thừa, không thiếu) và `CorrelationId` chưa?
- [ ] Consumer xử lý lỗi (VD `PaymentProcessedConsumer` khi `Success = false`) có đánh dấu đúng trạng thái rollback (VD `IsCanceled = true`) không?
- [ ] Service downstream không liên quan (VD document upload) có KHÔNG đăng ký lắng nghe event lỗi không liên quan tới nó không?

## 9. Checklist triển khai Orchestration Saga (MassTransit Saga State Machine)

- [ ] Đã tách rõ Command (ý định — VD `PaymentRequested`) và Event (kết quả — VD `PaymentSucceeded`/`PaymentFailed`) chưa?
- [ ] Saga State (VD `BookingSagaState`) có kế thừa `SagaStateMachineInstance` và có `CorrelationId` làm primary key chưa?
- [ ] Đã đăng ký `AddSagaStateMachine<TStateMachine, TState>().EntityFrameworkRepository(...)` chưa?
- [ ] Database riêng cho saga state (VD PostgreSQL) đã cấu hình đúng connection string chưa?
- [ ] State machine đã định nghĩa đủ `Initially`, `During`, và `Finalize()` cho từng nhánh thành công/thất bại chưa?
- [ ] Đã cân nhắc `ConcurrencyMode` (VD `Pessimistic`) phù hợp với tải production chưa?
- [ ] Đã đánh giá: saga có thực sự cần cả state machine phức tạp, hay 1 orchestrator đơn giản + bảng track trạng thái là đủ?

---

## 10. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Saga | Chuỗi local transaction phối hợp thay thế cho ACID transaction toàn cục |
| Compensating Transaction | Giao dịch bù trừ, đảo ngược hiệu ứng của giao dịch trước |
| Choreography | Điều phối phi tập trung dựa trên event |
| Orchestration | Điều phối tập trung qua 1 bộ điều phối (orchestrator) |
| Correlation ID | Định danh liên kết mọi message của cùng 1 saga instance |
| Pivot Transaction | Giao dịch bản lề, quyết định saga có tiếp tục hay không |
| Transactional Outbox Pattern | Pattern đảm bảo atomicity giữa ghi DB và publish event |
| Watchdog Service | Dịch vụ giám sát deadline, phát hiện message bị thiếu |
| Dead-letter Queue | Hàng đợi chứa message thất bại sau nhiều lần retry |
| Idempotency Table | Bảng lưu dấu vết các message đã xử lý để tránh xử lý trùng |
| Saga State Machine | Cơ chế quản lý vòng đời/trạng thái của 1 saga instance |
| Fan-out | Phát 1 event khởi tạo tới nhiều consumer song song |
| Retry Storm | Hiện tượng nhiều saga đồng thời retry gây quá tải hệ thống |
| Two-Phase Commit (2PC) | Kỹ thuật transaction phân tán yêu cầu mọi data store commit/rollback đồng thời (không tương thích với NoSQL/message broker) |

---

## 11. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Saga Pattern giải quyết vấn đề gì trong kiến trúc microservices?**
→ Giải quyết vấn đề đảm bảo tính nhất quán dữ liệu khi 1 business transaction trải dài qua nhiều microservice, mỗi service có database riêng — nơi ACID transaction toàn cục không còn khả thi (two-phase commit không tương thích với nhiều NoSQL/message broker). Saga chia transaction lớn thành chuỗi local transaction, mỗi bước có compensating transaction để rollback khi cần.

**Q2: Phân biệt Choreography và Orchestration, khi nào chọn cái nào?**
→ Choreography: phi tập trung, mỗi service tự trị lắng nghe/publish event, không có điểm kiểm soát trung tâm — phù hợp luồng đơn giản, ít participant, ưu tiên latency và autonomy. Orchestration: tập trung qua 1 orchestrator phát command và lắng nghe event — phù hợp luồng phức tạp nhiều bước, cần kiểm soát/debug rõ ràng, dù tạo ra 1 điểm kiểm soát có thể thành bottleneck nếu thiết kế kém.

**Q3: Vì sao lỗi nguy hiểm nhất trong Choreography Saga không phải là 1 event thất bại tường minh?**
→ Vì saga choreography chỉ hoạt động đúng khi mọi participant publish event thành công/thất bại kịp thời. Lỗi nguy hiểm nhất là sự VẮNG MẶT hoàn toàn của message mong đợi (VD service crash sau khi commit DB nhưng trước khi publish event) — không service nào khác biết điều gì đã xảy ra, dẫn tới trạng thái treo lơ lửng. Cần watchdog service với deadline ledger để phát hiện và xử lý trường hợp này.

**Q4: Transactional Outbox Pattern giải quyết vấn đề gì trong Saga?**
→ Đảm bảo tính atomic giữa việc lưu business data và publish event thông báo. Nếu chỉ lưu DB rồi publish riêng, có rủi ro publish thất bại (network, crash) khiến hệ thống không nhất quán. Outbox Pattern ghi ý định publish vào CÙNG transaction với business data, một dispatcher chạy nền đảm bảo event luôn được gửi — ngăn mất event ngay cả khi service crash giữa chừng.

**Q5: Vì sao Compensation (VD refund) cần được thiết kế idempotent?**
→ Vì message có thể bị gửi trùng (do retry từ producer, network jitter khiến republish). Nếu compensation handler không idempotent, việc xử lý trùng có thể gây hậu quả nghiêm trọng (VD hoàn tiền 2 lần). Giải pháp: lưu deduplication key (correlation ID + event type) trong idempotency table, kiểm tra trước khi xử lý; với API bên ngoài, dùng thêm idempotency key ở tầng gateway.

**Q6: Correlation ID có vai trò gì trong Saga, và tại sao không thể thiếu?**
→ Correlation ID là "sợi chỉ" xuyên suốt gắn mọi command/event của 1 saga instance lại với nhau, phục vụ 3 mục đích: tracing/observability (theo dõi toàn bộ saga như 1 span), idempotency/deduplication (nhận diện message trùng theo ID), và business auditability (tái dựng lại toàn bộ câu chuyện của 1 giao dịch khi cần audit/debug). Không có nó, gần như không thể trace 1 business transaction xuyên suốt nhiều service/queue.

**Q7: Vì sao Orchestration Saga cần 1 database riêng để lưu trạng thái, thay vì giữ trong bộ nhớ?**
→ Vì saga instance có thể trải dài phút hoặc giờ, và orchestrator process có thể bị restart, scale out, hoặc failover trong lúc đó. Nếu chỉ giữ trạng thái in-memory, mọi tiến độ sẽ mất khi process khởi động lại. Durable storage (VD MassTransit EF repository) đảm bảo orchestrator có thể tiếp tục đúng chỗ đã dừng, xử lý message trùng/retry idempotently, và cung cấp audit trail đầy đủ.

**Q8: Nhược điểm chính của mô hình Orchestration là gì?**
→ Tạo ra 1 điểm kiểm soát trung tâm — nếu thiết kế kém, orchestrator có thể trở thành bottleneck hoặc single point of failure. Ngoài ra, dù giảm coupling trực tiếp giữa các service, mọi service vẫn phải tuân theo contract command/event do orchestrator định nghĩa — không hoàn toàn tự trị. Việc tích hợp chặt với framework cụ thể (VD MassTransit SagaStateMachine) cũng có thể khiến logic nghiệp vụ bị coupling với API của framework, nên cần cân nhắc tách logic nghiệp vụ thuần túy ra khỏi infrastructure concern.

---

## 12. Bài tập thực hành đề xuất

1. Implement 1 choreography saga đơn giản cho luồng "Order → Payment → Shipping": Order service publish `OrderCreated`, Payment service lắng nghe và publish `PaymentProcessed`, Shipping service chỉ xử lý khi `PaymentProcessed.Success = true`.
2. Thêm watchdog service giám sát saga trên: nếu không nhận được `PaymentProcessed` trong 2 phút sau `OrderCreated`, phát `OrderTimeout` event.
3. Implement idempotent compensation: thêm bảng `ProcessedEvents` lưu deduplication key (correlation ID + event type), kiểm tra trước khi thực hiện refund logic.
4. Chuyển luồng trên sang orchestration dùng MassTransit Saga State Machine — so sánh độ phức tạp code và khả năng debug giữa 2 cách tiếp cận.
5. Thiết kế transactional outbox table cho Payment service: viết pseudocode cho outbox dispatcher đọc event pending và publish chúng.

---

*Ghi chú: Chương tiếp theo sẽ khám phá các kỹ thuật xây dựng microservices có khả năng phục hồi (resilient) tốt hơn.*
