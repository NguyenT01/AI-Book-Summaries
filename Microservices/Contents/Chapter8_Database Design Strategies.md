# Bài học: Database Design Strategies — Khi mỗi service muốn "ở riêng" nhưng vẫn cần "nói chuyện" với nhau

> Chương 8 — Database Design Strategies for Microservices
> Ví dụ xuyên suốt: Hệ thống quản lý phòng khám — Patient Records Service, Appointment Scheduling Service, Documents Service, mỗi service tự chọn công nghệ database phù hợp nhất

---

## Phần 1: Vấn đề thực tế — quản lý dữ liệu phân tán không giống quản lý một database duy nhất

Ta đã đi qua Event Sourcing (Chapter 7) — một cách giúp các service hòa giải (reconcile) thay đổi dữ liệu giữa nhau. Giờ ta lùi lại một bước, nhìn vào bức tranh rộng hơn: **quản lý dữ liệu trong toàn bộ kiến trúc microservices**.

Đây là một thách thức mang tính sống còn, quyết định trực tiếp tới khả năng mở rộng (scalability), khả năng phục hồi (resilience), và khả năng bảo trì (maintainability) của toàn hệ thống. Khác với ứng dụng monolith — nơi dữ liệu thường được quản lý tập trung trong một database — microservices khuyến khích mô hình **sở hữu dữ liệu phi tập trung (decentralized data ownership)**. Mỗi service tự chịu trách nhiệm cho việc lưu trữ, truy xuất, và duy trì tính nhất quán dữ liệu của riêng nó — điều này kéo theo hàng loạt phức tạp về toàn vẹn dữ liệu, transaction, và giao tiếp liên service.

### Analogy: Mỗi phòng ban có tủ hồ sơ riêng thay vì một kho lưu trữ chung

Hãy hình dung một bệnh viện lớn. Cách cũ: toàn bộ hồ sơ — từ lịch hẹn, bệnh án, tới hóa đơn — đều nằm trong **một kho lưu trữ trung tâm duy nhất**, ai cũng phải xếp hàng chờ tới lượt truy cập. Cách mới: mỗi phòng ban (Khoa Khám bệnh, Khoa Thu ngân, Phòng Chụp X-quang) có **tủ hồ sơ riêng**, tự quản lý theo cách phù hợp nhất với công việc của mình — Khoa Khám bệnh cần tủ hồ sơ có ngăn phân loại chi tiết (dữ liệu có cấu trúc chặt), Phòng X-quang cần kho lưu trữ phim lớn (dữ liệu nhị phân dung lượng cao). Nhưng giờ đây, khi cần thông tin chéo giữa các phòng ban, không thể tự ý mở tủ của phòng khác — phải có quy trình chính thức (giao tiếp liên service) để yêu cầu và nhận thông tin.

---

## Phần 2: Ba mô hình kiến trúc database cho microservices

### 1. Database per Service

Đây là nguyên tắc "chuẩn mực" của microservices: mỗi service sở hữu database riêng, có thể đặt trên hạ tầng hosting riêng để tối đa hóa uptime — một máy chủ gặp sự cố sẽ không kéo sập toàn bộ các service khác.

Từ nguyên tắc này, ta có thể mở rộng thành **polyglot persistence**: chọn loại database phù hợp nhất cho từng service, thay vì ép buộc một công nghệ đồng nhất cho tất cả. Cố gắng giữ một stack công nghệ đồng nhất nghe có vẻ gọn gàng, nhưng trong thực tế lại dẫn tới việc "đi đường tắt" và các dự án tích hợp cồng kềnh — khi nhu cầu dùng một công nghệ duy nhất lấn át cơ hội chọn công nghệ tốt nhất cho đúng vấn đề.

**Bốn cái giá phải trả của Database per Service**:

- **Additional cost**: giảm phụ thuộc hạ tầng giữa các service đồng nghĩa cần networking, server, license mạnh mẽ hơn. Cloud có thể giảm bớt chi phí này, nhưng vẫn luôn có một mức giá nhất định.
- **Heterogeneous development stack**: là lợi thế khi đáp ứng đúng nhu cầu nghiệp vụ của từng service, nhưng lại là gánh nặng khi cần tìm nhân sự duy trì nhiều công nghệ khác nhau. Cross-training giữa các team được khuyến nghị nhưng không phải lúc nào cũng hiệu quả — rủi ro có một service được xây bởi nhân sự cũ mà không ai hiện tại có thể bảo trì là hoàn toàn có thật.
- **Data synchronization**: đã nói ở Chapter 7 — eventual consistency đòi hỏi thêm code và hạ tầng để xử lý dữ liệu "lệch pha" tạm thời giữa các database.
- **Transactional handling**: không thể đảm bảo ACID transaction xuyên suốt nhiều database. Đây chính là lý do cần một pattern mới — **Saga pattern** (chương tiếp theo).
- **Communication failure**: vì một service không thể truy cập trực tiếp database của service khác, ta cần giao tiếp đồng bộ (Chapter 3) để hoàn tất một thao tác — điều này lại tạo thêm điểm rủi ro lỗi.

**Nguyên tắc cốt lõi cần khắc ghi**: đừng bao giờ áp dụng một pattern chỉ vì nó "được khuyến nghị". Luôn đánh giá đúng vấn đề cần giải quyết, rồi chọn giải pháp phù hợp nhất trong phạm vi ngân sách cho phép.

### 2. Shared Database

Có những lúc các microservice được phát triển bằng cùng một stack công nghệ giống hệt nhau. Trong trường hợp này, việc chia sẻ tài nguyên để cắt giảm chi phí và đơn giản hóa vận hành dài hạn là hợp lý.

Nhưng vì loose coupling vốn là lợi thế cốt lõi của tính độc lập trong microservices, cần cân nhắc kỹ lưỡng điều gì được phép chia sẻ. Khi chọn hướng này, best practice là thiết lập ranh giới rõ ràng giữa các service:

- **Isolation**: mỗi service duy trì connection pool riêng, tránh can thiệp lẫn nhau, tăng độ ổn định.
- **Configuration**: tinh chỉnh tham số (max pool size, connection timeout, idle time) theo tải và yêu cầu hiệu năng riêng của từng service.
- **Monitoring**: theo dõi connection pool usage để phát hiện connection leak hoặc bão hòa tài nguyên.

Có thể thiết lập thêm **resource quota** ở cấp database: user-level quota (giới hạn CPU, memory, I/O cho từng service), workload management (ưu tiên workload quan trọng vào giờ cao điểm), và alert/throttling khi có breach quota.

Về mặt kỹ thuật quan hệ, hai kỹ thuật phổ biến:

- **Tables-per-service**: định nghĩa bảng tối ưu riêng cho dữ liệu của từng service, chỉ model đúng những bảng cần thiết trong code của service đó. Thường có các bảng denormalized đại diện dữ liệu từ bảng khác để tăng hiệu quả query, phục vụ đúng nhu cầu riêng.
- **Schema-per-service**: dùng schema của database quan hệ để phân loại bảng theo từng service, giúp tinh chỉnh quyền truy cập và giảm gánh nặng quản trị bảo mật theo từng schema.

Cả hai kỹ thuật này giúp giảm nhầm lẫn, đơn giản hóa onboarding cho dev mới, và tinh gọn công việc bảo trì database.

**Trade-off quan trọng nhất**: implement tables/schema-per-service tốn ít tài nguyên nhất (tương đương xây một ứng dụng trên một database), và có ưu điểm dễ implement transaction (vì chỉ dùng một ORM duy nhất để quản lý thao tác trải trên nhiều bảng của nhiều service). **Nhưng** mô hình này giữ lại single point of failure và không scale tốt theo nhu cầu đa dạng của từng service.

Dù khả thi về mặt kỹ thuật, mô hình này **thỏa hiệp đáng kể tính độc lập của service**, dẫn tới coupling chặt khó scale/tiến hóa/deploy độc lập:

- **Increased coupling**: một mục tiêu cốt lõi của microservices là service autonomy. Shared database vi phạm nguyên tắc này bằng cách tạo coupling chặt ở tầng dữ liệu — nhiều service phụ thuộc vào cùng bảng/cấu trúc dữ liệu. Một thay đổi schema của service này có thể vô tình phá vỡ service khác; business logic nhúng trong trigger/stored procedure có thể phục vụ nhiều service cùng lúc, làm phức tạp việc suy luận và bảo trì.
- **Performance bottlenecks and resource contention**: khi số lượng service tăng, hoặc một service có traffic cao hơn, database trở thành nút thắt cổ chai nghiêm trọng. Dù đã thiết lập resource threshold, rủi ro thực tế là hiệu năng toàn hệ thống suy giảm do tranh chấp tài nguyên — một service quá tải có thể khiến các service khác downtime dù chúng không liên quan về mặt chức năng.
- **Reduced deployment agility**: trong kiến trúc decoupled đúng nghĩa, service nên deploy độc lập. Nhưng shared database đòi hỏi thay đổi schema phải được phối hợp cẩn thận giữa mọi service phụ thuộc để tránh lỗi runtime — điều này giới hạn khả năng deploy độc lập, thường cần kế hoạch release đồng bộ với team khác, làm chậm đáng kể vòng đời phát hành phần mềm và triệt tiêu lợi ích năng suất vốn có của microservices.

---

## Phần 3: Phát triển Database — kỹ năng nền tảng của developer hiện đại

Ngày nay, vai trò database developer chuyên trách đã dần chuyển hóa thành **fullstack developer** — một team microservice 2-3 người thường tự phát triển và duy trì cả UI, application code, lẫn database.

Phát triển database không chỉ dừng ở mức độ thoải mái với công nghệ — cần cân nhắc cả kinh nghiệm của team. Rất nhiều dev bỏ qua việc tham vấn nghiệp vụ và hiểu đầy đủ yêu cầu trước khi implement công nghệ — điều này thường dẫn tới thiết kế kém, phải làm lại, và tốn thêm chi phí bảo trì trong suốt vòng đời ứng dụng. Trước khi chốt công nghệ, cần đánh giá kỹ nhu cầu hệ thống, các entity, quy trình nghiệp vụ, và chính sách bảo mật/lưu trữ dữ liệu.

Vì microservices cho phép ta xây dựng ứng dụng và data store nhỏ hơn cho từng phần của toàn hệ thống, ta có cơ hội đặc biệt để phân tích chính xác nhu cầu lưu trữ của service đang tập trung, giảm độ phức tạp tổng thể so với việc xây một database "vạn năng" cho tất cả — nhờ đó giảm biên độ sai sót trong giai đoạn thiết kế.

---

## Phần 4: Relational Database — nền tảng vững chắc dựa trên nguyên tắc ACID

Database quan hệ tồn tại và thống trị ngành công nghệ database trong nhiều năm vì lý do chính đáng: được xây trên các nguyên tắc nghiêm ngặt, bổ trợ cho việc lưu trữ dữ liệu gọn gàng, hiệu quả, đồng thời đảm bảo độ chính xác. Nền tảng của các nguyên tắc này là **ACID**:

- **Atomicity**: mỗi transaction là tất cả-hoặc-không-gì. Nếu bất kỳ phần nào thất bại, toàn bộ transaction rollback để giữ toàn vẹn dữ liệu. Ví dụ: khi bệnh nhân đặt lịch và cần trả phí, ta phải đảm bảo cả việc đặt lịch lẫn thanh toán đều thành công — nếu một trong hai thất bại, ta rollback toàn bộ, không lưu gì cả.
- **Consistency**: đảm bảo transaction đưa database từ trạng thái hợp lệ này sang trạng thái hợp lệ khác, tuân thủ mọi rule/constraint/validation đã định nghĩa. Ví dụ: schema database enforce ràng buộc không cho phép thêm bệnh nhân thiếu họ và tên.
- **Isolation**: đảm bảo các transaction đồng thời thực thi độc lập mà không can thiệp lẫn nhau, ngăn dữ liệu bất thường. Ví dụ: nhờ isolation đúng chuẩn, hai người dùng cố đặt cùng một khung giờ với cùng bác sĩ không thể cùng đặt thành công.
- **Durability**: đảm bảo một khi transaction đã commit, nó vẫn tồn tại vĩnh viễn ngay cả khi hệ thống crash. Ví dụ: sau khi xác nhận lịch hẹn, chi tiết vẫn còn nguyên vẹn dù hệ thống có restart.

### Các hệ quản trị database quan hệ phổ biến

SQL Server, MySQL/MariaDB (mã nguồn mở), PostgreSQL (mã nguồn mở, mạnh mẽ đủ để xử lý từ dự án cá nhân tới data warehouse), Oracle Database (mạnh cho real-time transaction, data warehousing), IBM DB2 (đáng tin cậy cho hệ thống enterprise), và SQLite (nhẹ, không cần server, sống chung file system với ứng dụng — lựa chọn xuất sắc cho ứng dụng mobile-first).

### Primary Key, Foreign Key, và Normalization

Nguyên tắc thiết kế tốt khuyến khích mỗi bảng có một cột primary key luôn mang giá trị duy nhất để định danh bản ghi. Giá trị primary key này được tham chiếu bởi bảng khác dưới dạng foreign key, giúp giảm số lần dữ liệu bị lặp lại giữa các bảng.

Thông qua **normalization**, ta đánh giá dữ liệu nào được chia sẻ giữa các entity và giới thiệu foreign key để chia sẻ dữ liệu hiệu quả trên nhiều bảng. Mỗi foreign key đại diện một tham chiếu/index được tạo giữa các bảng — và một khi mối liên kết giữa primary/foreign key được thiết lập, ta tạo ra một quan hệ áp đặt ràng buộc về giá trị hợp lệ trong cột foreign key. Đây gọi là **referential integrity**.

### Cardinality — ba loại quan hệ

- **One-to-one**: giá trị primary key được tham chiếu đúng một lần ở bảng khác. VD: nếu một bệnh nhân chỉ có một địa chỉ trên hồ sơ, bảng lưu địa chỉ không được tham chiếu nhiều lần tới cùng bệnh nhân đó.
- **One-to-many**: giá trị primary key có thể được tham chiếu nhiều lần ở bảng khác. VD: một khách hàng đặt nhiều đơn hàng, ID của họ xuất hiện nhiều lần trong bảng orders.
- **Many-to-many**: primary key của bảng này được tham chiếu nhiều lần ở bảng kia, và ngược lại. Đây có thể gây nhầm lẫn, và implement trực tiếp sẽ vi phạm referential integrity. Giải pháp: giới thiệu một **linker table** (bảng trung gian) để liên kết các tổ hợp giá trị khóa từ cả hai bảng.

Ví dụ hệ thống phòng khám: `Appointments` liên kết tới `Customers` qua `CustomerId` (one-to-many). Khi mở rộng thêm phòng khám (`Rooms`), ta nhận ra quan hệ many-to-many: nhiều khách hàng đặt lịch hẹn có thể xảy ra ở nhiều phòng khác nhau. Nếu cố liên kết trực tiếp, chi tiết khách hàng hoặc phòng sẽ bị lặp lại để phản ánh mọi tổ hợp có thể. Giải pháp lý tưởng: bảng `Rooms` riêng, bảng `Customers` riêng, và bảng `Appointments` đứng giữa, liên kết cả hai theo nhu cầu thực tế mỗi ngày.

### Nhược điểm của Relational Database

Database quan hệ được thiết kế để enforce ACID như chế độ vận hành mặc định — điều này khiến việc thay đổi cấu trúc và tham chiếu bảng khá rắc rối, đặc biệt khi thay đổi liên quan tới cách các bảng quan hệ với nhau. Một nhược điểm tiềm ẩn khác là hiệu năng với tập dữ liệu lớn — database quan hệ vốn hiệu quả trong lưu trữ/truy xuất/tốc độ, công nghệ không có lỗi, nhưng **thiết kế của ta** sẽ hoặc bổ trợ hoặc kìm hãm tốc độ đó.

Đây chính là trade-off giữa thiết kế database quan hệ đúng chuẩn và việc duy trì tính toàn vẹn quan hệ: nguyên tắc thiết kế ưu tiên normalization, nơi dữ liệu bị trải rộng trên nhiều bảng — tốt cho tới khi ta cần dữ liệu đó và phải duyệt qua nhiều bảng với hàng nghìn bản ghi để lấy nó. Chính khiếm khuyết này khiến NoSQL/document storage database ngày càng phổ biến — chúng tìm cách lưu dữ liệu ở một nơi duy nhất, khiến quá trình truy xuất nhanh hơn nhiều.

---

## Phần 5: Non-Relational (NoSQL) Database — linh hoạt và scale ngang

Database phi quan hệ đã thay đổi đáng kể cách ta nhìn nhận nguyên tắc lưu trữ dữ liệu — đề xuất một kỹ thuật lưu trữ linh hoạt và scale tốt hơn, ưu ái phong cách phát triển agile hơn.

Phát triển agile xoay quanh việc biến đổi dự án khi triển khai — nghĩa là ta không cần scoping/planning quá mức như database quan hệ đòi hỏi ngay từ đầu. Thay vào đó, ta có thể bắt đầu từ ý tưởng hệ thống và dùng data store phi quan hệ cho phần nhỏ đó. Khi hệ thống tiến hóa, data store cũng tiến hóa theo, với rủi ro mất dữ liệu hoặc hao hụt tối thiểu.

NoSQL database được thiết kế từ nền tảng để hỗ trợ **horizontal scalability** và **high availability** — cực kỳ phù hợp cho môi trường microservices nơi service cần scale độc lập và linh hoạt theo tải.

### Sharding — cơ chế cốt lõi cho scale ngang

Cơ chế chính cho phép NoSQL scale ngang là **sharding** — thực hành phân vùng dữ liệu qua nhiều node (shard), mỗi node chịu trách nhiệm cho một tập con của dataset. MongoDB và Cassandra tự động phân phối dữ liệu qua shard dựa trên shard key; Couchbase và Amazon DynamoDB dùng partitioning và consistent hashing để phân phối dữ liệu, đảm bảo cân bằng tải đồng đều. Điều này cho phép scale độc lập — thêm node vào cluster để xử lý tăng traffic/lưu trữ mà không downtime. Cơ chế replication và distributed consensus đảm bảo dữ liệu vẫn khả dụng ngay cả khi node lỗi, và query có thể được route trực tiếp tới shard chịu trách nhiệm, cải thiện thời gian phản hồi.

### Ví dụ document trong hệ thống phòng khám

```json
{
  "AppointmentId": "5d83d288-6404-4cd5-8526-7af6eabd97b3",
  "Date": "2022-09-01 08:000",
  "Room": {
    "Id": "4fff8260-85ce-45cd-919e-35f0aaf7d51e",
    "Name": "Room 1"
  },
  "Customer": {
    "Id": "4fde8261-85df-65cd-819f-45f0bae9f52e",
    "FirstName": "John",
    "LastName": "Higgins"
  }
}
```

Khác với mô hình quan hệ nơi chi tiết bị chia trên nhiều bảng và tham chiếu qua lại, ở đây ta có thể gói gọn toàn bộ chi tiết cần thiết để đánh giá đầy đủ một lịch hẹn trong một document duy nhất.

### Ba loại NoSQL database

- **Document database**: lưu dữ liệu dưới dạng JSON object, hỗ trợ cấu trúc lồng nhau và collection trong một document. VD: MongoDB, Cosmos DB, CouchDB.
- **Key-value database**: lưu dữ liệu dưới dạng cặp key-value đơn giản, giá trị thường lưu dạng chuỗi, không hỗ trợ native đối tượng phức tạp. Thường dùng làm nơi lookup nhanh, như application caching. VD: Redis.
- **Graph database**: lưu dữ liệu dưới dạng node và edge. Node lưu thông tin về đối tượng (người, địa điểm), edge đại diện quan hệ giữa các node — liên kết cách các điểm dữ liệu quan hệ với nhau, khác với cách các bản ghi quan hệ với nhau. VD: Neo4j.

Ưu điểm rõ ràng của document database: không cần duyệt qua nhiều quan hệ/bảng để có được một bản ghi dữ liệu dễ đọc hoàn chỉnh — hữu ích khi cần thao tác đọc hiệu năng cao mà không muốn hy sinh hiệu năng hệ thống bằng query phức tạp.

Nhược điểm rõ ràng: document database làm rất ít (nếu có) để giảm rủi ro dư thừa dữ liệu. Ta sẽ lặp lại chi tiết vốn dĩ là dữ liệu có quan hệ — điều này khiến việc bảo trì dữ liệu về lâu dài có thể trở thành vấn đề. VD: nếu cần thêm một điểm dữ liệu mới cho mỗi khách hàng đã đặt lịch, ta phải duyệt qua toàn bộ bản ghi để thực hiện thay đổi đó — thao tác này sẽ dễ dàng và hiệu quả hơn nhiều nếu dùng mô hình quan hệ.

---

## Phần 6: Chọn công nghệ database — sáu yếu tố cần cân nhắc

Khi chọn bất kỳ công nghệ nào, ta đối mặt với nhiều yếu tố:

- **Maintainability**: công nghệ này dễ bảo trì tới đâu? Có yêu cầu hạ tầng/phần mềm quá mức không? Update/security patch được phát hành thường xuyên không?
- **Extensibility**: công nghệ này có thể dùng tới mức nào để implement phần mềm? Điều gì xảy ra khi yêu cầu thay đổi? Ta muốn công nghệ không quá cứng nhắc, hỗ trợ được sự biến động của nhu cầu nghiệp vụ.
- **Supporting technologies**: stack công nghệ ta dùng có phù hợp nhất không? Càng nhiều thư viện hỗ trợ tích hợp giữa các stack không đồng nhất, vấn đề này càng ít nghiêm trọng, nhưng vẫn cần cân nhắc từ ngày đầu.
- **Comfort level**: team có thoải mái với công nghệ này không? Dùng công nghệ mới luôn thú vị, nhưng cần nhận ra giới hạn và khoảng trống kiến thức của mình.
- **Maturity**: công nghệ này trưởng thành tới đâu? Nên chọn công nghệ đã được thử nghiệm và chứng minh, có cộng đồng/tài liệu hỗ trợ mạnh mẽ.
- **Appropriateness**: cuối cùng, công nghệ này phù hợp tới đâu cho ứng dụng? Cần cân nhắc throughput đọc/ghi và độ trễ tương ứng với yêu cầu dữ liệu của từng service.

### Khi nào chọn Relational Database

- Cần làm việc với báo cáo và query phức tạp.
- Ứng dụng có khối lượng transaction cao.
- Cần tuân thủ ACID.
- Service không tiến hóa nhanh (yêu cầu ổn định).

### Khi nào chọn Non-Relational Database

- Service liên tục tiến hóa, cần data store linh hoạt thích ứng yêu cầu mới mà không gây gián đoạn lớn.
- Dữ liệu có thể không luôn sạch hoặc đạt chuẩn nhất định.
- Cần hỗ trợ scale nhanh chóng với thay đổi code và chi phí tối thiểu.

---

## Phần 7: Chọn ORM — cầu nối giữa code và database

ORM (Object Relational Mapping) là nền tảng để ứng dụng giao tiếp với database. Ta đã khám phá hai ORM phổ biến trong .NET:

- **Entity Framework Core (EF Core)**: bộ tính năng toàn diện, bao gồm change tracking, lazy loading, migration tự động. Có hỗ trợ built-in cho một số NoSQL database, đáng chú ý là Azure Cosmos DB — cho phép làm việc với NoSQL bằng pattern quen thuộc của EF Core.
- **Dapper**: được đội ngũ Stack Overflow phát triển, nổi tiếng cực nhanh, nhẹ, dễ tích hợp cho ứng dụng hiệu năng cao và đơn giản. Thực thi raw SQL và map kết quả trực tiếp sang object C# với overhead tối thiểu — benchmark cho thấy hiệu năng Dapper rất gần với ADO.NET thuần. Không hỗ trợ native NoSQL như EF Core.

Nếu cả EF Core lẫn Dapper đều không hỗ trợ trực tiếp công nghệ database đã chọn, tốt nhất nên xác nhận sự tồn tại của thư viện hỗ trợ ổn định và trưởng thành (VD: MongoDB.NET driver là NuGet package chính thức cho MongoDB).

EF Core liên tục được cải tiến và hiện là mã nguồn mở, cung cấp interface/abstraction xuất sắc giúp giảm nhu cầu viết code đặc thù theo từng database. Đây là tính năng quan trọng cho phép tái sử dụng code trên nhiều công nghệ database khác nhau và giữ tính linh hoạt về công nghệ được dùng — có thể đổi công nghệ database mà không ảnh hưởng ứng dụng chính và hoạt động của nó.

Thông qua EF Core, ta có thể dùng cú pháp LINQ trong C# để thực thi query. EF Core sẽ cố gắng sinh cú pháp SQL hiệu quả nhất, thực hiện thao tác, và trả về dữ liệu dưới dạng object được tạo hình bởi class model tương ứng với bảng — quy trình này gọi là **materialization**:

```csharp
// Khởi tạo kết nối tới database
using var context = new ApplicationDatabaseContext();

// Truy vấn database lấy danh sách bệnh nhân
var patients = await context.Patients.ToListAsync();

// Duyệt qua danh sách và in ra màn hình
foreach (var patient in patients)
{
    Console.WriteLine($"{patient.FirstName} {patient.LastName}");
}
```

Abstraction của EF Core và LINQ cho phép ta viết code C# mà không cần lo lắng về loại database đang được query — cùng đoạn code này hoạt động tương tự cho SQL Server, SQLite, và Azure Cosmos DB. EF Core cũng hỗ trợ đầy đủ dependency injection, khiến việc quản lý connection và garbage collection gần như không còn là vấn đề.

---

## Phần 8: Code-First vs Schema-First — hai kỹ thuật phát triển database

### Code-First

Kỹ thuật này cho phép ta model database song song với code ứng dụng — tạo data model bằng class C# tiêu chuẩn, quản lý thay đổi qua **migration**.

Một migration đánh giá các thay đổi được áp dụng lên database và sinh ra lệnh (cuối cùng trở thành SQL script) để thực hiện các thay đổi đó. Sau khi migration được tạo, một file mới chứa "sổ cái" (ledger) toàn bộ thay đổi cần áp dụng lên database được sinh ra — bao gồm thay đổi tới data context, model/attribute, và constraint. Code sinh ra sẽ khác nhau tùy theo database engine EF Core được cấu hình target.

Class file của migration chứa method `Up` và `Down` — vạch rõ thay đổi nào sẽ áp dụng và thay đổi nào sẽ hoàn tác nếu migration bị revert:

```bash
# Xóa migration cuối (chỉ hoạt động nếu migration đó chưa được áp dụng)
dotnet ef migrations remove

# Liệt kê toàn bộ migration
dotnet ef migrations list

# Rollback database về một migration cụ thể
dotnet ef database update <MigrationName>
```

**Lưu ý quan trọng**: rollback migration có thể dẫn tới mất dữ liệu, đặc biệt nếu migration bị revert bao gồm thao tác xóa bảng/cột. Luôn đảm bảo có backup phù hợp trước khi rollback.

Một khi migration được áp dụng, bảng `__EFMigrationsHistory` được tạo (lần đầu) hoặc cập nhật (các lần sau) — bảng này được kiểm tra mỗi lần để xác định migration nào đã áp dụng, đảm bảo chỉ migration đang chờ (pending) mới được thực thi, tránh thao tác thừa và lỗi tiềm ẩn.

**Bốn ưu điểm chính của Code-First**:

- **Development speed**: viết domain class mà không cần thiết kế database trước — có lợi cho những ai có background lập trình nhưng ít kinh nghiệm thiết kế database.
- **Control over code**: kiểm soát hoàn toàn entity class, code sạch hơn, không bị rối bởi nội dung tự sinh.
- **Version control and migrations**: thay đổi model được track qua code, hỗ trợ version control; migration cho phép cập nhật schema mượt mà khi model tiến hóa, đảm bảo nhất quán giữa code và database.
- **Flexibility in complex relationships**: cho phép diễn đạt quan hệ phức tạp bằng code — hữu ích khi schema chưa được định nghĩa hoặc dự kiến sẽ tiến hóa.

Tuy nhiên, quản lý migration/update database có đường cong học tập dốc — developer cần hiểu vững C# và cấu hình EF Core để quản lý migration hiệu quả. Khi làm việc nhóm với nhiều người cùng tạo migration đồng thời, thay đổi có thể "lệch pha" và dẫn tới hàng giờ triage. Ở môi trường production, điều này càng phức tạp hơn, khuyến nghị quản lý cẩn thận cách migration được áp dụng.

Với developer ưa thích SQL script, EF Core cho phép sinh **idempotent migration script** — đảm bảo cập nhật schema có thể áp dụng an toàn bất kể trạng thái hiện tại của database, chứa các kiểm tra ngăn việc áp dụng lại migration đã thực thi, phù hợp cho việc deploy tới nhiều môi trường ở các giai đoạn migration khác nhau:

```bash
dotnet ef migrations script --idempotent --output "path/to/your/script.sql"
```

Một số database provider có thể có yêu cầu/giới hạn riêng khi sinh idempotent script (VD: đã có báo cáo về vấn đề với PostgreSQL ở một số phiên bản EF Core).

### Schema-First (Database-First)

Kỹ thuật này liên quan tới việc tạo database bằng công cụ quản trị database thông thường trước, sau đó **scaffold** database đó vào ứng dụng. Dùng EF Core, ta sẽ nhận được một database context class đại diện database và toàn bộ object của nó, class cho mỗi bảng và view:

```bash
dotnet ef dbcontext scaffold "Your_Connection_String" Microsoft.EntityFrameworkCore.Sqlite -o Models
```

Với solution có nhiều project, lệnh phức tạp hơn cần chỉ rõ startup project và project chứa database context/model file:

```bash
dotnet ef dbcontext scaffold "Your_Connection_String" Microsoft.EntityFrameworkCore.Sqlite --project YourDataProject --startup-project YourStartupProject --output-dir Models --context-dir Data --context YourDbContextName --force
```

Lệnh này cần chạy lại mỗi khi database thay đổi, kèm flag `--force` để ghi đè file hiện có. Vì thay đổi xảy ra ở tầng database, không có cơ chế tracking sẵn có để biết database đang ở version nào — đây là lý do code-first phổ biến hơn nhiều, vì migration giúp giải quyết chính xác vấn đề này.

---

## Phần 9: Best Practices — Polyglot Persistence, Sharding, và Eventual Consistency

### Polyglot Persistence Strategy

Chiến lược này dùng nhiều công nghệ lưu trữ dữ liệu đa dạng trong cùng một ứng dụng, mỗi cái được chọn để phù hợp với yêu cầu cụ thể. Mỗi service có thể dùng công nghệ database phù hợp nhất với nhu cầu lưu trữ/truy xuất của nó.

**Ba lợi ích cụ thể**:

- **Optimized data management**: cho phép database chuyên biệt được tinh chỉnh cho model dữ liệu và pattern truy cập cụ thể, nâng cao hiệu năng và hiệu quả.
- **Flexibility**: cho phép service tiến hóa độc lập, áp dụng công nghệ lưu trữ mới khi yêu cầu thay đổi.
- **Resilience**: cô lập vấn đề liên quan tới dữ liệu trong phạm vi một service, ngăn ảnh hưởng tới toàn hệ thống.

**Các cân nhắc implement nghiêm túc**:

- **Data consistency**: quản lý tính nhất quán giữa nhiều database khác nhau là thách thức. Implement chiến lược như event sourcing (Chapter 7) có thể giúp — kết hợp với polyglot persistence, nó cho phép audit vững chắc và quản lý dữ liệu theo thời gian. Nó hỗ trợ polyglot persistence bằng cách decouple write model khỏi read model, cho phép projection vào một hoặc nhiều database tối ưu cho query, mỗi cái có thể dùng công nghệ khác nhau.
- **Operational complexity**: duy trì nhiều hệ thống database tăng gánh nặng vận hành, cần team có chuyên môn quản lý công nghệ đa dạng, đồng nghĩa duy trì nhiều môi trường và chiến lược deploy khác nhau cho mỗi công nghệ.
- **Data integration**: tổng hợp dữ liệu từ nhiều nguồn đòi hỏi công cụ và service bổ sung. Vì các database lưu dữ liệu khác nhau, tổng hợp dữ liệu không đồng nhất cần logic biến đổi để chuẩn hóa và tái định dạng thành format thống nhất. Data integration hiệu quả thường cần công cụ chuyên biệt như ETL (Extract, Transform, Load), data pipeline (Apache Kafka, Azure Event Hubs) để stream event qua các service, và data warehouse (Snowflake, Azure Synapse, Amazon Redshift) để tập trung dữ liệu phục vụ query.

### Ví dụ Polyglot Persistence trong hệ thống phòng khám

- **Patient records service**: quản lý nhân khẩu học, bệnh sử, kế hoạch điều trị. Database quan hệ là lựa chọn tốt nhất cho dữ liệu có cấu trúc với quan hệ phức tạp, đảm bảo toàn vẹn dữ liệu và hỗ trợ ACID transaction.
- **Appointment scheduling service**: xử lý đặt/đổi/hủy lịch hẹn. Đã dùng database quan hệ, nhưng có thể cân nhắc NoSQL cho thao tác đọc/ghi nhanh vì toàn bộ chi tiết lịch hẹn có thể lưu trong một document.
- **Documents service**: lưu trữ và truy xuất tài liệu y tế lớn (X-quang, MRI, CT scan). Có thể tận dụng object storage như Amazon S3 hoặc Azure Blob Storage để quản lý file nhị phân lớn, cung cấp lưu trữ scale tốt và truy xuất hiệu quả.

### Database Sharding

Sharding giải quyết thách thức scale một database duy nhất để xử lý khối lượng dữ liệu lớn và tải traffic cao. Đây là một dạng **horizontal partitioning**, trong đó database được chia thành các mảnh nhỏ hơn, dễ quản lý hơn gọi là **shard**. Mỗi shard chứa một tập con dữ liệu, cùng nhau tạo thành toàn bộ dataset.

**Ba lợi ích**:

- **Improved performance**: hệ thống xử lý nhiều thao tác đọc/ghi đồng thời hơn nhờ phân phối dữ liệu qua nhiều shard, giảm nghẽn.
- **Enhanced scalability**: cho phép thêm shard để đáp ứng khối lượng dữ liệu tăng, hỗ trợ scale ngang.
- **Isolation**: lỗi hoặc tải nặng ở một shard không ảnh hưởng trực tiếp tới shard khác, tăng khả năng chịu lỗi.

**Hai kỹ thuật sharding tiêu chuẩn**:

- **Range-based sharding**: phân vùng dữ liệu theo một dải liên tục của một cột cụ thể (thường là identifier hoặc timestamp). Đây là phương pháp dễ implement nhất và tăng hiệu năng cho query nhắm tới một dải đã chỉ định.
- **Hash-based sharding**: áp dụng hàm hash lên một sharding key (VD: user ID, tenant ID) để xác định dữ liệu thuộc shard nào. Với thiết kế tốt, có thể đạt phân phối dữ liệu đồng đều. Phương pháp này phức tạp hơn khi scale, gây khó khăn hơn cho range query.

### Sharding với Azure SQL (Elastic Database Tools)

```csharp
// Cấu hình Shard Map Manager
var shardMapManager = ShardMapManagerFactory.GetSqlShardMapManager(
    shardMapManagerConnectionString,
    ShardMapManagerLoadPolicy.Lazy);

// Lấy shard map
ListShardMap<int> shardMap = shardMapManager.GetListShardMap<int>("CustomerShardMap");

// Lấy shard dựa theo customer ID
Shard shard = shardMap.GetShard(customerId);

// Tạo kết nối tới shard
using (SqlConnection conn = shard.OpenConnection(shardConnectionString, ConnectionOptions.Validate))
{
    // Thực hiện thao tác database
}
```

Tích hợp kỹ thuật sharding này với EF Core còn hạn chế, cần thêm bước để tạo database context object động, dùng đúng connection string:

```csharp
public class TenantDbContext : DbContext
{
    public TenantDbContext(DbConnection connection) : base(new
        DbContextOptionsBuilder<TenantDbContext>()
        .UseSqlServer(connection).Options)
    {
    }
    public DbSet<Patient> Patients { get; set; }
}

public class TenantService
{
    public void PerformTenantOperation(int tenantId)
    {
        var shard = ShardingService.GetShard(tenantId);
        using (var connection = shard.OpenConnection(
            Configuration.GetConnectionString("ShardMapManager"),
            ConnectionOptions.Validate))
        {
            using (var context = new TenantDbContext(connection))
            {
                var patients = context.Patients.ToList();
            }
        }
    }
}
```

### Sharding với Azure Cosmos DB (NoSQL)

Azure Cosmos DB là dịch vụ database NoSQL đa mô hình, phân tán toàn cầu, hỗ trợ sharding nội tại thông qua partitioning. Khi tạo container, ta định nghĩa một **partition key** xác định cách dữ liệu được phân phối qua các partition:

```csharp
await cosmosClient.GetDatabase("PatientsDB")
    .CreateContainerIfNotExistsAsync(
        new ContainerProperties
        {
            Id = "PatientRecords",
            PartitionKeyPath = "/PatientId"
        },
        throughput: 400);
```

Chọn shard key phù hợp là điều tối quan trọng — key nên đảm bảo phân phối đồng đều dữ liệu và pattern truy cập để tránh "hot spot" (điểm nóng quá tải).

### Eventual Consistency — cân bằng thay đổi dữ liệu trong hệ phân tán

Nhắc lại từ Chapter 4 và 7: eventual consistency đảm bảo, nếu đủ thời gian không có update mới, mọi bản sao của một điểm dữ liệu sẽ hội tụ về cùng giá trị. Mô hình này chấp nhận sự không nhất quán tạm thời, đảm bảo tính nhất quán cuối cùng sẽ đạt được.

Phù hợp cho các kịch bản ứng dụng có thể chịu đựng sự không nhất quán tạm thời — VD: đổi bác sĩ được chỉ định cho một lịch hẹn. Cần scope đúng kịch bản nào phù hợp với kỳ vọng "lỏng" này và khi nào update dữ liệu bắt buộc phải xử lý real-time.

### Outbox Pattern — kỹ thuật đảm bảo atomicity giữa DB và message publishing

Đây là pattern phổ biến giải quyết thách thức đảm bảo tính atomic giữa thao tác database và việc publish event. Khi một service cập nhật database và cần publish event thông báo cho service khác, có rủi ro không nhất quán nếu event thất bại publish sau khi transaction đã commit.

Với pattern này, ta implement một **transactional outbox table** trong database của service để tạm lưu event như một phần của cùng transaction sửa đổi business data. Một tiến trình riêng biệt đọc các event đang chờ (pending) từ bảng outbox và publish chúng tới message broker.

Pattern này kết hợp nhiều pattern khác và là cách xuất sắc để quản lý tính nhất quán dữ liệu trong giao tiếp bất đồng bộ giữa các service (Chapter 4) — data store trung gian giúp track message nào đã được gửi thành công và hỗ trợ retry khi cần thiết.

### Năm mức Consistency Level của Azure Cosmos DB

Azure Cosmos DB cung cấp 5 mức consistency được định nghĩa rõ ràng, cho phép developer chọn trade-off phù hợp nhất giữa consistency, availability, và performance:

- **Strong consistency**: đảm bảo thao tác đọc luôn trả về write đã commit gần nhất.
- **Bounded staleness**: đảm bảo thao tác đọc chậm hơn write không quá một khoảng thời gian hoặc số phiên bản đã chỉ định.
- **Session consistency**: đảm bảo tính nhất quán trong phạm vi một session của client, đảm bảo hành vi "đọc chính write của mình".
- **Consistent prefix**: đảm bảo thao tác đọc không bao giờ thấy write sai thứ tự — có thể chưa đầy đủ, nhưng luôn đúng thứ tự.
- **Eventual consistency**: mức đảm bảo yếu nhất. Đọc có thể trả về dữ liệu sai thứ tự hoặc cũ, nhưng các bản sao cuối cùng sẽ hội tụ.

Dù eventual consistency có thể cải thiện hiệu năng và availability, cần cân nhắc rằng thao tác đọc có thể đọc dữ liệu lỗi thời — đảm bảo ứng dụng được thiết kế để xử lý kịch bản này. Trong cấu hình multi-master, các write đồng thời có thể dẫn tới xung đột — Azure Cosmos DB cung cấp cơ chế giải quyết xung đột, bao gồm policy và quy trình tùy chỉnh.

---

## Tổng kết chương

Chương này khám phá các cách tiếp cận khác nhau để quản lý dữ liệu trong kiến trúc microservices. Khác với ứng dụng monolith dựa vào database tập trung, microservices ủng hộ mô hình phi tập trung nơi mỗi service tự quản lý dữ liệu của mình — kéo theo các thách thức về toàn vẹn dữ liệu, giao tiếp liên service, và transaction, được giải quyết qua nhiều chiến lược thiết kế khác nhau.

Ba mô hình quản lý database chính: database-per-service (cho phép polyglot persistence nhưng tăng độ phức tạp/chi phí), shared database (dùng schema/table riêng để giữ phân tách logic nhưng thỏa hiệp tính độc lập), và event-driven data propagation (đã bàn ở Chapter 7).

Ta đã ôn lại ưu/nhược điểm của relational (ACID, mạnh cho tính toàn vẹn dữ liệu) và non-relational database (linh hoạt, scale tốt, ưu tiên tốc độ hơn consistency chặt chẽ). Việc chọn ORM (EF Core vs Dapper) và kỹ thuật phát triển database (code-first vs schema-first) phụ thuộc trực tiếp vào công nghệ database đã chọn và bối cảnh dự án (dự án mới vs làm việc với database legacy).

Cuối cùng, ba kỹ thuật nâng cao: **polyglot persistence** (mỗi service chọn công nghệ tốt nhất cho riêng nó), **database sharding** (phân phối dữ liệu qua nhiều database/server để cải thiện hiệu năng và khả năng chịu lỗi), và **eventual consistency** (cho phép dữ liệu đồng bộ theo thời gian thay vì enforce nhất quán tức thời, hỗ trợ bởi CQRS, event sourcing, và outbox pattern).

**Bài học cốt lõi**: thiết kế chiến lược database cho microservices đòi hỏi lập kế hoạch cẩn thận, cân bằng giữa khả năng mở rộng, khả năng bảo trì, và tính nhất quán dữ liệu — không có một công thức đúng cho mọi trường hợp, chỉ có sự đánh giá đúng đắn nhu cầu thực tế của từng service.

Chủ đề transaction và quản lý transaction trong hệ phân tán — đặc biệt khó khăn — sẽ được giải quyết ở chương tiếp theo bằng **Saga Pattern**.
