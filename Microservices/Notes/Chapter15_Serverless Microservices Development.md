# Chapter 15 — Serverless Microservices Development: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| Serverless computing | Mô hình thực thi cloud-native, nhà cung cấp tự động cấp phát tài nguyên; developer chỉ viết code, không quản lý server |
| FaaS (Functions as a Service) | Mô hình serverless phổ biến nhất — deploy từng function độc lập, kích hoạt bởi sự kiện cụ thể |
| BaaS (Backend as a Service) | Dịch vụ backend dựng sẵn (auth, storage, database, messaging real-time) bổ trợ cho FaaS |
| Trigger | Cơ chế quyết định khi nào một function được kích hoạt (HTTP, queue, timer, blob...) |
| Binding | Liên kết khai báo (declarative) giữa function và tài nguyên bên ngoài (queue, table, blob) |
| Cold start | Độ trễ khi function được kích hoạt sau thời gian không hoạt động, do môi trường thực thi đã scale về 0 |
| Statelessness | Đặc tính function serverless không giữ trạng thái giữa các lần gọi, trừ khi dùng durable function |
| Azure Functions | Nền tảng FaaS của Microsoft Azure, hỗ trợ .NET, Node.js, Python, Java, PowerShell |
| Azure Durable Functions | Phần mở rộng của Azure Functions cho phép orchestration có trạng thái (stateful workflow) |
| Isolated worker model | Mô hình chạy .NET Function trong tiến trình riêng biệt, hỗ trợ đầy đủ DI kiểu ASP.NET Core và .NET 8 |
| In-process model | Mô hình function chạy trong chính tiến trình host của Azure Functions (cũ hơn isolated) |
| Orchestrator function | Function điều phối chuỗi activity function theo logic workflow (chaining, fan-out/fan-in...) |
| Activity function | Function thực thi một bước cụ thể trong workflow, được orchestrator gọi |
| Fan-out/fan-in | Pattern chạy nhiều tác vụ song song rồi gộp/chờ tất cả hoàn tất |
| Function chaining | Pattern chạy các tác vụ tuần tự, output bước này là input bước sau |
| WaitForExternalEvent | Method cho phép orchestration tạm dừng chờ sự kiện/tương tác con người bên ngoài |
| Monitor pattern | Pattern lặp kiểm tra điều kiện tới khi thỏa mãn hoặc timeout |
| Asynchronous HTTP API pattern | Client POST khởi tạo workflow dài, nhận 202 + status URL để poll kết quả |
| Azure Storage Table | Dịch vụ lưu trữ NoSQL key-value/dòng, dùng PartitionKey + RowKey định danh duy nhất |
| PartitionKey | Khóa xác định partition (đơn vị co giãn) trong Azure Table Storage |
| Azurite | Emulator giả lập Azure Storage (queue, table, blob) chạy local |
| Azure Functions Core Tools | Runtime CLI để tạo, chạy, debug Azure Functions tại local |
| Managed Identity | Cơ chế cho phép Azure resource (như Function) tự xác thực với resource khác mà không cần quản lý credential |
| Azure Key Vault | Kho lưu trữ an toàn cho secret, key, chứng chỉ |
| Application Insights | Công cụ telemetry chi tiết cho hiệu năng function, dependency call, chẩn đoán lỗi |
| Vendor lock-in | Rủi ro bị ràng buộc chặt vào API/mô hình sự kiện đặc thù của một nhà cung cấp cloud |

---

## 2. So sánh 3 mức hosting: Server vật lý — Container — Serverless

| Tiêu chí | Server vật lý / VM | Container (Ch14) | Serverless (Ch15) |
|---|---|---|---|
| Quản lý hạ tầng | Toàn bộ do team tự làm | Cần orchestration (K8s/Compose) | Không cần — nhà cung cấp lo hết |
| Chi phí | Cố định, kể cả lúc idle | Cố định theo giờ chạy container | Trả theo đúng thời gian function chạy |
| Tốc độ scale | Chậm, thủ công | Nhanh, cần policy auto-scale | Tự động, gần như tức thời |
| Cold start | Không có (luôn "nóng") | Thấp nếu container đã chạy sẵn | Có — đáng kể nếu function bị scale về 0 |
| Phù hợp cho | Workload ổn định, tải cao liên tục | Service chạy dài hạn, cần kiểm soát runtime | Workload ngắn hạn, hướng sự kiện, tải thất thường |
| Độ phức tạp vận hành | Cao | Trung bình–cao (cần đội DevOps) | Thấp (nhưng đổi bằng vendor lock-in) |

## 3. So sánh dịch vụ serverless: AWS vs Azure

| Nhóm chức năng | AWS | Azure |
|---|---|---|
| FaaS | AWS Lambda | Azure Functions |
| API Gateway | Amazon API Gateway | Azure API Management (APIM) |
| Event bus / routing | Amazon EventBridge | Azure Event Grid |
| Pub/sub notification | Amazon SNS | Azure Service Bus (topics) |
| Message queue | Amazon SQS | Azure Queue Storage / Service Bus (queues) |
| Workflow orchestration | AWS Step Functions | Azure Durable Functions |
| Observability | Amazon CloudWatch | Azure Monitor + Application Insights |
| NoSQL database | Amazon DynamoDB | Azure Cosmos DB |
| Object storage | Amazon S3 | Azure Blob Storage |
| Relational serverless DB | Amazon Aurora Serverless | Azure SQL Database (Serverless tier) |
| Secret management | AWS Secrets Manager (không nêu chi tiết trong chương) | Azure Key Vault |

## 4. Bốn pattern chính của Durable Functions

| Pattern | Cơ chế | Ví dụ HealthCare |
|---|---|---|
| Fan-out/fan-in | Chạy song song nhiều activity, chờ tất cả hoàn tất | Sau khi có kết quả xét nghiệm: đồng thời báo bệnh nhân + bác sĩ + cập nhật hồ sơ |
| Function chaining | Chạy tuần tự, output bước trước là input bước sau | Đăng ký bệnh nhân → xác minh bảo hiểm → gán bác sĩ → gửi email kích hoạt |
| Chờ sự kiện ngoài (WaitForExternalEvent) | Tạm dừng workflow không tốn compute, chờ tín hiệu | Chờ bác sĩ xác nhận đơn thuốc, chờ bệnh nhân đồng ý điều trị |
| Monitor | Lặp kiểm tra điều kiện tới khi thỏa mãn/timeout | Kiểm tra mỗi giờ xem bệnh nhân đã xác nhận lịch hẹn chưa, hủy sau 24h nếu không |
| Async HTTP API | POST khởi tạo, trả 202 + status URL, client poll | Xuất toàn bộ hồ sơ y tế bệnh nhân thành file tải về |

## 5. Vòng đời request trong ví dụ Appointment Booking

| Bước | Thành phần | Vai trò |
|---|---|---|
| 1 | `StartBookingHttp` (HTTP trigger) | Nhận POST, validate sơ bộ, đẩy message vào queue `appointments` |
| 2 | `StartBookingQueue` (Queue trigger) | Nhận message từ queue, khởi tạo `BookingOrchestrator` |
| 3 | `BookingOrchestrator` (Orchestrator trigger) | Điều phối: chain 2 bước tạo dữ liệu → fan-out 2 email |
| 4 | `AddPatientActivity` (Activity trigger) | Ghi bệnh nhân vào Azure Table `Patients` |
| 5 | `AddAppointmentActivity` (Activity trigger) | Ghi lịch hẹn vào Azure Table `Appointments`, PartitionKey = ngày hẹn |
| 6 | `SendPatientEmailActivity` + `SendAdminEmailActivity` (Activity trigger, song song) | Gửi email thông báo cho bệnh nhân và admin |

---

## Checklist kỹ thuật khi triển khai

- [ ] Mỗi function chỉ đảm nhiệm đúng một trách nhiệm (single responsibility), đặt tên theo "làm gì" chứ không phải "làm thế nào"
- [ ] Đã đánh giá: workload này có bản chất ngắn hạn/thất thường/hướng sự kiện — phù hợp serverless — hay cần chạy ổn định liên tục, phù hợp container hơn?
- [ ] Dùng isolated worker model (`dotnet-isolated`) cho project Azure Functions .NET mới
- [ ] DI graph tối giản — chỉ đăng ký service thực sự cần, tránh khuếch đại cold start
- [ ] Đã chọn đúng lifetime cho service: Singleton cho tài nguyên dùng chung (HttpClient), Scoped cho ngữ cảnh theo request
- [ ] Với workflow nhiều bước/cần retry/cần chờ sự kiện: đã dùng Durable Functions thay vì tự chain HTTP call thủ công
- [ ] PartitionKey của Azure Table được thiết kế có chủ đích (theo domain truy vấn thường gặp), không để mặc định ngẫu nhiên
- [ ] Secret/API key/connection string không hardcode — lưu ở Azure Key Vault, truy cập qua Managed Identity
- [ ] Đã bật Application Insights, kiểm tra correlation ID xuyên suốt orchestration để trace được toàn bộ workflow
- [ ] Đã cân nhắc cold start cho các endpoint nhạy cảm về độ trễ (ví dụ tra cứu bệnh nhân) — cân nhắc container hóa hoặc reserved plan nếu cần
- [ ] Đã thiết lập bộ công cụ local (VS Code + Azure Tools, Azure Functions Core Tools, Azurite) trước khi deploy thật lên Azure

---

## Anti-patterns / Lưu ý cần tránh

- **Gộp nhiều tác vụ không liên quan vào một function "cho tiện"** → phá vỡ single responsibility, khó test, khó trace lỗi.
- **Chain quá nhiều function HTTP nhỏ lẻ thay vì dùng orchestrator** → tăng overhead hiệu năng, tracing trở nên rối rắm.
- **Coi function serverless có thể giữ trạng thái giữa các lần gọi như một service thông thường** → sai bản chất stateless; phải dùng external store hoặc Durable Functions.
- **Hardcode secret/connection string trong code hoặc biến môi trường thường** → rủi ro bảo mật nghiêm trọng; luôn dùng Key Vault + Managed Identity.
- **Bỏ qua ảnh hưởng của cold start với endpoint quan trọng, độ trễ cao là chấp nhận được** → làm xói mòn trải nghiệm người dùng ở các luồng nghiệp vụ then chốt (ví dụ tra cứu hồ sơ y tế khẩn cấp).
- **Chuyển toàn bộ hệ thống sang serverless chỉ vì xu hướng, không đánh giá đặc tính workload** → một số service chạy ổn định, tải cao liên tục sẽ phù hợp và rẻ hơn với container hóa (Chapter 14).
- **Dựa hoàn toàn vào `depends_on` kiểu tuần tự thủ công thay vì Durable Functions cho workflow có retry/compensation** → mất khả năng phục hồi tự động khi một bước thất bại giữa chừng.
- **Không đo lường vendor lock-in trước khi thiết kế sâu vào API đặc thù của một cloud provider** → khó di chuyển workload nếu sau này cần đổi nhà cung cấp.

---

## Từ vựng chuyên ngành (Anh - Việt)

| Thuật ngữ | Nghĩa |
|---|---|
| Serverless computing | Điện toán serverless (không quản lý server trực tiếp) |
| Function as a Service (FaaS) | Hàm như một dịch vụ |
| Backend as a Service (BaaS) | Backend như một dịch vụ |
| Trigger | Sự kiện kích hoạt |
| Binding | Liên kết khai báo với tài nguyên ngoài |
| Cold start | Độ trễ khởi động nguội |
| Statelessness | Tính không trạng thái |
| Durable orchestration | Điều phối workflow bền vững, có trạng thái |
| Fan-out/fan-in | Tỏa ra song song / gộp lại |
| Function chaining | Xâu chuỗi function tuần tự |
| Vendor lock-in | Ràng buộc/khóa chặt vào một nhà cung cấp |
| Partition key | Khóa phân vùng dữ liệu |
| Managed identity | Danh tính được quản lý (tự động, không cần credential thủ công) |
| Observability | Khả năng quan sát hệ thống (log, trace, metric) |
| Pay-per-execution | Tính phí theo mỗi lần thực thi |
| Event-driven execution | Thực thi hướng sự kiện |
| Isolated worker model | Mô hình worker chạy tách biệt tiến trình |
| Compensation logic | Logic bù trừ khi một bước trong workflow thất bại |

---

## Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: "Serverless" có nghĩa là không còn server nào cả không?**
→ Không. Server vẫn tồn tại, nhưng trách nhiệm cấp phát, scale, và bảo trì nó được chuyển hẳn sang nhà cung cấp cloud. Developer chỉ cần viết function và định nghĩa trigger, không phải quan tâm tới việc quản lý hạ tầng bên dưới.

**Q2: Vì sao serverless phù hợp tự nhiên với microservices, nhưng không phải lúc nào cũng là lựa chọn tốt nhất?**
→ Serverless khớp với microservices vì cả hai đều hướng tới các đơn vị nhỏ, độc lập, co giãn riêng biệt — và serverless còn loại bỏ luôn gánh nặng orchestration (như Kubernetes). Nhưng nó không phù hợp cho workload cần chạy ổn định liên tục hoặc rất nhạy cảm về độ trễ, vì cold start và bản chất stateless có thể gây vấn đề ở những trường hợp đó — lúc này container hóa (Chapter 14) hoặc kết hợp cả hai là lựa chọn hợp lý hơn.

**Q3: Cold start là gì và làm sao giảm thiểu ảnh hưởng của nó?**
→ Cold start là độ trễ xảy ra khi một function được gọi sau thời gian im lìm, do môi trường thực thi đã bị scale về 0 và cần được cấp phát lại, tải runtime, khởi tạo trước khi xử lý request. Giảm thiểu bằng cách dùng gói function dự trữ sẵn (premium/reserved plan), giảm DI graph để khởi động nhanh hơn, hoặc chuyển các endpoint cực nhạy cảm về độ trễ sang mô hình container hóa.

**Q4: Vì sao serverless function nên được thiết kế stateless, và làm sao xử lý khi cần trạng thái?**
→ Stateless giúp function scale độc lập, dễ tái sử dụng, và tránh phụ thuộc vào một instance cụ thể còn "sống" hay không. Khi cần trạng thái xuyên suốt nhiều bước, dùng external state store (Cosmos DB, Redis) hoặc — tốt hơn — dùng Durable Functions, vốn tự quản lý trạng thái orchestration mà developer không phải tự code cơ chế lưu trữ riêng.

**Q5: Phân biệt Function chaining và Fan-out/fan-in trong Durable Functions?**
→ Function chaining chạy các bước theo thứ tự nghiêm ngặt, output bước trước là input bước sau — dùng khi các bước phụ thuộc lẫn nhau (ví dụ phải tạo bệnh nhân trước khi tạo lịch hẹn). Fan-out/fan-in chạy nhiều tác vụ độc lập song song rồi chờ tất cả hoàn tất — dùng khi các bước không phụ thuộc nhau (ví dụ gửi email cho bệnh nhân và admin cùng lúc), giúp tổng thời gian chạy bằng đúng tác vụ dài nhất thay vì cộng dồn tuần tự.

**Q6: PartitionKey trong Azure Table Storage dùng để làm gì, và nên thiết kế nó ra sao?**
→ PartitionKey xác định partition — đơn vị co giãn của Table Storage — mọi entity cùng PartitionKey nằm trong cùng một phân vùng logic, giúp Azure định vị nhanh khi đọc/ghi mà không phải quét toàn bảng. Nên thiết kế PartitionKey theo tiêu chí truy vấn thường gặp nhất — ví dụ trong hệ thống đặt lịch hẹn, dùng ngày hẹn làm PartitionKey giúp truy vấn "lịch hẹn hôm nay" cực nhanh.

**Q7: Vì sao nên phát triển Azure Functions ở local (dùng Azurite, Core Tools) trước khi deploy lên cloud thật?**
→ Vì phát triển local giúp vòng lặp phản hồi nhanh hơn (không chờ deploy/mạng), tiết kiệm chi phí (tránh phí Azure khi test lặp đi lặp lại), và đảm bảo tính nhất quán đa nền tảng cho cả team. Tuy nhiên cần nhớ local không mô phỏng được đầy đủ các yếu tố production thực tế như cold start hay độ trễ mạng thật.

**Q8: Vendor lock-in trong serverless là gì, và có cách nào giảm thiểu rủi ro này?**
→ Vì mỗi cloud provider có API và mô hình sự kiện đặc thù riêng (trigger, binding, cú pháp cấu hình), code viết cho Azure Functions không thể chuyển thẳng sang AWS Lambda mà không sửa đổi đáng kể — đây chính là vendor lock-in. Giảm thiểu bằng cách tách logic nghiệp vụ thuần túy ra khỏi lớp adapter đặc thù nền tảng (tương tự tinh thần Clean Architecture), để nếu cần đổi provider, chỉ phải viết lại lớp adapter mỏng, không phải viết lại toàn bộ logic.

---

## Bài tập thực hành đề xuất

1. Cài đặt bộ công cụ local (VS Code + Azure Tools, Azure Functions Core Tools, Azurite qua Docker), khởi tạo project `AppointmentBooking` theo đúng các bước trong chương, chạy thử `func start` và gọi thử function HTTP trigger bằng Postman.
2. Viết thêm một activity function mới `VerifyInsuranceEligibilityActivity`, chèn nó vào `BookingOrchestrator` như một bước chaining trước khi tạo appointment — quan sát cách orchestrator xử lý khi bước này throw exception.
3. Thử nghiệm pattern `WaitForExternalEvent`: viết một orchestrator đơn giản tạm dừng chờ sự kiện "PhysicianApproved" trước khi tiếp tục, rồi dùng một HTTP-triggered function riêng để gửi sự kiện đó vào orchestration đang chờ.
4. So sánh thời gian phản hồi của cùng một function khi gọi lần đầu (cold) và gọi lần thứ hai ngay sau đó (warm) — ghi lại số liệu thực tế để hiểu rõ mức độ ảnh hưởng của cold start.
5. Thiết kế lại PartitionKey cho bảng `Appointments` theo một tiêu chí khác (ví dụ theo `ProviderId` thay vì theo ngày), thử nghĩ xem truy vấn nào sẽ nhanh lên và truy vấn nào sẽ chậm đi — từ đó rút ra nguyên tắc chọn PartitionKey theo pattern truy vấn thực tế của hệ thống.

---

*Ghi chú: Chương tiếp theo sẽ chuyển sang khảo sát Observability and Monitoring with Modern Tools — đào sâu các khái niệm đã xuất hiện rải rác trong chương này (Application Insights, correlation ID, trace, KQL) thành một trụ cột kiến trúc độc lập, tiếp tục mạch Part 4 — Cloud Development Strategies.*
