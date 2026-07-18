# Chapter 5 — Aggregator Pattern: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| **Aggregator Pattern** | Service trung gian tổng hợp dữ liệu từ nhiều microservice, trả về 1 response thống nhất cho client |
| **API Gateway** | Biến thể của Aggregator, thêm routing, rate limiting, authentication tập trung |
| **Backend for Frontend (BFF)** | Aggregator riêng biệt thiết kế cho từng loại client (web/mobile/desktop) |
| **Graceful Degradation** | Khi 1 service lỗi, vẫn trả về dữ liệu một phần/fallback thay vì fail toàn bộ |
| **Task.WhenAll** | Cơ chế C# chạy nhiều Task song song, chờ tất cả hoàn tất |
| **Response Compression** | Kỹ thuật nén payload response để giảm băng thông (Gzip, Brotli, Deflate) |
| **In-memory Caching** | Cache lưu trong bộ nhớ ứng dụng, volatile — mất khi restart |
| **Single Point of Failure (SPOF)** | Điểm lỗi duy nhất — Aggregator có thể trở thành SPOF mới nếu thiết kế sai |

---

## 2. Vấn đề Aggregator giải quyết

| Không có Aggregator | Có Aggregator |
|---|---|
| UI tự gọi N service riêng lẻ | UI chỉ gọi 1 endpoint duy nhất |
| UI tự xử lý lỗi từng service | Aggregator tập trung xử lý lỗi, trả response thống nhất |
| UI coupling chặt với từng service (endpoint, schema, auth) | UI chỉ biết địa chỉ Aggregator, không cần biết chi tiết bên dưới |
| Độ trễ cộng dồn nếu gọi tuần tự | Aggregator gọi song song (`Task.WhenAll`), giảm độ trễ tổng |
| Sửa 1 service phải sửa UI theo | Aggregator hấp thụ thay đổi, UI không bị ảnh hưởng |

---

## 3. Bốn nhóm lợi ích — bảng ôn nhanh cho phỏng vấn

| Nhóm | Lợi ích chính |
|---|---|
| **Security** | Giảm attack surface, tập trung auth, giấu credential khỏi UI, sanitize dữ liệu |
| **Optimization** | Giảm network round-trip, gọi song song, caching tại tầng aggregator |
| **Failure management** | Trả partial data khi 1 service lỗi, tập trung retry/logging, UI không cần tự xử lý lỗi |
| **Data transformation** | Gộp dữ liệu nhiều service thành 1 object, che schema thay đổi, thêm field tính toán |

---

## 4. Cái giá phải trả và cách khắc phục

| Rủi ro | Giải pháp thiết kế |
|---|---|
| Aggregator down → mất quyền truy cập toàn bộ dữ liệu dù service gốc vẫn khỏe | Horizontal scaling: load balancer + nhiều instance + dynamic scaling rules (CPU, latency, queue depth) |
| Traffic tăng đột biến làm nghẽn Aggregator | Asynchronous processing: dùng message queue cho thao tác không nhạy cảm về độ trễ |
| Tranh chấp tài nguyên khi cần tracking state | Sharding/partitioning theo khách hàng, khu vực, loại API |

---

## 5. Checklist kỹ thuật khi implement Aggregator trong .NET

- [ ] Đã dùng `Task.WhenAll` để gọi song song thay vì `await` tuần tự từng service chưa?
- [ ] DTO có chọn đúng kiểu (`record` cho immutable, `class` cho cần mutate sau khởi tạo) không?
- [ ] Có kiểm tra `null` sau khi await từng Task (VD: `if (patient is null) return NotFound();`) không?
- [ ] Endpoint tương ứng ở service gốc đã có sẵn chưa, hay cần bổ sung (VD: `GetAppointmentsByPatientId`)?
- [ ] Đã đăng ký từng `HttpClient` qua `AddHttpClient<T>()` (tránh socket exhaustion — nhắc lại từ Chapter 3) chưa?

## 6. Checklist Best Practices (5 nhóm)

- [ ] **Asynchronous processing**: mọi cuộc gọi tới service downstream có chạy song song khi độc lập với nhau không?
- [ ] **Caching**: dữ liệu ít thay đổi có được cache không? Đã cân nhắc TTL hợp lý (VD 5 phút) chưa? (Lưu ý: in-memory cache volatile, không phải giải pháp dài hạn)
- [ ] **Compression**: đã bật `AddResponseCompression` với Gzip/Brotli cho payload lớn chưa?
- [ ] **Error handling**: đã phân loại đúng transient/permanent/unexpected error và quyết định chiến lược graceful degradation chưa?
- [ ] **Logging/monitoring**: Aggregator có log tập trung đủ chi tiết để trace lỗi giữa nhiều service không?

---

## 7. Ba loại lỗi khi gọi API — bảng ôn nhanh

| Loại lỗi | Ví dụ | Có nên retry? |
|---|---|---|
| **Transient** | Network timeout, rate limiting | Có — thường tự hết |
| **Permanent** | 404 Not Found, 401 Unauthorized | Không — retry vô ích |
| **Unexpected** | Lỗi deserialize, null reference | Tùy trường hợp — cần xử lý riêng |

---

## 8. So sánh 3 thuật toán nén — bảng ôn nhanh

| Thuật toán | Đặc điểm |
|---|---|
| **Gzip** | Cân bằng tỷ lệ nén/tốc độ, phổ biến nhất |
| **Brotli** | Tỷ lệ nén tốt hơn Gzip, đặc biệt cho nội dung tĩnh (HTML/CSS/JS) |
| **Deflate** | Tương tự Gzip, implementation khác, mức thấp hơn |

---

## 9. Từ vựng chuyên ngành (Anh - Việt) cần thuộc để phỏng vấn

| Thuật ngữ | Nghĩa |
|---|---|
| Aggregator Pattern | Pattern tổng hợp dữ liệu từ nhiều service thành 1 response |
| API Gateway | Cổng vào trung tâm, có thêm routing/rate limiting/auth |
| Backend for Frontend (BFF) | Aggregator riêng cho từng loại client |
| Graceful Degradation | Suy giảm nhẹ nhàng — vẫn hoạt động một phần khi có lỗi |
| Partial Data | Dữ liệu trả về không đầy đủ do 1 phần nguồn bị lỗi |
| Network Overhead | Chi phí phát sinh do nhiều lần gọi mạng |
| Round-trip | Một lượt gửi request và nhận response |
| Response Compression | Nén dữ liệu response để giảm băng thông |
| Sanitize | Làm sạch/kiểm tra dữ liệu trước khi trả về client |
| Attack Surface | Bề mặt tấn công — số điểm có thể bị khai thác |
| Horizontal Scaling | Mở rộng theo chiều ngang — thêm instance |
| Sharding/Partitioning | Chia nhỏ workload theo tiêu chí (khách hàng, khu vực...) |
| Load Shedding | Kỹ thuật chủ động từ chối bớt request khi quá tải để bảo vệ hệ thống |

---

## 10. Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Aggregator Pattern giải quyết vấn đề gì trong kiến trúc microservices?**
→ Giải quyết vấn đề UI phải tự gọi trực tiếp nhiều microservice, dẫn tới coupling chặt, độ trễ cộng dồn, và logic xử lý lỗi rời rạc. Aggregator đứng giữa UI và các service, gom dữ liệu, tổng hợp thành 1 response, giúp UI chỉ cần gọi 1 endpoint duy nhất.

**Q2: Aggregator có nhược điểm gì, và làm sao khắc phục?**
→ Aggregator có thể trở thành single point of failure mới — nếu nó down, client mất quyền truy cập toàn bộ dữ liệu dù các service gốc vẫn hoạt động bình thường. Khắc phục bằng horizontal scaling (nhiều instance + load balancer), asynchronous processing (message queue cho thao tác không nhạy cảm về độ trễ), và sharding/partitioning.

**Q3: Vì sao nên dùng `Task.WhenAll` thay vì `await` tuần tự khi gọi nhiều service trong Aggregator?**
→ Vì các cuộc gọi service độc lập với nhau có thể chạy song song. `await` tuần tự cộng dồn thời gian của tất cả các cuộc gọi (VD: 4 service x 100ms = 400ms), trong khi `Task.WhenAll` chạy đồng thời, tổng thời gian chỉ bằng thời gian của cuộc gọi chậm nhất (~100ms).

**Q4: Phân biệt Aggregator Pattern, API Gateway, và Backend for Frontend (BFF).**
→ Cả ba đều dùng chung triết lý gom dữ liệu và giấu độ phức tạp. Aggregator là khái niệm tổng quát nhất — tổng hợp dữ liệu từ nhiều service. API Gateway mở rộng thêm routing, rate limiting, authentication tập trung cho toàn hệ thống. BFF là một Aggregator được thiết kế riêng cho từng loại client (web, mobile, desktop) vì mỗi loại cần định dạng dữ liệu khác nhau.

**Q5: Graceful Degradation là gì và tại sao quan trọng trong Aggregator?**
→ Là khả năng hệ thống vẫn hoạt động ở một mức độ nào đó khi một phần bị lỗi, thay vì fail hoàn toàn. Trong Aggregator, nếu 1 microservice downstream lỗi, Aggregator vẫn có thể trả về dữ liệu một phần (partial data) từ các service còn khỏe mạnh, giữ ứng dụng vẫn dùng được thay vì làm sập toàn bộ trải nghiệm người dùng.

**Q6: Khi nào nên dùng `record` và khi nào nên dùng `class` cho DTO trong Aggregator?**
→ Dùng `record` khi dữ liệu là immutable (không cần thay đổi sau khi khởi tạo) — phù hợp value object. Dùng `class` khi cần mutate dữ liệu sau khi khởi tạo — ví dụ trong ví dụ Chapter 5, DTO `AppointmentByPatientId` cần cập nhật `DoctorName` sau khi gọi song song tới Doctor Service, nên phải dùng `class`.

**Q7: Ba loại lỗi khi gọi API qua Aggregator là gì, và cách xử lý khác nhau ra sao?**
→ Transient error (timeout, rate limit) nên retry vì thường tự hết. Permanent error (404, 401) không nên retry vì vô ích. Unexpected error (deserialize fail, null reference) cần xử lý tùy ngữ cảnh cụ thể. Aggregator nên catch exception từ `HttpClient`, trả về giá trị mặc định/partial data để lỗi 1 service không ảnh hưởng tới các cuộc gọi khác.

**Q8: Vì sao Response Compression đặc biệt hữu ích cho Aggregator Service?**
→ Vì Aggregator tổng hợp dữ liệu từ nhiều service nên payload trả về thường lớn hơn payload của từng service riêng lẻ. Nén (đặc biệt Brotli cho nội dung text-based) có thể giảm kích thước payload đáng kể (tới 200%), tiết kiệm băng thông và cải thiện thời gian phản hồi cho client.

---

## 11. Bài tập thực hành đề xuất

1. Dựng thử một `AggregatorService` gọi song song tới 2 Web API giả lập (dùng `Task.Delay` để mô phỏng độ trễ mạng), đo thời gian phản hồi khi gọi tuần tự so với `Task.WhenAll`.
2. Thêm in-memory caching (`IMemoryCache`) vào Aggregator cho endpoint `patient-dashboard/{id}`, kiểm tra hành vi khi cache hit/miss.
3. Bật Response Compression (Gzip + Brotli) cho Aggregator, so sánh kích thước payload trước/sau khi nén với một response JSON lớn (VD: danh sách 100+ item).
4. Thiết kế graceful degradation: giả lập 1 trong các service downstream trả lỗi 500, đảm bảo Aggregator vẫn trả về `PatientDashboard` với các field còn lại có dữ liệu, field bị lỗi để `null` kèm thông tin lỗi rõ ràng.

---

*Ghi chú: Chương tiếp theo sẽ tiếp tục khám phá các pattern thiết kế khác giúp xây dựng hệ thống microservices vững chắc hơn.*
