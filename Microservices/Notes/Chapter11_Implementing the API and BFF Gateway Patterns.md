# Chapter 11 — API and BFF Gateway Patterns: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| API Gateway | Giao diện thống nhất cho client bên ngoài, trừu tượng hóa các microservice phía sau, xử lý cross-cutting concerns (auth, rate limit, logging...) |
| Reverse Proxy | Server trung gian chuyển tiếp request tới backend, chỉ làm việc ở tầng transport (load balancing, TLS termination, caching, compression), không có business-aware logic |
| BFF (Backend for Frontend) | Backend riêng biệt, tối ưu cho từng loại client (web/mobile/third-party), tránh over/under-fetching |
| Thin Gateway | Gateway chỉ lo routing, auth cơ bản, aggregation tối thiểu — gần giống reverse proxy |
| Thick Gateway | Gateway ôm luôn business logic phức tạp, orchestration, choreography — anti-pattern nếu không kiểm soát |
| Service Mesh | Quản lý giao tiếp NỘI BỘ giữa các service (east-west traffic): service discovery, load balancing, mTLS, retry — qua sidecar proxy |
| YARP | Yet Another Reverse Proxy — reverse proxy dạng middleware .NET, lập trình tùy biến bằng C# |
| Ocelot | API gateway đầy đủ tính năng trên .NET Core: cache, JWT, rate limit, aggregation, circuit breaker (Polly) |
| Route (Ocelot/YARP) | Định nghĩa ánh xạ từ upstream path (client gọi) sang downstream path (service thật) |
| Cluster (YARP) | Tập hợp các destination (địa chỉ instance service) mà một route có thể forward tới |
| QoSOptions (Ocelot) | Cấu hình circuit breaker + timeout dựa trên Polly, gắn theo từng route |
| Rate Limiting | Kỹ thuật giới hạn số request/client trong khoảng thời gian, chống lạm dụng và DoS |
| API Key | Token đơn giản qua header/query string, dùng cho server-to-server hoặc app đơn giản |
| JWT | Token tự chứa (self-contained), mang danh tính + claim, dùng với OAuth 2.0/OpenID Connect |

---

## 2. So sánh API Gateway vs Reverse Proxy vs Service Mesh

| Tiêu chí | Reverse Proxy | API Gateway | Service Mesh |
|---|---|---|---|
| Vị trí | Rìa hệ thống (edge) | Rìa hệ thống (edge) | Bên trong lưới service |
| Loại traffic | Bắc-Nam (client → backend) | Bắc-Nam | Đông-Tây (service → service) |
| Business logic | Không (thuần transport) | Có thể có (routing, aggregation, transform) | Không (chỉ hạ tầng giao tiếp) |
| Xác thực nghiệp vụ (JWT, quota) | Không | Có | Không (mTLS ở tầng vận chuyển) |
| Ví dụ công nghệ | NGINX, HAProxy, Apache | YARP, Ocelot, Kong, Azure APIM | Istio, Linkerd |
| Overhead | Rất thấp | Trung bình (tùy độ dày) | Có thêm sidecar mỗi service |

## 3. So sánh YARP vs Ocelot

| Tiêu chí | YARP | Ocelot |
|---|---|---|
| Bản chất | Middleware reverse proxy .NET | API gateway đầy đủ tính năng |
| Cấu hình | appsettings.json (Routes/Clusters) | ocelot.json (Routes/Aggregates) |
| Cache built-in | Không có sẵn, tự cài | Có sẵn |
| JWT auth | Có, qua ASP.NET Core pipeline chuẩn | Có, khai báo trực tiếp trong route config |
| Rate limiting | Cần thêm AspNetCoreRateLimit (hoặc rate limit riêng của YARP) | Có sẵn `RateLimitOptions` |
| Circuit breaker / retry | Tự cài Polly | Có sẵn qua `Ocelot.Provider.Polly` + `QoSOptions` |
| Aggregation | Không có sẵn, tự viết | Có sẵn `IDefinedAggregator` |
| Mức độ tùy biến code | Rất cao (viết C# trực tiếp trong pipeline) | Cấu hình JSON là chính, custom qua class riêng |
| Khi nào dùng | Cần kiểm soát routing thấp, hiệu năng thô, tùy biến sâu | Cần đầy đủ tính năng gateway "ra lò sẵn", ít code |

## 4. So sánh Aggregator Service (Ch5) vs API Gateway Aggregation (Ch11)

| Tiêu chí | Aggregator Service riêng (Ch5) | Aggregation trong Gateway (Ch11) |
|---|---|---|
| Vị trí đặt logic | Một microservice độc lập | Ngay trong gateway (VD: Ocelot `IDefinedAggregator`) |
| Chi phí mở rộng | Phải tự code thêm mỗi khi có tương tác mới | Khai báo route + aggregator, tận dụng hạ tầng có sẵn |
| Resilience đi kèm | Phải tự tích hợp Polly riêng | Có sẵn qua QoSOptions của route |
| Rủi ro | Trùng lặp code, maintenance overhead | Nguy cơ biến gateway thành "thick gateway" nếu lạm dụng |

## 5. Bốn thuật toán Rate Limiting

| Thuật toán | Cơ chế | Ưu điểm | Nhược điểm |
|---|---|---|---|
| Fixed window | Đếm request trong khoảng cố định, reset theo mốc | Đơn giản, dễ implement | Có thể bị burst ngay ranh giới 2 window |
| Sliding window | Cửa sổ đếm trượt liên tục theo thời gian | Chính xác, mượt hơn fixed window | Phức tạp hơn để cài đặt |
| Token bucket | Token nạp theo tốc độ cố định, hết token thì chặn | Chịu burst tốt | Cần quản lý state (bucket) |
| Leaky bucket | Xử lý request theo tốc độ cố định, dư thì queue/drop | Làm mượt traffic đầu ra | Có thể trì hoãn request hợp lệ khi buffer đầy |

---

## Checklist kỹ thuật khi triển khai

### Khi dựng API Gateway nói chung
- [ ] Đã xác định rõ gateway chỉ lo routing/policy, KHÔNG chứa domain-specific logic (tránh thick gateway)
- [ ] Đã có kế hoạch redundancy: nhiều gateway instance sau load balancer, tránh single point of failure
- [ ] Đã phân biệt rõ phạm vi Gateway (north-south) và Service Mesh (east-west) nếu hệ thống có cả hai
- [ ] Đã đánh giá build custom vs dùng managed gateway (cân nhắc vendor lock-in)
- [ ] Đã có health check/service discovery để gateway route vòng qua instance không khỏe

### Khi dùng YARP
- [ ] Mỗi route có `RouteId` (ngầm định qua key) và `ClusterId` tương ứng
- [ ] Đã cấu hình `Match.Path` đúng pattern, kiểm tra route conflict theo độ cụ thể/`Order`
- [ ] Đã đăng ký `app.UseAuthentication()` TRƯỚC `app.MapReverseProxy()`
- [ ] Nếu cần API key/JWT, đã chèn middleware kiểm tra TRƯỚC khi forward

### Khi dùng Ocelot
- [ ] Đã cấu hình đủ `DownstreamPathTemplate`, `DownstreamScheme`, `DownstreamHostAndPorts`, `UpstreamPathTemplate`, `UpstreamHttpMethod` cho mỗi route
- [ ] Đã thêm `Ocelot.Provider.Polly` và cấu hình `QoSOptions` (ExceptionsAllowedBeforeBreaking, DurationOfBreak, TimeoutValue) cho các route quan trọng
- [ ] Nếu cần gộp nhiều service, đã khai báo `Aggregates` + implement `IDefinedAggregator`
- [ ] Đã cấu hình `AuthenticationOptions` (AuthenticationProviderKey, AllowedScopes) cho route cần bảo vệ
- [ ] Đã cấu hình `RateLimitOptions` (EnableRateLimiting, Period, Limit) cho route cần giới hạn tần suất
- [ ] Program.cs có đủ thứ tự: `AddJsonFile("ocelot.json")` → `AddOcelot().AddPolly()` → `UseAuthentication()` → `UseAuthorization()` → `await UseOcelot()`

### Khi triển khai BFF
- [ ] Mỗi loại client (web/mobile) có file cấu hình gateway riêng (VD: ocelot.WebPortal.json, ocelot.MobilePortal.json)
- [ ] Dùng chung một base Docker image, mount riêng file cấu hình qua volume cho từng BFF instance
- [ ] Đã xác định rõ dữ liệu/aggregation nào cần khác nhau giữa các BFF (tránh over-fetching/under-fetching)

### Khi triển khai bảo mật + rate limiting
- [ ] API key không hardcode — lưu trong secrets manager/biến môi trường
- [ ] JWT: đã cấu hình đúng `Authority`, `Audience`, `RequireHttpsMetadata = true`
- [ ] Rate limiting: đã cấu hình `RealIpHeader` đúng nếu chạy sau proxy/load balancer khác
- [ ] Rate limiting: đã bật `AddMemoryCache()` (bắt buộc để middleware đếm request)
- [ ] Đã xác nhận response khi vượt hạn mức trả về đúng 429 + header Retry-After

---

## Anti-patterns / Lưu ý cần tránh

1. **Thick Gateway**: nhồi business logic, quy tắc nghiệp vụ vào gateway thay vì để trong microservice — phá vỡ tính tự trị và phi tập trung của kiến trúc.
2. **Single instance gateway trong production**: không có redundancy/failover → gateway sập là cả hệ thống sập.
3. **Nhầm lẫn API Gateway với Service Mesh**: dùng gateway để giải quyết vấn đề giao tiếp nội bộ giữa các service (đáng lẽ là việc của mesh), hoặc ngược lại.
4. **Dùng reverse proxy trần (NGINX/HAProxy thô) để đóng vai gateway đầy đủ**: thiếu khả năng xử lý JWT, quota, aggregation — không đủ cho nhu cầu API gateway thực thụ.
5. **Rate limiting không tính IP thật khi sau proxy**: quên cấu hình `RealIpHeader`/`X-Forwarded-For` khiến mọi client bị tính chung một IP (của proxy) → rate limit sai.
6. **StackBlockedRequests = true không cân nhắc kỹ**: có thể gây hiệu ứng khóa dây chuyền kéo dài thời gian client bị chặn.
7. **Vendor lock-in với managed gateway**: chọn commercial gateway mà không đánh giá kỹ ràng buộc thiết kế, dẫn đến khó migrate sau này.

---

## Từ vựng chuyên ngành (Anh - Việt)

| Thuật ngữ | Nghĩa |
|---|---|
| API Gateway | Cổng API, lớp trung gian thống nhất truy cập microservices |
| Backend for Frontend (BFF) | Backend chuyên biệt cho từng loại frontend |
| Reverse Proxy | Proxy ngược, chuyển tiếp request tới backend |
| Cross-cutting concerns | Các mối quan tâm xuyên suốt (auth, logging, rate limit...) |
| Service Discovery | Cơ chế phát hiện/định vị địa chỉ service |
| Service Mesh | Lưới dịch vụ, quản lý giao tiếp nội bộ giữa service |
| Circuit Breaker | Bộ ngắt mạch, ngăn gọi tiếp vào service đang lỗi |
| Rate Limiting | Giới hạn tần suất truy cập |
| Fan-out | Một request tách thành nhiều lời gọi song song |
| Chokepoint | Điểm chặn/thắt cổ chai dùng để kiểm soát tập trung |
| Upstream | Địa chỉ phơi ra cho client (trong Ocelot) |
| Downstream | Địa chỉ thật của microservice phía sau |
| Thin/Thick Gateway | Gateway mỏng (chỉ routing)/gateway dày (ôm business logic) |
| Token Bucket / Leaky Bucket | Thuật toán rate limit kiểu "xô token"/"xô rò rỉ" |
| Sidecar Proxy | Proxy đi kèm mỗi service trong service mesh |
| Trust Boundary | Ranh giới tin cậy giữa các hệ thống |
| SSL/TLS Termination | Điểm kết thúc xử lý mã hóa SSL/TLS |

---

## Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: API Gateway khác Reverse Proxy ở điểm nào?**
→ Reverse proxy chỉ làm việc ở tầng transport: load balancing, TLS termination, caching, compression — không có business-aware logic. API gateway có thêm khả năng xác thực JWT/API key, aggregation, transform payload, và các cross-cutting concern khác. Reverse proxy có thể coi là "thành phần con" hoặc nền tảng để xây gateway lên trên (như YARP).

**Q2: API Gateway khác Service Mesh ở điểm nào?**
→ Gateway quản lý traffic bắc-nam (client bên ngoài → hệ thống), đứng ở rìa. Service mesh quản lý traffic đông-tây (giữa các service nội bộ), hoạt động qua sidecar proxy, lo service discovery, load balancing, mTLS, retry. Hai thứ bổ trợ nhau, không thay thế nhau.

**Q3: Thick Gateway là gì và tại sao là anti-pattern?**
→ Là gateway ôm quá nhiều business logic, orchestration, choreography phức tạp thay vì chỉ routing/policy đơn thuần. Nó phá vỡ tính tự trị của từng microservice, khiến gateway trở thành một "monolith trá hình" ở tầng hạ tầng, khó bảo trì và là điểm nghẽn cả về logic lẫn khả năng chịu lỗi.

**Q4: Aggregation trong Ocelot khác gì so với Aggregator Service tự viết ở Chapter 5?**
→ Về bản chất tư duy giống nhau: gộp nhiều lời gọi downstream thành một response. Khác biệt là NƠI đặt logic — Aggregator Service là một microservice riêng phải tự bảo trì, tự tích hợp resilience; còn Ocelot Aggregation nằm ngay trong gateway, tận dụng luôn QoSOptions/circuit breaker có sẵn theo route, giảm chi phí phát triển khi có thêm tương tác mới.

**Q5: Khi nào chọn YARP, khi nào chọn Ocelot?**
→ YARP phù hợp khi cần kiểm soát routing ở mức thấp, hiệu năng cao, và muốn tùy biến sâu bằng C# thuần (ít ràng buộc bởi cấu hình JSON sẵn). Ocelot phù hợp khi cần đầy đủ tính năng gateway "ra lò sẵn" — cache, JWT, rate limit, aggregation, circuit breaker — với ít code tùy biến hơn, cấu hình chủ yếu qua JSON.

**Q6: Circuit breaker trong QoSOptions của Ocelot hoạt động ra sao?**
→ `ExceptionsAllowedBeforeBreaking` quy định số lỗi liên tiếp tối đa trước khi circuit chuyển sang Open (chặn ngay các request tiếp theo). `DurationOfBreak` là thời gian circuit ở trạng thái Open trước khi chuyển Half-Open để thử một request dò đường. `TimeoutValue` là deadline cứng cho mỗi request; vượt ngưỡng bị tính là một lỗi, cộng dồn vào bộ đếm exception.

**Q7: BFF Pattern giải quyết vấn đề gì mà một API Gateway chung không giải quyết được?**
→ Một gateway chung buộc mọi client theo cùng một hợp đồng dữ liệu, dễ dẫn tới over-fetching (client nhận dư dữ liệu không cần) hoặc under-fetching (phải gọi thêm nhiều lần). BFF tạo backend riêng cho từng loại client, tối ưu đúng cấu trúc dữ liệu, giảm độ phức tạp xác thực, và cho phép mỗi frontend tiến hóa độc lập mà không ảnh hưởng frontend khác.

**Q8: Tại sao rate limiting cần cấu hình đúng RealIpHeader khi hệ thống chạy sau reverse proxy/load balancer?**
→ Nếu không chỉ định đúng header (VD: `X-Forwarded-For`), middleware rate limiting sẽ lấy nhầm địa chỉ IP của chính proxy/load balancer làm "IP client", khiến toàn bộ client thật bị gộp chung vào một bộ đếm — dẫn đến rate limit áp dụng sai, hoặc chặn oan tất cả người dùng cùng lúc.

---

## Bài tập thực hành đề xuất

1. Dựng lại một gateway YARP đơn giản định tuyến tới 2 service giả lập (dùng `dotnet new webapi` tạo 2 API dummy trả JSON tĩnh), xác nhận route/cluster hoạt động đúng qua browser.
2. Chuyển đổi gateway trên sang Ocelot, thêm `QoSOptions` với `ExceptionsAllowedBeforeBreaking = 2`, cố tình làm một service chậm/lỗi để quan sát circuit breaker trip.
3. Viết một `IDefinedAggregator` gộp dữ liệu từ 2 route giả lập (VD: appointments + doctors) thành một response tổng hợp.
4. Cấu hình `RateLimitOptions` giới hạn 3 request/5 giây trên một route, dùng công cụ như Postman hoặc curl loop để xác nhận nhận được 429 sau khi vượt hạn mức.
5. Dựng docker-compose với 2 instance Ocelot (web-bff, mobile-bff) dùng chung image nhưng mount 2 file cấu hình khác nhau, xác nhận mỗi BFF chỉ expose đúng tập route được định nghĩa riêng cho nó.

---

*Ghi chú: Chương tiếp theo (Chapter 12) sẽ chuyển sang Micro Frontends — mở rộng tinh thần microservices lên chính lớp giao diện người dùng, tiếp nối trực tiếp từ ý tưởng BFF vừa học ở chương này.*
