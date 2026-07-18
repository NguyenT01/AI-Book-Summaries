# Chapter 7 — Event Sourcing Pattern: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Event Sourcing** | Lưu trạng thái hệ thống dưới dạng chuỗi event bất biến, thay vì chỉ lưu trạng thái hiện tại |
| **Event** | Bản ghi bất biến của một sự việc đã xảy ra, thường là kết quả xử lý thành công 1 command |
| **Event Store** | Nguồn lưu trữ có thứ tự, dễ truy vấn, bền vững, append-only cho các event |
| **Replaying** | Quá trình lấy event và tái dựng lại trạng thái của aggregate |
| **Aggregate ID** | Định danh gắn event với 1 entity/process cụ thể (không nhất thiết chỉ dùng trong DDD) |
| **VersionNumber** | Số đại diện phiên bản trạng thái của aggregate, tăng mỗi khi có event mới |
| **Snapshot** | Bản lưu trạng thái aggregate tại 1 thời điểm, giúp giảm số event cần replay |
| **Optimistic Concurrency Control** | Kỹ thuật so sánh version mong đợi với version hiện tại trước khi ghi, trong 1 transaction |
| **INotification / Publish** | Cơ chế MediatR cho domain event — 1 event có thể có NHIỀU handler (khác `IRequest`/`Send`) |
| **Retention Policy / Archiving** | Chính sách giới hạn thời gian giữ event trong live store, di chuyển event cũ sang lưu trữ khác |

---

## 2. Vấn đề Event Sourcing giải quyết

| Vấn đề | Nguyên nhân | Giải pháp Event Sourcing |
|---|---|---|
| Mỗi service có DB riêng (Chapter 1) | Không còn ACID transaction toàn cục | Lưu mọi thay đổi dưới dạng event, dùng để đồng bộ/tái dựng dữ liệu |
| CQRS tách đọc/ghi (Chapter 6) | Rủi ro data không nhất quán giữa 2 model | Event store làm nguồn sự thật duy nhất, phục vụ cả ghi lẫn đọc |
| Cần audit trail | Khó biết "tại sao" dữ liệu ở trạng thái hiện tại | Event lưu lại toàn bộ lịch sử — trả lời được "why", không chỉ "what" |

---

## 3. Sáu thuộc tính chuẩn của Event

| Thuộc tính | Vai trò |
|---|---|
| Event name | Tên mô tả loại event |
| Timestamp | Thời điểm chính xác xảy ra |
| Event ID | Định danh duy nhất cho event |
| Aggregate ID | Gắn event với entity/process cụ thể |
| Payload | Dữ liệu chi tiết của event |
| Metadata | Ngữ cảnh bổ sung (nguồn, version schema) |

**Nguyên tắc bất biến**: Event đại diện sự thật đã xảy ra — business logic KHÔNG BAO GIỜ được sửa event sau khi ghi.

---

## 4. Năm lợi ích cốt lõi — bảng ôn nhanh

| Lợi ích | Giải thích ngắn |
|---|---|
| Auditability | Tái tạo lịch sử hoàn chỉnh — phục vụ compliance, troubleshooting |
| Decoupling | Producer không cần biết consumer — qua message broker (Chapter 4) |
| Scalability | Phân phối xử lý event qua nhiều consumer, scale theo nhu cầu |
| Reproducibility | Event bất biến → replay để tái tạo trạng thái tại bất kỳ thời điểm |
| Flexibility in data modeling | Lưu "why" đằng sau thay đổi, không chỉ "what" |

---

## 5. Bốn thách thức chính — checklist trước khi áp dụng

- [ ] Đã có chiến lược quản lý thay đổi cấu trúc event (versioning, backward compatibility) chưa?
- [ ] Đã tính tới việc query trạng thái hiện tại phức tạp hơn (không lưu trực tiếp) chưa?
- [ ] Đã có cơ chế retry/replay để xử lý lỗi và đảm bảo consistency chưa?
- [ ] Đã có kế hoạch dùng snapshot để tránh replay hàng nghìn event chưa?

---

## 6. Replaying vs Command — phân biệt quan trọng

| | Command | Replay |
|---|---|---|
| Bản chất | Thao tác GHI, thay đổi dữ liệu | Thao tác ĐỌC, tái dựng trạng thái |
| Có thể lặp lại tùy ý? | Không nên (side-effect, có thể chạy dài) | Có — chỉ đọc lại event đã lưu |
| Mục đích | Tạo ra event mới | Dùng event có sẵn để dựng lại trạng thái |

---

## 7. Hai lựa chọn thiết kế Event Store

| Tiêu chí | Relational Database | Non-Relational (Document) Database |
|---|---|---|
| Ưu điểm | Nghiêm ngặt, dễ enforce chất lượng dữ liệu | Schema linh hoạt, dữ liệu lồng nhau, dễ tiến hóa |
| Nhược điểm | Ít linh hoạt, dễ rơi vào "rabbit hole" nhiều bảng | Ít ràng buộc chất lượng dữ liệu hơn |
| Cách làm | 1 bảng log duy nhất (Id, AggregateId, Timestamp, EventType, Payload, VersionNumber) | Document JSON, có thể gồm cả snapshot + history trong 1 document |
| Ví dụ công nghệ | PostgreSQL (JSONB), MySQL (JSON), SQL Server (NVARCHAR+JSON funcs), Oracle (JSON 21c+) | MongoDB, Cosmos DB, DynamoDB |

**Tối ưu chung cho cả 2**: compound/text index, partition theo Aggregate ID, paginated event reading.

---

## 8. Checklist triển khai Event Store với PostgreSQL + EF Core

- [ ] Đã cài `Npgsql.EntityFrameworkCore.PostgreSQL` chưa?
- [ ] `EventEntity` có đánh dấu `Payload` với `[Column(TypeName = "jsonb")]` chưa?
- [ ] `EventId` có đánh dấu `[Key]` làm primary key chưa?
- [ ] Connection string có lưu qua environment variable/secret vault (không hardcode) chưa?
- [ ] Đã chạy đúng 2 lệnh migration: `dotnet ef migrations add ... --context EventStoreDbContext` và `dotnet ef database update --context EventStoreDbContext` chưa?

## 9. Checklist Domain Event với MediatR

- [ ] Domain event class có kế thừa `INotification` (KHÔNG phải `IRequest`) không?
- [ ] Handler có implement `INotificationHandler<T>` với method `Handle` không?
- [ ] Controller có dùng `_mediator.Publish(...)` (không phải `Send`) để phát domain event không?
- [ ] Có phân biệt rõ: notification/event nên đặt gần Commands hay tạo folder `EventHandlers` riêng theo quy ước team?
- [ ] Payload event có đủ thông tin cần thiết không (tránh chỉ gửi ID rồi bắt handler tự query lại)?
- [ ] Việc lưu event vào event store có nằm trong cùng transaction với thao tác chính không?

---

## 10. `IRequest`/`Send` (CQRS) vs `INotification`/`Publish` (Event) — bảng so sánh

| Tiêu chí | IRequest + Send (Chapter 6) | INotification + Publish (Chapter 7) |
|---|---|---|
| Số lượng handler | Đúng 1 handler | Có thể nhiều handler cùng lắng nghe |
| Có phản hồi? | Có (command/query trả về giá trị) | Không (chỉ broadcast, không mong đợi response) |
| Dùng cho | Command (ghi) / Query (đọc) | Domain event — thông báo việc đã xảy ra |
| Ví dụ | `CreateAppointmentCommand`, `GetAppointmentsQuery` | `AppointmentCreatedEvent`, `AppointmentCancelledEvent` |

---

## 11. Năm Best Practices — bảng ôn nhanh

| Best Practice | Nội dung chính |
|---|---|
| **Thiết kế event cụ thể** | Dùng `AppointmentScheduled` thay vì `AppointmentUpdated` chung chung |
| **Forward/Backward Compatibility** | Field `Version` tường minh; thay đổi chỉ nên additive (không xóa/đổi tên field cũ) |
| **Consistency + Idempotency** | Event lưu atomic cùng DB chính; handler phải idempotent; dùng optimistic concurrency (version check trong transaction) |
| **Snapshot** | Lưu trạng thái tại mốc (VD sau 100 event), giảm chi phí replay; đánh đổi giữa tốc độ và dung lượng lưu trữ |
| **Chọn đúng Data Store/ORM** | PostgreSQL+EF Core (linh hoạt, hạn chế query JSON) vs EventStoreDB (chuyên biệt, cộng đồng nhỏ hơn) |

---

## 12. Quy trình Snapshot — 4 bước

1. Định nghĩa cấu trúc snapshot (`AggregateId`, `Version`, `PayloadSnapshot`, `CreatedAt`)
2. Xác định tần suất tạo (VD: sau mỗi 100 event)
3. Theo dõi số event kể từ snapshot cuối, tạo mới khi đạt ngưỡng
4. Khi load aggregate: lấy snapshot gần nhất trước → replay chỉ event sau đó

**Đánh đổi**: snapshot thường xuyên = tái dựng nhanh hơn, nhưng tốn storage, ảnh hưởng backup/recovery, cần cleanup snapshot cũ.

---

## 13. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Event Sourcing | Lưu trạng thái dưới dạng chuỗi event bất biến |
| Immutable | Bất biến — không thể sửa đổi sau khi tạo |
| Append-only | Chỉ cho phép thêm, không sửa/xóa |
| Replaying | Tái dựng trạng thái bằng cách đọc lại chuỗi event |
| Aggregate ID | Định danh gắn event với 1 entity/process |
| Optimistic Concurrency | Kiểm soát tranh chấp bằng so sánh version trước khi ghi |
| Snapshot | Bản lưu trạng thái tại 1 thời điểm để tối ưu replay |
| Denormalized Table | Bảng lưu dữ liệu liên quan dạng đầy đủ, không chỉ ID tham chiếu |
| JSONB | Kiểu cột JSON dạng nhị phân tối ưu của PostgreSQL |
| Materialized View | Document/view chứa cả snapshot hiện tại lẫn lịch sử event |
| Retention Policy | Chính sách giới hạn thời gian giữ dữ liệu trong live store |
| Archiving | Di chuyển/nén dữ liệu cũ sang lưu trữ tiết kiệm chi phí hơn |
| Idempotent Handler | Handler xử lý event nhiều lần vẫn cho kết quả giống nhau |
| Forward/Backward Compatibility | Khả năng hệ thống mới/cũ vẫn hiểu được event của phiên bản khác |

---

## 14. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Event Sourcing khác gì so với cách lưu trữ dữ liệu truyền thống?**
→ Truyền thống chỉ lưu trạng thái hiện tại (current state), mỗi lần update sẽ ghi đè giá trị cũ. Event Sourcing lưu toàn bộ chuỗi sự kiện bất biến đã dẫn tới trạng thái đó — trạng thái hiện tại được tính toán bằng cách replay lại toàn bộ (hoặc một phần) event. Điều này cho phép audit trail đầy đủ, khả năng tái tạo trạng thái tại bất kỳ thời điểm nào, và không mất thông tin về "tại sao" dữ liệu thay đổi.

**Q2: Vì sao Event Sourcing thường được dùng cùng với CQRS?**
→ CQRS tách model đọc và ghi, tạo ra rủi ro không nhất quán dữ liệu giữa 2 phía. Event Sourcing giải quyết vấn đề này bằng cách coi event store là nguồn sự thật duy nhất (source of truth) — command sinh ra event, event vừa cập nhật write model vừa có thể trực tiếp phục vụ read model của CQRS, giảm nhu cầu đồng bộ thủ công giữa 2 data store.

**Q3: Optimistic Concurrency Control trong Event Sourcing hoạt động như thế nào?**
→ Khi ghi 1 event mới, hệ thống load version hiện tại của aggregate, so sánh với version mong đợi (expected version) mà client gửi lên. Nếu khớp, event mới được insert và version tăng lên; nếu không khớp (do có writer khác đã ghi trước), thao tác bị từ chối để tránh race condition. Toàn bộ quá trình đọc-so sánh-ghi này nằm trong 1 database transaction duy nhất.

**Q4: Tại sao cần Snapshot trong hệ thống Event Sourcing?**
→ Vì replay hàng nghìn event để dựng lại trạng thái của 1 aggregate có thể rất tốn kém về hiệu năng. Snapshot lưu sẵn trạng thái tại 1 mốc thời gian (VD sau mỗi 100 event) — khi cần dựng lại trạng thái, hệ thống chỉ cần lấy snapshot gần nhất rồi replay các event xảy ra SAU snapshot đó, giảm đáng kể số lượng event cần xử lý.

**Q5: Idempotency quan trọng thế nào đối với Event Handler trong Event Sourcing?**
→ Vì event có thể bị replay hoặc xử lý nhiều lần (do lỗi retry, network issue...), handler phải đảm bảo xử lý cùng 1 event nhiều lần vẫn cho ra đúng 1 kết quả, không gây side-effect trùng lặp (VD gửi email 2 lần, cộng tiền 2 lần). Đây là nguyên tắc idempotency đã học từ Chapter 4, áp dụng lại trong ngữ cảnh Event Sourcing.

**Q6: Sự khác biệt giữa `IRequest`/`Send` và `INotification`/`Publish` trong MediatR?**
→ `IRequest` kết hợp `Send` dùng cho command/query — luôn có đúng 1 handler xử lý và có thể trả về giá trị (dùng ở CQRS, Chapter 6). `INotification` kết hợp `Publish` dùng cho domain event — có thể có NHIỀU handler cùng lắng nghe 1 event, không mong đợi giá trị trả về, phù hợp cho mô hình pub-sub nội bộ trong 1 service.

**Q7: Vì sao PostgreSQL JSONB không phải giải pháp hoàn hảo cho Event Store phức tạp?**
→ Vì khả năng query/filter bên trong nội dung JSONB còn hạn chế — ví dụ cần lọc event có ngày tương lai buộc phải load toàn bộ event rồi filter ở tầng ứng dụng thay vì filter ngay trong câu SQL. Ngoài ra, Event Store vốn được tối ưu cho ghi (write-optimized); dùng nó cho query đọc phức tạp dễ gây nghẽn hiệu năng — nên dùng read model chuyên biệt (denormalized) cho nhu cầu đọc phức tạp thay vì query trực tiếp trên event store.

**Q8: Retention Policy và Archiving trong Event Sourcing là gì, và tại sao cần thiết?**
→ Retention Policy quy định khoảng thời gian giữ event trong event store chính (VD chỉ giữ 2-3 năm gần nhất). Event cũ hơn được archive — di chuyển/nén sang giải pháp lưu trữ rẻ hơn (blob storage, data lake, cold-tier DB). Điều này giữ event store chính gọn nhẹ, nhanh, đồng thời vẫn bảo tồn dữ liệu lịch sử cho audit/compliance/phân tích khi cần.

---

## 15. Bài tập thực hành đề xuất

1. Dựng thử Event Store với PostgreSQL + EF Core cho 1 aggregate đơn giản (VD `Order`), implement `OrderCreatedEvent`, `OrderShippedEvent` qua MediatR `INotification`.
2. Viết logic replay: từ danh sách event đã lưu cho 1 `AggregateId`, tái dựng lại trạng thái hiện tại của aggregate bằng cách lặp qua và áp dụng từng event theo thứ tự timestamp.
3. Implement optimistic concurrency: thêm cột `VersionNumber`, viết logic kiểm tra version trước khi insert event mới, xử lý trường hợp version conflict.
4. Thiết kế cơ chế snapshot cho aggregate có nhiều event (VD sau mỗi 50 event tạo 1 snapshot), đo thời gian dựng trạng thái có snapshot so với không có snapshot.
5. Thiết kế retention policy: viết job giả lập archive các event cũ hơn 1 năm sang 1 bảng "cold storage" riêng, giữ bảng chính gọn nhẹ.

---

*Ghi chú: Chương tiếp theo sẽ tiếp tục khám phá các pattern thiết kế khác giúp xây dựng hệ thống microservices vững chắc hơn (VD: Database Design Patterns for Microservices).*
