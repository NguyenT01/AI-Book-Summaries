# Chapter 6 — CQRS Pattern: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **CQS (Command-Query Segregation)** | Nguyên tắc cấp code: mỗi method chỉ nên thực hiện hành động (command) HOẶC trả dữ liệu (query), không cả hai |
| **CQRS (Command Query Responsibility Segregation)** | Mở rộng CQS lên cấp kiến trúc: tách model (và có thể cả data store) cho đọc và ghi |
| **Command** | Thao tác thay đổi trạng thái hệ thống (create, update, delete) |
| **Query** | Thao tác lấy dữ liệu, không thay đổi trạng thái |
| **Over-fetching** | Lấy dư thừa dữ liệu không cần thiết trong 1 request |
| **Under-fetching** | Lấy thiếu dữ liệu, phải gọi thêm request khác để bổ sung |
| **Denormalization** | Hợp nhất dữ liệu từ nhiều bảng/service thành 1 cấu trúc tối ưu cho đọc |
| **Mediator Pattern** | Behavioral pattern: các thành phần giao tiếp qua 1 mediator trung tâm, không phụ thuộc trực tiếp lẫn nhau |
| **MediatR** | Thư viện .NET phổ biến nhất để implement CQRS, dựa trên mediator pattern |
| **Vertical Slice Architecture** | Tổ chức code theo feature/domain thay vì theo loại kỹ thuật (command/query/handler) |

---

## 2. Ba lợi ích cốt lõi — bảng ôn nhanh

| Lợi ích | Giải thích |
|---|---|
| **Scalability** | Đọc thường nặng hơn ghi (join, filter nhiều) → scale đọc/ghi độc lập bằng data store riêng |
| **Performance** | Dùng caching, SQL tinh chỉnh riêng cho query mà model chung không cho phép |
| **Simplicity** | Mỗi model chỉ chịu trách nhiệm 1 thao tác → sửa đổi có tác động tối thiểu tới phần còn lại |

---

## 3. Hai mô hình kiến trúc CQRS

| Mô hình | Đặc điểm | Khi nào dùng |
|---|---|---|
| **1 database** | Command/Query tách model nhưng dùng chung DB | Phổ biến, dễ áp dụng, đủ cho hầu hết trường hợp |
| **2 database** | Write DB (transactional/ACID) + Read DB (denormalized, cache/NoSQL) | Cần scale đọc/ghi độc lập thực sự, chấp nhận thêm độ phức tạp |

---

## 4. Nhược điểm CQRS — checklist trước khi áp dụng

- [ ] Service này có phải CRUD đơn giản không? → Nếu có, KHÔNG nên dùng CQRS
- [ ] Business logic có đủ phức tạp để justify việc tách model không?
- [ ] Nếu dùng nhiều data store, đã có kế hoạch xử lý data consistency (event-sourcing, SLA) chưa?
- [ ] Đã đánh giá chi phí hạ tầng bổ sung (nhiều DB = nhiều điểm lỗi, cần monitoring nhiều hơn) chưa?
- [ ] Team có đủ kinh nghiệm để tránh over-engineering không?

---

## 5. Ứng viên nên/không nên dùng CQRS

| Nên dùng CQRS | Không nên dùng CQRS |
|---|---|
| Appointment Booking (cập nhật liên tục + view real-time tối ưu) | Authentication (login/token — atomic, đơn giản) |
| Patient Record Management (cần consistency khi sửa + truy xuất hiệu quả) | Notifications (gửi theo template, không cần model riêng) |
| Medical History (tách real-time aggregation khỏi reporting nặng) | Document Delivery (chỉ cung cấp file, không có logic phức tạp) |

**Tiêu chí đánh giá chung**: độ phức tạp business logic, tần suất/bản chất đọc-ghi, yêu cầu khả năng mở rộng.

---

## 6. Checklist triển khai MediatR trong .NET

- [ ] Đã cài `MediatR` và đăng ký `AddMediatR(cfg => cfg.RegisterServicesFromAssembly(...))` chưa?
- [ ] Nếu dự án đã dùng MassTransit, có kiểm tra trùng tên interface không (cả 2 đều có khái niệm tương tự)?
- [ ] Command model có kế thừa đúng `IRequest<TReturn>` (hoặc `IRequest` nếu không cần trả về) không?
- [ ] Handler có implement đúng `IRequestHandler<TCommand, TReturn>` với method `Handle` không?
- [ ] Nếu handler không cần trả về giá trị, có return `Unit.Value` đúng chuẩn MediatR không?
- [ ] Controller có inject `IMediator` và chỉ gọi `_mediator.Send(...)` thay vì tự viết business logic không?

---

## 7. Command vs Query — bảng so sánh nhanh

| Tiêu chí | Command | Query |
|---|---|---|
| Mục đích | Thay đổi trạng thái (create/update/delete) | Lấy dữ liệu, không đổi trạng thái |
| Kiểu trả về | Có thể có hoặc không (Unit.Value nếu không) | Luôn có (dữ liệu cần trả) |
| Đặt tên | Động từ mệnh lệnh: `CreateXCommand`, `UpdateXCommand` | `GetXQuery`, `GetXByIdQuery` |
| Ví dụ | `CreateAppointmentCommand` | `GetAppointmentsQuery`, `GetAppointmentByIdQuery` |

---

## 8. Best Practices — checklist tổ chức code

### Naming Convention
- [ ] Command dùng động từ mệnh lệnh (`CreateAppointmentCommand`, `UpdatePatientCommand`)
- [ ] Query bắt đầu bằng `Get` (`GetPatientsQuery`, `GetAppointmentByIdQuery`)
- [ ] Tên class/file cho biết ngay thuộc phía Command hay Query mà không cần đọc code

### Tổ chức file/folder — 2 cách tiếp cận
| Cách | Mô tả |
|---|---|
| **Tách theo loại kỹ thuật** | `Commands/`, `Queries/` — mỗi cái có subfolder riêng cho từng thao tác |
| **Vertical Slice (group theo feature)** | `Features/Appointments/Commands/`, `Features/Appointments/Queries/` — căn chỉnh với DDD, dễ điều hướng hơn khi dự án lớn |

### SRP vs số lượng file
- [ ] Dự án nhỏ/vừa: tuân thủ SRP nghiêm ngặt (1 class/file)
- [ ] Dự án lớn, nhiều file gây khó điều hướng: cân nhắc gộp model + handler + DTO vào cùng 1 file (đánh đổi có chủ đích, quyết định ở cấp team)

---

## 9. Bốn kỹ thuật tăng tốc thao tác đọc — bảng ôn nhanh

| Kỹ thuật | Cách dùng | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **EF Core No-Tracking** | `.AsNoTracking()` | Giảm memory, tăng tốc query chỉ đọc | Không dùng được nếu cần update entity sau đó |
| **Denormalized Data Store** | DB/table/materialized view riêng, tối ưu cho query | Giảm chi phí join/API call lúc runtime | Cần code đồng bộ + job nền, thêm độ phức tạp |
| **ADO.NET** | Raw SQL qua `SqlConnection`/`SqlCommand` | Kiểm soát hoàn toàn SQL, hiệu năng cao nhất | Nhiều boilerplate code |
| **Dapper** | Micro-ORM, `connection.QueryAsync<T>()` | Cân bằng hiệu năng/dễ dùng, ít abstraction hơn EF Core | Vẫn cần viết SQL thủ công |

---

## 10. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Command-Query Segregation (CQS) | Nguyên tắc cấp code: method chỉ hành động HOẶC trả dữ liệu |
| Command Query Responsibility Segregation (CQRS) | Mở rộng CQS ở cấp kiến trúc, tách model đọc/ghi |
| Over-fetching / Under-fetching | Lấy dư/lấy thiếu dữ liệu so với nhu cầu thực tế |
| Denormalization | Hợp nhất dữ liệu chuẩn hóa thành cấu trúc tối ưu cho đọc |
| Mediator Pattern | Pattern hành vi: giao tiếp qua 1 trung gian, giảm coupling |
| In-process Messaging | Giao tiếp giữa các thành phần trong cùng 1 process (không qua network) |
| Materialized View | View được lưu trữ vật lý, pre-aggregate sẵn để tăng tốc query |
| Micro-ORM | ORM nhẹ, ít abstraction hơn ORM đầy đủ (VD: Dapper so với EF Core) |
| Vertical Slice Architecture | Tổ chức code theo feature end-to-end thay vì theo layer kỹ thuật |
| Event Sourcing | Kỹ thuật lưu trạng thái dưới dạng chuỗi sự kiện, hỗ trợ đồng bộ giữa nhiều data store trong CQRS đầy đủ |
| Service-Level Agreement (SLA) | Cam kết về mức độ dịch vụ, ở đây dùng để thông báo độ trễ đồng bộ dữ liệu |

---

## 11. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: CQRS khác CQS ở điểm nào?**
→ CQS là nguyên tắc ở cấp code: mỗi method chỉ nên hành động (command) hoặc trả dữ liệu (query), không cả hai — cải thiện thiết kế method-level. CQRS mở rộng nguyên tắc này lên cấp kiến trúc: tách hẳn model, thậm chí cả data store, cho đọc và ghi — giải quyết vấn đề ở quy mô hệ thống lớn, không chỉ ở method riêng lẻ.

**Q2: Khi nào KHÔNG nên dùng CQRS?**
→ Khi service chỉ thực hiện CRUD đơn giản, không có business logic phức tạp, không có nhu cầu tách biệt rõ ràng giữa đọc/ghi. Ví dụ: authentication service (login/token validation là thao tác atomic), notification service (gửi theo template cố định). Áp dụng CQRS trong các trường hợp này chỉ tạo thêm độ phức tạp không cần thiết (over-engineering).

**Q3: Vì sao thao tác đọc thường nặng hơn thao tác ghi trong hệ thống thực tế?**
→ Vì một lần đọc thường cần dữ liệu từ nhiều bảng, dẫn tới join và filter phức tạp, làm giảm tốc độ truy xuất. Trong khi đó, thao tác ghi thường chỉ tác động tới một phạm vi dữ liệu cụ thể, atomic hơn. Đây là lý do CQRS khuyến khích tối ưu riêng biệt cho đọc — như dùng data store denormalized hoặc caching.

**Q4: MediatR giải quyết vấn đề gì trong việc implement CQRS?**
→ MediatR implement mediator design pattern, cho phép controller gửi command/query qua một mediator trung tâm (`IMediator.Send()`) thay vì tự gọi trực tiếp business logic. Điều này decouple handler khỏi bên gọi, giúp controller trở nên mỏng, business logic được cô lập trong handler riêng biệt, dễ test và bảo trì hơn.

**Q5: Sự khác biệt giữa 2 cách tổ chức file trong CQRS: tách theo loại kỹ thuật vs vertical slice architecture?**
→ Tách theo loại kỹ thuật tạo folder `Commands/` và `Queries/` riêng biệt ở cấp cao nhất — rõ ràng về mặt kỹ thuật nhưng khi dự án lớn, các file liên quan tới cùng 1 feature bị rải rác. Vertical slice architecture (group theo feature) tổ chức theo domain/feature trước, sau đó mới chia command/query bên trong — căn chỉnh tốt hơn với DDD (Chapter 2), dễ điều hướng khi cần sửa đổi 1 feature cụ thể.

**Q6: Denormalization trong CQRS là gì, và cái giá phải trả là gì?**
→ Là việc hợp nhất dữ liệu từ nhiều bảng/nguồn thành 1 cấu trúc tối ưu sẵn cho việc đọc (database, table, hoặc materialized view riêng). Lợi ích: giảm đáng kể chi phí tính toán của join/API call/aggregation lúc runtime. Cái giá: cần thêm code đồng bộ và job chạy nền để đảm bảo read-only data store luôn phản ánh đúng trạng thái hiện tại — đây là nguồn phức tạp lớn nhất của CQRS đầy đủ (2 data store).

**Q7: So sánh EF Core, Dapper, và ADO.NET trong ngữ cảnh tối ưu query của CQRS.**
→ EF Core là ORM đầy đủ, dễ dùng nhưng có overhead abstraction (materialization, tracking). ADO.NET là API mức thấp nhất, cho kiểm soát SQL hoàn toàn, hiệu năng cao nhất nhưng nhiều boilerplate. Dapper là micro-ORM, điểm cân bằng giữa 2 cái — vẫn viết SQL thủ công nhưng tự động map sang object .NET, ít abstraction hơn EF Core nên phù hợp cho phía query cần hiệu năng cao với join/projection phức tạp.

**Q8: Tại sao `AsNoTracking()` trong EF Core lại giúp tăng tốc thao tác đọc?**
→ Mặc định EF Core tracking mọi entity lấy từ database để phát hiện thay đổi (change tracking), phục vụ cho việc update sau này. Với query chỉ đọc (không cần update), việc tracking này là overhead không cần thiết — tốn thêm memory và thời gian xử lý. `AsNoTracking()` tắt cơ chế này, giảm memory usage và tăng tốc query.

---

## 12. Bài tập thực hành đề xuất

1. Implement CQRS với MediatR cho một service CRUD đơn giản bạn từng viết: tạo `CreateXCommand`, `GetXQuery`, `GetXByIdQuery` và handler tương ứng. So sánh độ phức tạp trước/sau — tự đánh giá liệu CQRS có thực sự cần thiết cho service này không.
2. Viết một query handler dùng Dapper thay vì EF Core cho thao tác lấy danh sách có join phức tạp (VD: appointment kèm tên bác sĩ), so sánh thời gian thực thi với cách dùng EF Core thông thường.
3. Thiết kế một denormalized read model cho 1 trong các service trong hệ thống phòng khám — xác định rõ: cần đồng bộ dữ liệu bằng cơ chế nào (event, job nền), tần suất đồng bộ, và độ trễ chấp nhận được.
4. Refactor code tổ chức theo "Commands/Queries" folder sang vertical slice architecture (group theo feature) cho ít nhất 1 feature, nhận xét về sự khác biệt trong việc điều hướng codebase.

---

*Ghi chú: Chương tiếp theo sẽ tiếp tục khám phá các pattern thiết kế khác giúp xây dựng hệ thống microservices vững chắc hơn.*
