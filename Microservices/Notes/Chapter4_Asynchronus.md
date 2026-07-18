# Chapter 4 — Asynchronous Communication: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Asynchronous Communication** | Service gửi message không cần chờ phản hồi ngay, tách rời (decoupled) thời gian xử lý giữa các bên |
| **Message Queue** | Hàng đợi một-đối-một, đảm bảo thứ tự FIFO, mỗi message chỉ xử lý bởi 1 consumer |
| **Message Bus / Pub-Sub** | Một message được nhiều subscriber cùng nhận, publisher không cần biết ai đang lắng nghe |
| **Command Message** | Yêu cầu 1 hành động cụ thể xảy ra, hướng tới 1 service cụ thể, thì hiện tại/mệnh lệnh |
| **Event Message** | Thông báo hành động đã xảy ra, có thể gửi nhiều service, thì quá khứ |
| **Eventual Consistency** | Chấp nhận dữ liệu tạm thời không đồng bộ giữa các service, cuối cùng sẽ nhất quán |
| **CAP Theorem** | Không thể đồng thời đảm bảo cả Consistency, Availability, Partition tolerance |
| **Idempotency** | Xử lý cùng 1 message nhiều lần vẫn cho kết quả giống hệt, không gây tác dụng phụ |
| **Correlation ID** | ID dùng để lần theo đường đi của 1 message/request xuyên suốt nhiều service |

---

## 2. Ba mô hình giao tiếp bất đồng bộ

| Mô hình | Quan hệ | Đặc điểm | Use case điển hình |
|---|---|---|---|
| **Message Queue** | 1-đối-1 | FIFO, xử lý đúng 1 lần, polling model | Hệ thống thanh toán, cần giữ đúng thứ tự |
| **Pub-Sub (Message Bus)** | 1-đối-nhiều | Tách rời triệt để, nhiều consumer độc lập | Event-driven architecture, thông báo cho nhiều bên |
| **Event Streaming** | Luồng liên tục | Xử lý real-time | Kafka — phân tích dữ liệu thời gian thực |

---

## 3. CAP Theorem — bảng ôn nhanh

| Thuộc tính | Ý nghĩa |
|---|---|
| **Consistency (C)** | Mọi lần đọc trả về dữ liệu mới nhất, nếu không đảm bảo được thì báo lỗi |
| **Availability (A)** | Luôn trả về dữ liệu cho request đọc, dù không phải bản mới nhất |
| **Partition tolerance (P)** | Hệ thống vẫn hoạt động dù có lỗi kết nối tạm thời giữa các phần |

**Ghi nhớ**: Trong hệ phân tán, Partition tolerance gần như bắt buộc phải chấp nhận (mạng không bao giờ hoàn hảo) → thực chất phải chọn giữa **C và A**. Microservices ưu tiên Availability cao → chấp nhận Eventual Consistency thay vì Strong Consistency.

---

## 4. Message Delivery Guarantee — ba mức độ

| Mức độ | Đặc điểm | Rủi ro | Khi nào dùng |
|---|---|---|---|
| **At-most-once** | Gửi tối đa 1 lần | Có thể mất message | Chấp nhận mất mát, ưu tiên hiệu năng, không có retry logic |
| **At-least-once** | Không mất, có thể trùng | Duplicate message | Phổ biến nhất — billing, order processing (cần idempotent consumer) |
| **Exactly-once** | Không mất, không trùng | Khó implement nhất | Giao dịch tài chính cực kỳ quan trọng — Kafka hỗ trợ qua idempotent producer + transactional write |

---

## 5. Checklist: Event Definition tốt cần có gì?

- [ ] **Event schema** rõ ràng — định nghĩa đầy đủ attribute và kiểu dữ liệu
- [ ] **Event type** — phân loại rõ để service chỉ subscribe đúng loại cần
- [ ] **Event versioning** — chiến lược đảm bảo backward compatibility khi schema thay đổi
- [ ] **Event metadata** — timestamp, source identifier, correlation ID để trace và debug

---

## 6. Checklist triển khai RabbitMQ + MassTransit trong .NET

- [ ] Message class có được định nghĩa rõ ràng (class/record/interface theo yêu cầu MassTransit) không?
- [ ] `AddMassTransit` + `UsingRabbitMq()` đã đăng ký đúng trong `Program.cs` chưa?
- [ ] Publisher: đã lưu database **trước**, publish event **sau** chưa? (Core operation không chờ event xử lý xong)
- [ ] Consumer: có implement đúng `IConsumer<T>` với method `Consume` không?
- [ ] Consumer và publisher dùng chung namespace/shared library nếu khác project không?
- [ ] Có cân nhắc tách riêng `NotificationsService` thay vì để 1 service làm 2 việc (vi phạm single responsibility) không?

---

## 7. Checklist xử lý 2 cạm bẫy chính

### Message Ordering
- [ ] Event có field `Timestamp` (UTC) chưa?
- [ ] Consumer có lưu lại timestamp đã xử lý gần nhất cho từng entity (nên lưu DB/cache bền vững, không chỉ local variable) không?
- [ ] Có cân nhắc `PrefetchCount = 1` + `UseConcurrencyLimit(1)` khi cần giữ thứ tự nghiêm ngặt không? (Đánh đổi: giảm throughput)

### Duplicate Messages
- [ ] Event có `MessageId` duy nhất chưa?
- [ ] Consumer có check `ProcessedMessageIds` (nên lưu DB/cache bền vững) trước khi xử lý không?
- [ ] Message handler có được thiết kế idempotent (xử lý N lần vẫn ra kết quả giống 1 lần) không?

---

## 8. Bốn mối lo hạ tầng cấp cao (Additional Considerations)

| Mối lo | Vấn đề | Giải pháp |
|---|---|---|
| **Scalability** | Nghẽn khi publisher/subscriber tăng | Distributed broker, load balancing, partition topic, consumer group |
| **Fault tolerance** | Mất message/downtime khi 1 phần lỗi | Redundancy (nhiều instance), persistent storage cho message |
| **Security** | Truy cập trái phép vào message | Authentication/authorization cho topic, mã hóa in-transit và at-rest |
| **Logging & Tracing** | Khó debug do bản chất phân tán/bất đồng bộ | Correlation ID, centralized logging/monitoring |

---

## 9. So sánh Message Broker — bảng ôn nhanh cho phỏng vấn

| Broker | Ưu điểm | Nhược điểm |
|---|---|---|
| Apache Kafka | Throughput cao, real-time analytics/streaming | Setup/quản lý phức tạp, có thể quá mức cho nhu cầu đơn giản |
| Azure Service Bus | Tích hợp tốt với Azure, tính năng nâng cao | Ràng buộc hệ sinh thái Azure, chi phí |
| Amazon SQS | Dễ dùng, tích hợp AWS tốt | Giới hạn ở queue-based, thiếu tính năng nâng cao |
| NATS | Đơn giản, độ trễ thấp | Persistence hạn chế, ít tính năng |
| ActiveMQ | Đa giao thức, trưởng thành | Tốn tài nguyên, cần bảo trì nhiều |
| RabbitMQ | Đa giao thức, dễ cài đặt, hỗ trợ pub-sub/queue | Kém hiệu năng ở workload cao, không hỗ trợ event streaming |

---

## 10. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Asynchronous Communication | Giao tiếp bất đồng bộ — không chờ phản hồi ngay |
| Decoupling | Tách rời — giảm phụ thuộc trực tiếp giữa các service |
| Message Queue | Hàng đợi tin nhắn, một-đối-một, FIFO |
| Publish-Subscribe (Pub-Sub) | Mô hình một-đối-nhiều, publisher không biết ai đang lắng nghe |
| Message Broker | Hệ thống trung gian định tuyến message (VD RabbitMQ, Kafka) |
| Command Message | Message yêu cầu hành động cụ thể xảy ra |
| Event Message | Message thông báo hành động đã xảy ra |
| Eventual Consistency | Nhất quán cuối cùng — dữ liệu đồng bộ sau một khoảng thời gian |
| CAP Theorem | Định lý CAP — Consistency, Availability, Partition tolerance |
| Idempotent Handler | Handler xử lý message trùng lặp mà không gây lỗi/tác dụng phụ |
| At-most-once / At-least-once / Exactly-once | Ba mức đảm bảo gửi message |
| Correlation ID | ID dùng để trace một luồng xuyên suốt nhiều service |
| Prefetch Count | Số lượng message consumer lấy cùng lúc từ queue |
| Polling Model | Mô hình consumer chủ động kiểm tra message theo chu kỳ |
| Exchange / Binding | Thành phần RabbitMQ định tuyến message tới queue |

---

## 11. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Khi nào nên dùng giao tiếp bất đồng bộ thay vì đồng bộ?**
→ Khi thao tác không cần phản hồi ngay lập tức cho người dùng — ví dụ gửi email xác nhận, ghi log, cập nhật calendar. Thao tác "core" (người dùng cần biết kết quả ngay) nên xử lý đồng bộ; thao tác "phụ trợ" nên tách ra xử lý bất đồng bộ để tránh chặn trải nghiệm người dùng.

**Q2: Phân biệt Message Queue và Pub-Sub.**
→ Message Queue là mô hình một-đối-một: mỗi message chỉ được xử lý bởi đúng 1 consumer, đảm bảo FIFO. Pub-Sub là mô hình một-đối-nhiều: một message có thể được nhiều subscriber độc lập cùng nhận và xử lý, phù hợp cho event-driven architecture.

**Q3: Eventual Consistency là gì, và vì sao microservices phải chấp nhận nó?**
→ Là việc chấp nhận dữ liệu giữa các service có thể tạm thời không đồng bộ, nhưng cuối cùng sẽ nhất quán. Cần thiết vì theo CAP theorem, hệ phân tán không thể đảm bảo cả Consistency, Availability, và Partition tolerance cùng lúc — và microservices luôn ưu tiên Availability + phải chấp nhận Partition tolerance (do mạng không hoàn hảo), nên buộc phải hy sinh Strong Consistency.

**Q4: Vì sao cần thiết kế idempotent message handler?**
→ Vì hầu hết message broker dùng at-least-once delivery — message có thể bị gửi lại (do mất acknowledgment, timeout...). Nếu handler không idempotent, xử lý message trùng lặp có thể gây hậu quả nghiêm trọng (gửi email 2 lần, trừ tiền 2 lần). Idempotent handler đảm bảo dù nhận message N lần, kết quả vẫn giống như xử lý 1 lần.

**Q5: Làm sao xử lý vấn đề message không đúng thứ tự trong hệ thống pub-sub?**
→ Gắn timestamp (hoặc sequence number) vào message. Consumer lưu lại timestamp đã xử lý gần nhất cho từng entity (nên lưu bền vững, không chỉ biến local), chỉ xử lý message có timestamp mới hơn. Có thể giới hạn concurrency (`PrefetchCount = 1`, `UseConcurrencyLimit(1)`) để giữ thứ tự nghiêm ngặt hơn, đánh đổi với throughput thấp hơn.

**Q6: Command Message và Event Message khác nhau thế nào?**
→ Command message yêu cầu một hành động cụ thể xảy ra, hướng tới đúng 1 service, dùng thì hiện tại/mệnh lệnh (VD: "tạo mục lịch"). Event message chỉ thông báo rằng một hành động đã xảy ra, có thể gửi tới nhiều service, dùng thì quá khứ (VD: `AppointmentCreated`).

**Q7: Vì sao không nên để 1 service vừa xử lý nghiệp vụ chính vừa dispatch notification?**
→ Vi phạm nguyên tắc single responsibility của microservices — service đang gánh 2 trách nhiệm khác nhau. Cách tiếp cận chuẩn hơn là tách một `NotificationsService` trung tâm phục vụ chung cho mọi service cần gửi thông báo, giúp giảm trách nhiệm từng service và cho phép xử lý bất đồng bộ thực sự trên toàn hệ thống.

**Q8: Tại sao at-least-once là mức delivery guarantee phổ biến nhất, thay vì exactly-once?**
→ Vì exactly-once cực kỳ khó implement, thường đòi hỏi distributed transaction hoặc cơ chế phối hợp phức tạp giữa broker và consumer. At-least-once dễ đạt được hơn và chỉ cần consumer tự đảm bảo idempotency để xử lý trùng lặp an toàn — cân bằng tốt giữa độ phức tạp và độ tin cậy.

---

## 12. Bài tập thực hành đề xuất

1. Dựng thử RabbitMQ bằng Docker, tạo một publisher đơn giản gửi event `OrderCreated` và một consumer log ra console khi nhận được.
2. Implement idempotent consumer: thêm `MessageId` vào event, dùng `ConcurrentDictionary` (hoặc Redis nếu muốn thực tế hơn) để phát hiện và bỏ qua message trùng lặp.
3. Thử nghiệm `PrefetchCount` và `UseConcurrencyLimit`: đo throughput khi giới hạn xử lý tuần tự (1 message/lần) so với xử lý song song, rút ra nhận xét về đánh đổi giữa thứ tự và hiệu năng.
4. Vẽ sơ đồ CAP theorem cho hệ thống bạn đang làm: xác định hệ thống hiện tại đang nghiêng về Consistency hay Availability, và lý giải vì sao lựa chọn đó phù hợp (hoặc không phù hợp) với nghiệp vụ.

---

*Ghi chú: Chương tiếp theo (Chapter 5) sẽ tiếp tục khám phá các pattern thiết kế nâng cao hơn cho hệ thống microservices.*
