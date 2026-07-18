# Chapter 2 — Domain-Driven Design: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi — thuộc lòng

| Khái niệm | Định nghĩa ngắn gọn | Ví dụ (hệ thống phòng khám) |
|---|---|---|
| **Ubiquitous Language** | Từ vựng chung, thống nhất giữa dev và chuyên gia nghiệp vụ, dùng cả trong họp lẫn trong code | Gọi là `Patient` trong code, không phải `Customer` |
| **Bounded Context** | Ranh giới logic nơi một model, ngôn ngữ, quy tắc áp dụng | Document Management context khác Appointment Management context |
| **Entity** | Định danh bởi identity, có vòng đời, thuộc tính có thể đổi nhưng identity không đổi | `Patient` (định danh bởi Patient ID) |
| **Value Object** | Định danh bởi giá trị thuộc tính, bất biến (immutable), hoán đổi được nếu giá trị giống nhau | `TimeSlot(Start, End)` |
| **Aggregate** | Cụm entity + value object được xử lý như 1 khối để đảm bảo tính nhất quán | `Appointment` aggregate gồm Patient, Doctor, TimeSlot, Location |
| **Aggregate Root** | Entity duy nhất là điểm truy cập từ bên ngoài vào aggregate | `Appointment` là root, mọi thay đổi phải qua nó |
| **Invariant** | Quy tắc định nghĩa trạng thái hợp lệ, phải luôn được enforce | "Không được đặt lịch trong quá khứ" |

---

## 2. Phân biệt Entity vs Value Object (câu hỏi phỏng vấn kinh điển)

| Tiêu chí | Entity | Value Object |
|---|---|---|
| Định danh bởi | Identity (ID) | Giá trị thuộc tính |
| Mutable? | Có (thuộc tính đổi được) | Không (immutable — đổi phải tạo mới) |
| So sánh bằng nhau | So theo ID | So theo toàn bộ giá trị |
| Kiểu C# khuyên dùng | `class` | `record` (C# 10+) |
| Ví dụ | Patient, Doctor, Appointment | TimeSlot, Address, Money |

**Analogy để nhớ**: Entity = con người (CCCD không đổi dù thuộc tính đổi). Value Object = tờ tiền 100k (hai tờ khác nhau nhưng giá trị bằng nhau thì coi là như nhau).

---

## 3. Rich Domain Model vs Anemic Domain Model

| | Anemic Model | Rich Model |
|---|---|---|
| Chứa gì | Chỉ property | Property + method thể hiện hành vi nghiệp vụ |
| Logic nghiệp vụ ở đâu | Tầng Service bên ngoài | Ngay trong entity/aggregate |
| Rủi ro | Spaghetti code nếu domain phức tạp | Over-engineering nếu domain đơn giản |
| Khi nào dùng | CRUD service đơn giản | Domain phức tạp, nhiều business rule |

**Câu hỏi tự hỏi**: Domain này có đủ phức tạp để cần rich model không, hay chỉ là CRUD?

---

## 4. Aggregate — Quy tắc thiết kế bắt buộc nhớ

1. **Mọi thay đổi phải đi qua aggregate root** — không đụng trực tiếp vào entity con bên trong.
2. **Quan hệ một chiều**: root tham chiếu tới dependent, không bao giờ ngược lại (VD: Appointment → Patient qua ID, Patient không cần biết Appointment nào).
3. **Câu hỏi kiểm tra khi thiết kế**: "Tôi có thể định nghĩa object này mà không cần object kia không?"
4. **Cách xác định aggregate root**: xét xem việc xóa record này có nên kéo theo xóa cascade các object khác trong hierarchy không.
5. **Quan hệ vượt aggregate**: dùng ID reference + domain event + eventual consistency, không nhúng trực tiếp toàn bộ object của aggregate khác.

---

## 5. Checklist: Nên áp dụng DDD hay không?

- [ ] Domain có đủ phức tạp (nhiều business rule, quy trình) để justify công sức đầu tư không?
- [ ] Có chuyên gia nghiệp vụ sẵn sàng đồng hành xuyên suốt dự án không?
- [ ] Độ phức tạp nằm ở nghiệp vụ hay ở kỹ thuật (performance, tích hợp hệ thống)? Nếu là kỹ thuật → DDD không giải quyết đúng vấn đề.
- [ ] Team có đủ thời gian để làm modeling session (event-storming) với domain expert không?

> Nếu phần lớn câu trả lời là "không" → cân nhắc anemic model / CRUD đơn giản, đừng ép DDD.

---

## 6. DDD ↔ Microservices — mối liên kết

- Mỗi **bounded context** là ứng viên tự nhiên để trở thành 1 **microservice** — có model, ngôn ngữ, quy tắc, stack công nghệ riêng.
- Không phải bounded context nào cũng hiển nhiên ngay từ đầu — ví dụ: hệ thống gửi email/alert xác nhận lịch khám có thể tách thành service riêng (message-queue based) dù ban đầu tưởng chỉ là logic phụ trong web app.
- DDD là **hướng dẫn ban đầu** để định nghĩa ranh giới nghiệp vụ rõ ràng, phụ thuộc tối thiểu — tiền đề để thiết kế service loosely coupled.

---

## 7. Ưu điểm / Nhược điểm DDD — bảng ôn nhanh

| Ưu điểm | Nhược điểm |
|---|---|
| Giao tiếp rõ ràng hơn (Ubiquitous Language) | Tốn nguồn lực, thời gian đầu tư lớn |
| Bám sát mục tiêu nghiệp vụ | Overhead phức tạp cho domain đơn giản |
| Linh hoạt, dễ bảo trì (thay đổi khoanh vùng) | Đòi hỏi chuyên môn domain sâu |
| Codebase sạch (tách layer rõ ràng) | Không phù hợp mọi dự án (đặc biệt domain đơn giản/kỹ thuật nặng) |

---

## 8. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Ubiquitous Language | Ngôn ngữ chung thống nhất giữa dev và nghiệp vụ |
| Bounded Context | Ranh giới logic nơi một model/ngôn ngữ/quy tắc áp dụng |
| Entity | Đối tượng định danh bởi identity, có vòng đời |
| Value Object | Đối tượng định danh bởi giá trị, bất biến |
| Aggregate | Cụm entity/value object xử lý như một khối nhất quán |
| Aggregate Root | Entity duy nhất là điểm truy cập vào aggregate từ bên ngoài |
| Invariant | Quy tắc bất biến định nghĩa trạng thái hợp lệ của domain |
| Anemic Domain Model | Model chỉ chứa dữ liệu, không có hành vi |
| Rich Domain Model | Model chứa cả dữ liệu lẫn hành vi nghiệp vụ |
| Context Mapping | Kỹ thuật biểu diễn quan hệ giữa các bounded context |
| Eventual Consistency | Dữ liệu nhất quán sau một khoảng thời gian, không tức thời |
| Domain Event | Sự kiện phát sinh trong domain, dùng để đồng bộ giữa các aggregate/service |
| GUID | Định danh duy nhất toàn cục, ưu tiên trong hệ phân tán thay vì integer tuần tự |

---

## 9. Anti-patterns cần tránh khi thiết kế DDD

- [ ] Có đang để entity con bị sửa trực tiếp từ bên ngoài, không qua aggregate root không? → Vi phạm consistency boundary.
- [ ] Có đang thiết kế quan hệ hai chiều không cần thiết giữa các aggregate (navigation property cả hai phía) không? → Tăng coupling không cần thiết.
- [ ] Có đang áp DDD đầy đủ (entity, value object, aggregate, repository, factory) cho một service CRUD đơn giản không? → Over-engineering.
- [ ] Value object có đang bị mutate trực tiếp (thay vì tạo instance mới) không? → Vi phạm nguyên tắc immutable.
- [ ] Có dùng integer ID tuần tự xuyên suốt nhiều service thay vì GUID không? → Rủi ro trùng ID trong hệ phân tán.

---

## 10. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Phân biệt Entity và Value Object, cho ví dụ.**
→ Entity định danh bởi identity, mutable, vòng đời riêng (VD: Patient). Value Object định danh bởi giá trị, immutable, hoán đổi được nếu giá trị giống nhau (VD: TimeSlot, Money). Trong C#, value object nên dùng `record` để có value-based equality miễn phí.

**Q2: Aggregate Root là gì và vì sao cần nó?**
→ Là entity duy nhất được phép truy cập từ bên ngoài vào một aggregate. Mọi thay đổi phải đi qua root để đảm bảo business rule/invariant luôn được enforce nhất quán, tránh việc entity con bị sửa trực tiếp gây vỡ tính toàn vẹn dữ liệu.

**Q3: Vì sao DDD khuyến khích quan hệ một chiều giữa các aggregate?**
→ Để giảm coupling. Nếu Appointment và Patient tham chiếu hai chiều, ta tạo phụ thuộc cứng không cần thiết. Chỉ cần Appointment biết PatientId là đủ; Patient không cần biết về Appointment nào đang tồn tại.

**Q4: Bounded Context liên hệ thế nào với việc thiết kế microservices?**
→ Mỗi bounded context là ứng viên tự nhiên để trở thành 1 microservice, vì nó đã có sẵn ranh giới ngữ nghĩa, model, ngôn ngữ và quy tắc nghiệp vụ riêng — tiền đề tốt cho một service loosely coupled, tự chứa.

**Q5: Khi nào KHÔNG nên dùng DDD?**
→ Khi domain đơn giản (CRUD thuần túy), khi độ phức tạp chính nằm ở kỹ thuật (performance, tích hợp) chứ không phải nghiệp vụ, hoặc khi không có đủ nguồn lực/chuyên gia nghiệp vụ đồng hành. Áp DDD trong các trường hợp này gây over-engineering.

**Q6: Vì sao nên dùng GUID thay vì integer tuần tự làm identity trong microservices?**
→ GUID cho phép mỗi service tự sinh định danh cục bộ mà không cần điều phối tập trung (decentralized generation), tránh trùng lặp khi merge dữ liệu từ nhiều service, và khó đoán hơn nên an toàn hơn trước enumeration attack. Integer tuần tự chỉ an toàn trong phạm vi 1 service/1 database.

**Q7: Rich Domain Model và Anemic Domain Model khác nhau thế nào, khi nào dùng cái nào?**
→ Rich model tự chứa hành vi nghiệp vụ trong entity (method tự validate, tự xử lý logic). Anemic model chỉ có property, logic đẩy ra ngoài Service layer. Rich model phù hợp domain phức tạp nhiều business rule; anemic model phù hợp CRUD service đơn giản, tránh over-engineering.

**Q8: Eventual Consistency là gì, và vì sao cần nó khi làm việc với nhiều aggregate/service?**
→ Là chiến lược chấp nhận dữ liệu sẽ nhất quán sau một khoảng thời gian, không cần đồng bộ tức thời — thường thực hiện qua domain event bất đồng bộ. Cần thiết vì enforce strong consistency thời gian thực trên toàn hệ thống phân tán dễ gây tight coupling và nghẽn hiệu năng.

---

## 11. Bài tập thực hành đề xuất

1. Xác định 3 bounded context trong một hệ thống bạn đang làm (hoặc hệ thống phòng khám), vẽ context map thể hiện khái niệm nào được chia sẻ giữa các context.
2. Viết class `Address` như một value object dùng `record` trong C#, đảm bảo validate dữ liệu trong constructor.
3. Thiết kế aggregate `Prescription` (đơn thuốc) cho hệ thống phòng khám: xác định root, entity con, value object, và ít nhất 2 invariant cần enforce.
4. Refactor một class anemic model bạn từng viết thành rich model — chuyển ít nhất 2 logic nghiệp vụ từ Service layer vào trong entity.

---

*Ghi chú: Chương tiếp theo (Chapter 3) sẽ khám phá giao tiếp đồng bộ (synchronous communication) giữa các microservice.*
