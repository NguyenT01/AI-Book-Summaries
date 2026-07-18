# Chapter 8 — Database Design Strategies: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Database per Service** | Mỗi microservice sở hữu database riêng, có thể trên hạ tầng hosting riêng |
| **Polyglot Persistence** | Mỗi service chọn công nghệ database phù hợp nhất (SQL, NoSQL, Graph...) thay vì đồng nhất |
| **Shared Database** | Nhiều service dùng chung 1 database nhưng phân tách logic qua schema/table |
| **Tables-per-service** | Mỗi service chỉ dùng/model đúng các bảng thuộc về mình trong 1 database chung |
| **Schema-per-service** | Dùng schema của DB quan hệ để phân loại bảng theo từng service |
| **Referential Integrity** | Ràng buộc đảm bảo giá trị foreign key luôn hợp lệ, tham chiếu đúng primary key |
| **Normalization** | Kỹ thuật tách dữ liệu qua nhiều bảng để giảm dư thừa, dùng foreign key liên kết |
| **Sharding** | Phân vùng dữ liệu qua nhiều node (shard), mỗi node giữ 1 tập con dataset |
| **Materialization** | Quy trình EF Core chuyển kết quả SQL thành object C# |
| **Code-First** | Model database bằng class C#, quản lý thay đổi qua migration |
| **Schema-First (Database-First)** | Tạo database trước, scaffold vào ứng dụng |
| **Outbox Pattern** | Lưu event vào bảng outbox cùng transaction với business data, đảm bảo atomicity |
| **Eventual Consistency** | Dữ liệu đồng bộ theo thời gian thay vì tức thời |

---

## 2. ACID — bốn nguyên tắc, bảng ví dụ nhanh

| Nguyên tắc | Ý nghĩa | Ví dụ hệ thống phòng khám |
|---|---|---|
| **Atomicity** | Transaction là tất cả-hoặc-không-gì | Đặt lịch + thanh toán: nếu 1 trong 2 fail, rollback toàn bộ |
| **Consistency** | Transaction luôn đưa DB từ trạng thái hợp lệ này sang hợp lệ khác | Không cho thêm bệnh nhân thiếu họ tên |
| **Isolation** | Transaction đồng thời không can thiệp lẫn nhau | 2 người không thể cùng đặt trùng 1 khung giờ với cùng bác sĩ |
| **Durability** | Transaction đã commit thì tồn tại vĩnh viễn dù crash | Lịch hẹn vẫn còn sau khi hệ thống restart |

---

## 3. So sánh Relational vs Non-Relational Database

| Tiêu chí | Relational (SQL) | Non-Relational (NoSQL) |
|---|---|---|
| Nguyên tắc nền tảng | ACID | Linh hoạt, ưu tiên tốc độ/scale |
| Cấu trúc dữ liệu | Bảng, có schema cố định | Document/key-value/graph, schema linh hoạt |
| Quan hệ dữ liệu | Chuẩn hóa (normalize), dùng foreign key | Thường denormalize, gộp dữ liệu vào 1 document |
| Khi nào dùng | Báo cáo/query phức tạp, transaction cao, cần ACID, service ổn định | Service tiến hóa nhanh, dữ liệu không chuẩn hóa, cần scale nhanh |
| Ví dụ công nghệ | SQL Server, PostgreSQL, MySQL, Oracle, SQLite | MongoDB, Cosmos DB, DynamoDB, Redis, Neo4j |
| Nhược điểm chính | Query phức tạp qua nhiều bảng chậm khi dữ liệu lớn | Dư thừa dữ liệu, khó bảo trì khi cần sửa hàng loạt |

---

## 4. Ba loại NoSQL Database

| Loại | Đặc điểm | Ví dụ |
|---|---|---|
| **Document Database** | Lưu JSON object, hỗ trợ lồng nhau | MongoDB, Cosmos DB, CouchDB |
| **Key-Value Database** | Cặp key-value đơn giản, dùng cho lookup nhanh | Redis |
| **Graph Database** | Node + Edge, biểu diễn quan hệ phức tạp | Neo4j |

---

## 5. Cardinality — ba loại quan hệ

| Loại | Mô tả | Ví dụ |
|---|---|---|
| **One-to-one** | PK tham chiếu đúng 1 lần ở bảng khác | 1 bệnh nhân — 1 địa chỉ |
| **One-to-many** | PK tham chiếu nhiều lần ở bảng khác | 1 khách hàng — nhiều đơn hàng |
| **Many-to-many** | PK cả 2 bảng tham chiếu qua lại nhiều lần, cần linker table | Khách hàng ↔ Phòng khám (qua bảng Appointments) |

---

## 6. Sáu yếu tố chọn công nghệ (database/ORM/stack nói chung)

| Yếu tố | Câu hỏi cần trả lời |
|---|---|
| Maintainability | Dễ bảo trì không? Update/patch thường xuyên không? |
| Extensibility | Có hỗ trợ tốt khi yêu cầu thay đổi không? |
| Supporting technologies | Có phù hợp với stack hiện tại không? |
| Comfort level | Team có đủ quen thuộc không? |
| Maturity | Đã được kiểm chứng, có cộng đồng mạnh không? |
| Appropriateness | Có phù hợp với throughput/latency yêu cầu thực tế không? |

---

## 7. So sánh EF Core vs Dapper

| Tiêu chí | EF Core | Dapper |
|---|---|---|
| Loại | ORM đầy đủ | Micro-ORM |
| Tính năng | Change tracking, lazy loading, migration tự động | Raw SQL, map trực tiếp sang object |
| Hiệu năng | Tốt, có overhead abstraction | Rất nhanh, gần bằng ADO.NET thuần |
| Hỗ trợ NoSQL | Có (VD Azure Cosmos DB) | Không native |
| Khi nào dùng | Cần đầy đủ tính năng, migration, đa dạng DB | Cần hiệu năng cao, kiểm soát SQL chặt |

---

## 8. So sánh Code-First vs Schema-First

| Tiêu chí | Code-First | Schema-First (Database-First) |
|---|---|---|
| Điểm bắt đầu | Viết class C# trước | Tạo database trước bằng công cụ DB |
| Quản lý thay đổi | Qua migration (`Up`/`Down`) | Chạy lại lệnh `scaffold` mỗi khi DB đổi |
| Version tracking | Có (bảng `__EFMigrationsHistory`) | Không có cơ chế tracking sẵn |
| Phù hợp khi | Dự án mới, cần kiểm soát code | Làm việc với database legacy có sẵn |
| Lệnh chính | `dotnet ef migrations add`, `dotnet ef database update` | `dotnet ef dbcontext scaffold ... -o Models` |

---

## 9. Checklist Migration trong EF Core

- [ ] Đã tạo migration bằng `dotnet ef migrations add <Name>` chưa?
- [ ] Đã kiểm tra migration bằng `dotnet ef migrations list` trước khi xóa/rollback chưa?
- [ ] Đã backup database trước khi rollback (`dotnet ef database update <MigrationName>`) chưa? (Rollback có thể mất dữ liệu)
- [ ] Nếu cần SQL script để deploy production, đã dùng `--idempotent` chưa (đảm bảo áp dụng an toàn bất kể trạng thái DB hiện tại)?
- [ ] Nếu làm việc nhóm, đã có quy trình tránh migration bị "lệch pha" khi nhiều người cùng tạo migration đồng thời chưa?

---

## 10. Hai kỹ thuật Sharding

| Kỹ thuật | Cách hoạt động | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **Range-based** | Phân vùng theo dải liên tục (ID, timestamp) | Dễ implement, tốt cho range query | Có thể tạo hot spot nếu dữ liệu không đều |
| **Hash-based** | Áp dụng hash function lên sharding key | Phân phối đồng đều hơn | Khó range query, phức tạp khi scale |

---

## 11. Năm mức Consistency Level của Azure Cosmos DB

| Mức | Đặc điểm |
|---|---|
| Strong | Luôn đọc được write mới nhất |
| Bounded staleness | Đọc chậm hơn write tối đa X thời gian/version |
| Session | Nhất quán trong phạm vi 1 session — "đọc chính write của mình" |
| Consistent prefix | Không thấy write sai thứ tự (có thể chưa đầy đủ) |
| Eventual | Yếu nhất — có thể đọc dữ liệu cũ/sai thứ tự, cuối cùng hội tụ |

---

## 12. Checklist: Database per Service vs Shared Database

- [ ] Team có đủ nhân sự/ngân sách để duy trì nhiều công nghệ database khác nhau không? (Nếu không → cân nhắc Shared Database)
- [ ] Có cần polyglot persistence thực sự không, hay chỉ là "muốn dùng cho ngầu"?
- [ ] Nếu dùng Shared Database, đã thiết lập connection pool riêng cho từng service chưa (Isolation)?
- [ ] Đã thiết lập resource quota (CPU/memory/I/O) theo user-level cho từng service chưa?
- [ ] Nếu dùng Shared Database, đã chọn tables-per-service hay schema-per-service, và có naming convention rõ ràng chưa?
- [ ] Đã đánh giá rủi ro: 1 service traffic cao có thể làm nghẽn/downtime các service khác dùng chung DB không?

---

## 13. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Decentralized Data Ownership | Sở hữu dữ liệu phi tập trung — mỗi service tự quản lý data của mình |
| Polyglot Persistence | Đa dạng công nghệ lưu trữ, mỗi service chọn loại phù hợp nhất |
| Referential Integrity | Toàn vẹn tham chiếu — ràng buộc foreign key luôn hợp lệ |
| Normalization | Chuẩn hóa — tách dữ liệu qua nhiều bảng để giảm dư thừa |
| Denormalization | Phi chuẩn hóa — gộp dữ liệu để tăng tốc query, chấp nhận dư thừa |
| Cardinality | Bản số — tính chất số lượng của 1 quan hệ (1-1, 1-n, n-n) |
| Linker Table | Bảng trung gian giải quyết quan hệ many-to-many |
| Sharding | Phân vùng dữ liệu ngang qua nhiều node/server |
| Partition Key / Shard Key | Trường dữ liệu quyết định cách phân phối dữ liệu vào các shard |
| Materialization | Quy trình chuyển kết quả query thành object ngôn ngữ lập trình |
| Idempotent Script | Script có thể chạy nhiều lần mà không gây lỗi/trùng lặp |
| Scaffolding | Quy trình sinh code model từ database đã có sẵn |
| Outbox Pattern | Pattern đảm bảo atomicity giữa ghi DB và publish event |
| Hot Spot | Điểm dữ liệu/shard bị quá tải do phân phối không đều |
| Consistent Hashing | Kỹ thuật hash đảm bảo phân phối dữ liệu cân bằng qua các node |

---

## 14. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Vì sao microservices khuyến khích "database per service" thay vì dùng chung 1 database?**
→ Để đảm bảo tính độc lập (autonomy) và loose coupling giữa các service — một service có thể thay đổi schema, công nghệ, hoặc scale mà không ảnh hưởng service khác. Dùng chung database sẽ tạo coupling chặt ở tầng dữ liệu, một thay đổi schema có thể vô tình phá vỡ service khác, và một service traffic cao có thể làm nghẽn toàn bộ hệ thống.

**Q2: Polyglot Persistence là gì, và khi nào nên áp dụng?**
→ Là chiến lược cho phép mỗi microservice chọn công nghệ database phù hợp nhất với nhu cầu riêng (SQL cho dữ liệu có cấu trúc/transaction cao, NoSQL cho dữ liệu linh hoạt/scale nhanh, object storage cho file lớn). Nên áp dụng khi các service có nhu cầu dữ liệu thực sự khác biệt (VD: Patient Records cần ACID, Documents Service cần lưu file lớn) — không nên áp dụng chỉ vì muốn dùng công nghệ mới.

**Q3: Trade-off chính giữa Relational và Non-Relational Database là gì?**
→ Relational mạnh về tính toàn vẹn dữ liệu (ACID), phù hợp cho transaction phức tạp và báo cáo, nhưng hiệu năng giảm khi cần join nhiều bảng với dữ liệu lớn. Non-Relational linh hoạt, scale ngang tốt, tốc độ đọc/ghi nhanh vì dữ liệu gộp trong 1 document, nhưng đánh đổi bằng dư thừa dữ liệu và khó bảo trì khi cần sửa hàng loạt bản ghi liên quan.

**Q4: Sharding khác gì so với Replication?**
→ (Câu hỏi mở rộng ngoài nội dung chương, nhưng thường được hỏi kèm) Sharding chia dữ liệu thành các tập con khác nhau trên nhiều node (mỗi node giữ 1 phần dữ liệu khác nhau) — mục tiêu là scale khối lượng dữ liệu và traffic. Replication sao chép toàn bộ dữ liệu sang nhiều node giống hệt nhau — mục tiêu là tăng availability/fault tolerance. Hai kỹ thuật thường được kết hợp trong hệ thống lớn.

**Q5: Range-based sharding và Hash-based sharding khác nhau thế nào, khi nào dùng loại nào?**
→ Range-based phân vùng theo dải giá trị liên tục (VD theo ID hoặc timestamp) — dễ implement, tốt cho range query nhưng dễ tạo hot spot nếu dữ liệu phân bố không đều theo thời gian. Hash-based áp dụng hàm hash lên sharding key để phân phối đồng đều hơn, nhưng khó thực hiện range query và phức tạp hơn khi cần thêm/bớt shard.

**Q6: Outbox Pattern giải quyết vấn đề gì?**
→ Giải quyết vấn đề đảm bảo tính atomic giữa việc cập nhật database và publish event thông báo cho service khác. Nếu publish event thất bại sau khi transaction DB đã commit, hệ thống rơi vào trạng thái không nhất quán. Outbox Pattern lưu event vào 1 bảng outbox trong CÙNG transaction với business data, sau đó một tiến trình riêng đọc và publish các event pending — đảm bảo event không bao giờ bị mất dù publish thất bại tạm thời.

**Q7: Code-First và Schema-First khác nhau ở điểm nào, và ưu tiên chọn khi nào?**
→ Code-First: viết class C# trước, EF Core sinh database qua migration — có version tracking rõ ràng (bảng `__EFMigrationsHistory`), phù hợp cho dự án mới. Schema-First (Database-First): tạo database trước bằng công cụ quản trị, sau đó scaffold vào ứng dụng — không có cơ chế tracking version sẵn có, cần chạy lại lệnh scaffold mỗi khi DB đổi, phù hợp khi làm việc với database legacy đã tồn tại.

**Q8: Azure Cosmos DB cung cấp 5 mức consistency level — tại sao cần nhiều mức thay vì chỉ Strong hoặc Eventual?**
→ Vì các ứng dụng khác nhau có nhu cầu trade-off khác nhau giữa consistency, availability, và performance. Strong consistency đảm bảo chính xác nhất nhưng chậm và ít khả dụng hơn trong điều kiện mạng gián đoạn; Eventual nhanh và khả dụng cao nhất nhưng có thể đọc dữ liệu cũ. Các mức trung gian (Bounded staleness, Session, Consistent prefix) cho phép chọn điểm cân bằng phù hợp với từng use case cụ thể thay vì chỉ có 2 lựa chọn cực đoan.

---

## 15. Bài tập thực hành đề xuất

1. Thiết kế schema database cho 3 service trong hệ thống phòng khám (Patient Records, Appointment Scheduling, Documents) — với mỗi service, chọn công nghệ database phù hợp (relational/NoSQL/object storage) và giải thích lý do dựa trên 6 yếu tố đã học.
2. Thực hành tạo và rollback 1 migration bằng EF Core: `dotnet ef migrations add`, kiểm tra bảng `__EFMigrationsHistory`, sau đó rollback bằng `dotnet ef database update <MigrationName>`.
3. Viết ví dụ bảng many-to-many với linker table cho quan hệ Doctor ↔ Specialty (một bác sĩ có nhiều chuyên khoa, một chuyên khoa có nhiều bác sĩ).
4. Thiết kế outbox pattern đơn giản: tạo bảng `OutboxEvents` trong cùng database với bảng nghiệp vụ chính, viết pseudocode cho tiến trình đọc và publish event pending.
5. So sánh thời gian query giữa mô hình relational (join 3 bảng) và mô hình document (1 document gộp sẵn dữ liệu) cho cùng 1 truy vấn lấy chi tiết đầy đủ 1 lịch hẹn.

---

*Ghi chú: Chương tiếp theo (Chapter 9) sẽ giải quyết vấn đề transaction xuyên suốt nhiều microservice bằng Saga Pattern.*
