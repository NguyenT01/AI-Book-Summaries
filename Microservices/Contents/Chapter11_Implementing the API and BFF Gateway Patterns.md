# Bài học: Khi client phải nhớ tên mọi cánh cửa — và cách dựng một cổng chung duy nhất

> Chương 11 — Implementing the API and BFF Gateway Patterns
> Ví dụ xuyên suốt: hệ thống quản lý phòng khám (Appointments, Patients, Doctors, Documents...) — lần này ta xây một lớp cổng đứng trước toàn bộ các service đó, để client web và mobile không còn phải "biết mặt" từng service riêng lẻ.

---

## Phần 1: Vấn đề thực tế — client đang gánh việc không phải của nó

Nhớ lại hồi Chapter 5, khi bàn về Aggregator Pattern, mình đã nói: cứ để nhiều service tồn tại độc lập thì UI sẽ phải tự đi gọi từng cái rồi gộp kết quả lại. Aggregator giải quyết được bài toán "gộp dữ liệu", nhưng nó chỉ là một service phụ trợ — bản thân client vẫn phải biết địa chỉ của aggregator, và nếu hệ thống có thêm nhiều luồng nghiệp vụ khác chưa được aggregator hỗ trợ, client vẫn phải tự mò đường tới thẳng service gốc.

Giờ hãy hình dung một buổi sáng, đội frontend của bạn báo cáo: ứng dụng web đang gọi trực tiếp tới 6 địa chỉ khác nhau — appointments-svc, patients-svc, doctors-svc, documents-svc, billing-svc, notification-svc. Mỗi khi một service đổi domain, đổi cổng, hoặc thêm một bước xác thực mới, đội frontend lại phải sửa code, build lại, deploy lại. Tệ hơn, ứng dụng mobile cũng phải lặp lại y hệt logic gọi API đó, dẫn đến hai bộ code gần như giống nhau nhưng bảo trì độc lập. Đây chính là cái giá của việc để client tự "biết" toàn bộ cấu trúc backend — nó biến phần lẽ ra phải đơn giản nhất trong hệ thống (client) thành nơi gánh vác trách nhiệm điều phối, retry, xử lý lỗi từng phần — những việc không thuộc về nó.

Về bản chất, một ứng dụng service-oriented truyền thống có 3 lớp: client (thứ người dùng thấy, chỉ tiêu thụ dữ liệu), server (nơi chứa business logic), và database (nơi lưu dữ liệu). Trong kiến trúc monolith, cả 3 lớp này gói gọn trong một khối, client chỉ cần biết một địa chỉ duy nhất. Nhưng khi ta tách thành microservices, server không còn là MỘT khối nữa — nó vỡ ra thành nhiều mảnh, mỗi mảnh một địa chỉ, một hợp đồng API riêng. Và nếu không có gì đứng ra "dán" các mảnh đó lại thành một mặt tiền duy nhất, client sẽ phải tự làm việc dán ghép đó — bằng cách tuần tự hóa các lời gọi, quản lý phụ thuộc giữa chúng, xử lý retry, và gộp các phản hồi rời rạc. Điều này không chỉ làm phình to logic frontend, mà còn khiến frontend bị khóa chặt vào cấu trúc nội bộ backend — chỉ cần route hoặc hợp đồng của một service đổi, hiệu ứng domino kéo sang tận client.

### Analogy: Tòa nhà văn phòng nhiều công ty và một quầy lễ tân duy nhất

Hãy tưởng tượng một tòa nhà cao tầng, mỗi tầng là một công ty riêng biệt — công ty kế toán, công ty pháp lý, công ty IT. Nếu không có quầy lễ tân, mọi khách đến phải tự biết công ty nào ở tầng nào, phải tự bấm đúng thang máy, tự đi tìm đúng cửa, và nếu công ty đó chuyển tầng, khách phải tự cập nhật thông tin đó. Quầy lễ tân (API gateway) giải quyết việc này: khách chỉ cần bước vào sảnh, nói "tôi cần gặp phòng kế toán", lễ tân biết chính xác tầng nào, phòng nào, và hướng dẫn khách đi đúng đường — thậm chí có thể gọi trước để "gộp" luôn một cuộc hẹn liên tầng (kế toán + pháp lý) nếu khách cần cả hai. Khách không cần biết tòa nhà có bao nhiêu tầng, ai đang ở đâu — họ chỉ cần nhớ một địa chỉ: sảnh chính.

---

## Phần 2: API Gateway là gì, và tại sao nó "được việc" hơn Aggregator tự viết

API Gateway đóng vai trò một giao diện thống nhất (unified interface) cho mọi client bên ngoài — nó trừu tượng hóa toàn bộ độ phức tạp của các microservices phía sau, đồng thời xử lý luôn các mối quan tâm xuyên suốt (cross-cutting concerns) như xác thực, giới hạn tần suất request (rate limiting), và ghi log. Gateway đứng giữa client và các service, phơi ra cho client một base URL duy nhất — với client, đó là "một API service thống nhất"; với các microservices phía sau, gateway đóng vai trò một đường dẫn (conduit), chuyển tiếp request tới đúng nơi cần đến.

Điểm đáng chú ý: gateway KHÔNG phải một khái niệm hoàn toàn mới so với Aggregator ở Chapter 5 — nó thực chất là một bước tiến hóa. Khi logic điều phối và gộp dữ liệu (orchestration, aggregation) nằm trong một aggregator service tự viết, mỗi khi có thêm một tương tác service mới, đội dev phải tự tay mở rộng logic đó — dễ dẫn đến trùng lặp code và gánh nặng bảo trì. Trong khi đó, một API gateway đúng nghĩa tập trung hóa toàn bộ logic điều phối này ngay tại lớp hạ tầng, giúp việc xử lý nhiều request backend, gộp kết quả, và cô lập logic phức tạp ra khỏi từng microservice trở nên đơn giản hơn nhiều — kết quả là các service phía sau giữ được sự thuần khiết, chỉ tập trung vào domain logic của riêng mình.

### Những việc gateway gánh thay cho toàn hệ thống

| Nhóm việc | Gateway làm gì | Vì sao tập trung hóa tốt hơn để rải rác |
|---|---|---|
| Centralized logging | Ghi log tập trung toàn bộ traffic, theo dõi thành công/lỗi từ các service phía sau | Tránh phải cài đặt logging lặp lại ở từng service, tránh log bị phân tán, "ồn ào" khắp nơi |
| Caching | Lưu tạm dữ liệu để giảm số lần gọi service gốc, dùng cả khi service gốc tạm thời offline (partial failure) | Tăng hiệu năng đọc trên các endpoint traffic cao, tăng khả năng chịu lỗi |
| Security | Tập trung xác thực/ủy quyền, có thể whitelist theo IP | Gỡ gánh nặng xác thực khỏi từng service riêng lẻ |
| Service monitoring | Chủ động health-check các service phía sau, định tuyến vòng qua instance không khỏe | Gateway biết tình trạng sức khỏe TRƯỚC KHI chuyển tiếp request, tránh gọi vào chỗ đã hỏng |
| Service discovery | Giữ một "sổ đăng ký" toàn bộ địa chỉ service phía sau | Client chỉ cần biết một địa chỉ ổn định duy nhất |
| Rate limiting | Giới hạn tần suất gọi từ cùng một nguồn, phòng chống DoS | Áp dụng luật chung ngay tại cửa ngõ, không cần từng service tự lo |
| Scalability | Hỗ trợ horizontal scaling, load balancing, tích hợp auto-scaling | Aggregator tự viết cần thêm code thủ công để làm được điều tương tự |

Điều cốt lõi cần nhớ: gateway lấy đi phần lớn trách nhiệm khỏi client, giúp việc scale và đa dạng hóa client code (web, mobile, third-party) trở nên dễ dàng hơn rất nhiều.

**Bài học cốt lõi: Aggregator là một service bạn tự viết để giải quyết MỘT bài toán gộp dữ liệu cụ thể; Gateway là một lớp hạ tầng đứng ra giải quyết TOÀN BỘ các mối quan tâm xuyên suốt của hệ thống — bao gồm cả việc làm những gì Aggregator từng làm.**

---

## Phần 3: Cái giá phải trả — API Gateway không phải viên đạn bạc

Không có pattern nào miễn phí, và gateway cũng vậy — nó mang theo ba rủi ro rất thật:

1. **Single point of failure**: Gateway là cửa ngõ TRUNG TÂM cho mọi request. Nếu không thiết kế cho khả năng chịu lỗi và tính sẵn sàng cao, một sự cố ở gateway đồng nghĩa cả hệ thống sập theo. Cách khắc phục kinh điển: triển khai nhiều instance gateway phía sau một load balancer, không bao giờ chạy một instance duy nhất trong production.

2. **Nguy cơ trở thành "thick gateway"**: Nếu domain logic hoặc business rule bị rò rỉ vào gateway thay vì ở lại trong microservice, gateway sẽ phình to dần thành một khối logic khổng lồ — đi ngược lại chính tinh thần phi tập trung của microservices. Nguyên tắc sống còn: giữ gateway càng "mỏng" càng tốt, chỉ lo routing và policy, còn domain-specific logic luôn thuộc về microservice.

3. **Vendor lock-in**: Dùng một gateway thương mại có sẵn (managed service) có thể giới hạn khả năng tùy biến, vì bạn buộc phải sống theo các ràng buộc thiết kế của nhà cung cấp đó — khiến việc di chuyển sang giải pháp khác sau này vừa phức tạp vừa tốn kém.

### Phân biệt API Gateway và Service Mesh — dễ nhầm nhưng khác bản chất

Đây là điểm nhiều người mới học hay lẫn lộn. API Gateway quản lý traffic đi VÀO hệ thống từ bên ngoài (north-south traffic) — nó đứng ở rìa hệ thống, lo routing, xác thực, rate limiting, gộp response, dịch giao thức khi request từ ngoài đi vào. Service Mesh thì quản lý giao tiếp NỘI BỘ giữa các service với nhau (east-west traffic) — nó vận hành bên trong lưới microservices, xử lý service discovery, load balancing, observability, mã hóa mTLS, và chính sách retry, thường thông qua các sidecar proxy đi kèm mỗi service.

Hai thứ này bổ trợ nhau chứ không thay thế nhau: gateway lo cửa ngõ và sự trừu tượng hóa cho client, mesh lo hành vi nội bộ giữa các service. Giữ rõ ranh giới này giúp đội ngũ áp dụng đúng pattern vào đúng phần của kiến trúc.

---

## Phần 4: Reverse Proxy — người anh em "mỏng" hơn Gateway

Trước khi đi sâu vào công nghệ cụ thể, cần phân biệt rõ Reverse Proxy và API Gateway — vì hai khái niệm này thường bị dùng lẫn lộn.

Reverse Proxy là một máy chủ trung gian đứng giữa client và backend server, chuyên trách chuyển tiếp request tới đúng tài nguyên backend. Nó hoạt động thay mặt backend, lấy dữ liệu và trả về cho client, tạo ra một lớp trừu tượng cách ly client khỏi chi tiết hạ tầng nội bộ. Các chức năng chính của reverse proxy gồm:

- **Load balancing**: phân bổ đều request tới nhiều backend server, tối ưu tài nguyên và tăng độ tin cậy.
- **Security và bảo vệ**: che chắn backend khỏi bị lộ trực tiếp ra ngoài, giảm rủi ro DDoS, request độc hại, truy cập trái phép.
- **Caching**: lưu bản sao dữ liệu hay được truy cập, giảm thời gian phản hồi và tải lên backend.
- **SSL/TLS termination**: xử lý mã hóa/giải mã, gỡ tải tính toán khỏi backend, tập trung quản lý chứng chỉ.
- **Compression**: nén response trước khi gửi về client, tối ưu băng thông.

Điểm mấu chốt: reverse proxy CHỦ ĐÍCH mỏng — nó chỉ làm việc ở tầng transport, không tiêm logic có ý thức về nghiệp vụ (business-aware logic) như xác thực JWT, thực thi quota, biến đổi payload, hay gộp service. Vì độ mỏng đó, overhead nó tạo ra rất nhỏ, phù hợp khi throughput thô là ưu tiên hàng đầu. Các đại diện quen thuộc: NGINX, HAProxy, Apache HTTP Server — những công cụ này cấu hình thủ công, thường cần khởi động lại service để áp dụng thay đổi, và nếu dùng "y nguyên" thì chưa đủ để đóng vai một API gateway thực thụ.

**Bài học cốt lõi: Reverse proxy giống một "façade" thuần túy — chỉ đảm nhiệm việc mạng; API gateway giống một "nhân viên biên phòng" có thẩm quyền kiểm tra giấy tờ, ra quyết định, và tùy biến hành vi tùy theo loại khách.**

---

## Phần 5: Triển khai Gateway bằng YARP — reverse proxy kiểu .NET, cực kỳ linh hoạt

Vì các reverse proxy truyền thống chưa đủ "thông minh" để làm gateway, .NET đưa ra YARP — Yet Another Reverse Proxy. Khác với các reverse proxy độc lập, YARP không phải một ứng dụng server riêng — nó là một thư viện middleware tích hợp thẳng vào ứng dụng .NET. Điểm mạnh lớn nhất của nó là khả năng tùy biến: bạn có thể chèn logic tùy chỉnh vào bất kỳ giai đoạn nào trong vòng đời request/response, bằng chính C# quen thuộc — không cần học một ngôn ngữ cấu hình riêng biệt.

### Cài đặt

```bash
dotnet new webapi -n Gateway.Yarp
dotnet sln add Gateway.Yarp/Gateway.Yarp.csproj
```

```bash
cd Gateway.Yarp
dotnet add package Yarp.ReverseProxy
```

### Đăng ký middleware trong Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy().LoadFromConfig
 (builder.Configuration.GetSection("ReverseProxy"));
var app = builder.Build();
app.UseAuthentication(); 
app.MapReverseProxy(); 
app.Run();
```

`MapReverseProxy()` thêm pipeline proxy làm gốc của ứng dụng. Cấu hình được nạp từ `appsettings.json`:

```json
{
 "ReverseProxy": {
 "Routes": {
 "appointmentsRoute": {
 "ClusterId": "appointmentsClu",
 "Match": { "Path": "/appointments/{**catch-all}" }
 },
 "documentsRoute": {
 "ClusterId": "documentsClu",
 "Match": { "Path": "/documents/{**catch-all}" }
 },
 "patientsRoute": {
 "ClusterId": "patientsClu",
 "Match": { "Path": "/patients/{**catch-all}" }
 },
 "doctorsRoute": {
 "ClusterId": "doctorsClu",
 "Match": { "Path": "/doctors/{**catch-all}" }
 }
 },
 "Clusters": {
 "appointmentsClu": {
 "Destinations": {
 "d1": { "Address": "https://appointments-svc/" }
 }
 },
 "documentsClu": {
 "Destinations": {
 "d1": { "Address": "https://documents-svc/" }
 }
 },
 "patientsClu": {
 "Destinations": {
 "d1": { "Address": "https://patients-svc/" }
 }
 },
 "doctorsClu": {
 "Destinations": {
 "d1": { "Address": "https://doctors-svc/" }
 }
 }
 }
 }
}
```

Cách đọc cấu hình này: mục `Routes` là tập hợp không sắp thứ tự các định nghĩa route, mỗi route ánh xạ một request đến từ client sang một hoặc nhiều instance backend. Mỗi route cần có `RouteId` (định danh duy nhất) và `ClusterId` (liên kết tới một entry trong mục `Clusters`). Route còn cần một `Match` — có thể là danh sách hostname hoặc một path pattern viết theo cú pháp route template quen thuộc của ASP.NET Core. Khi proxy nhận request, nó đánh giá các match này theo độ cụ thể — template cụ thể nhất thắng, trừ khi có `Order` được set rõ ràng, lúc đó số nhỏ hơn được ưu tiên trước.

Mục `Clusters` là tập hợp không sắp thứ tự các cluster được đặt tên. Một cluster đại diện cho một "upstream" logic, chứa tập các destination được đặt tên — tức các instance service cụ thể và địa chỉ của chúng, bất kỳ instance nào trong đó đều đủ điều kiện nhận traffic cho các route liên quan. Khi xử lý request, YARP kết hợp luật của route với cấu hình của cluster để chọn ra một destination phù hợp — hoàn tất bước chuyển giao từ rìa hệ thống (edge) tới microservice.

Sau khi chỉnh địa chỉ cluster về đúng địa chỉ launch thật (ví dụ `https://localhost:7206/api`), chạy cả service appointments lẫn gateway YARP, rồi gửi request tới base URL của gateway kèm `/appointments`, bạn sẽ nhận về đúng dữ liệu từ service appointments — nhưng client hoàn toàn không biết địa chỉ thật của service đó.

YARP cho ta quyền kiểm soát route và forward gần như hoàn toàn theo kiểu "native" — nhưng nó chưa đủ để xử lý một số việc như biến đổi request phức tạp hay chuyển tiếp token bảo mật. Đây là lý do ta cần một công nghệ được thiết kế RIÊNG cho vai trò gateway: Ocelot.

---

## Phần 6: Triển khai Gateway bằng Ocelot — full-fledged gateway với cache, JWT, rate limit, circuit breaker

Ocelot là một API gateway mã nguồn mở xây trên nền .NET Core. Nó biến đổi các HTTP request đến và chuyển tiếp chúng tới đúng địa chỉ microservice dựa trên cấu hình định sẵn — cấu hình viết bằng JSON, nơi ta định nghĩa route "upstream" (địa chỉ phơi ra cho client) và "downstream" (địa chỉ thật của microservice phía sau).

### Ocelot có sẵn những gì

- **Cache management**: lưu response từ downstream service trong memory hoặc distributed cache, tránh phải round-trip đầy đủ cho các request giống hệt nhau.
- **Rate limiter**: giới hạn số request mỗi client (theo IP hoặc API key) được thực hiện trong một khoảng thời gian.
- **Tích hợp logging chuẩn .NET Core**: mọi logging provider bạn đang dùng đều chạy được ngay, không cần cấu hình lại.
- **JWT authentication**: xác thực token phát hành bởi identity provider (IdentityServer4, Auth0, Microsoft Entra ID...) trước khi forward request.
- **Retry và circuit breaker (dùng Polly)**: nhắc lại từ Chapter 10 — đây chính là tinh thần resilience đã học, giờ được áp dụng ngay tại lớp gateway.
- **Aggregating**: gộp nhiều lời gọi downstream thành một response — kế thừa trực tiếp tinh thần Aggregator Pattern ở Chapter 5, nhưng lần này nằm ngay trong gateway thay vì một service riêng.
- **Request transformation**: biến đổi cả request đi vào lẫn response đi ra.

### Cài đặt

```bash
dotnet new webapi -n Gateway.Ocelot
dotnet sln add Gateway.Ocelot/Gateway.Ocelot.csproj
```

```bash
dotnet add package Ocelot
dotnet add package Ocelot.Provider.Polly
```

### File cấu hình ocelot.json

```json
{
 "Routes": [
 {
 "Key": "appointmentsRoute",
 "DownstreamPathTemplate": "/api/v1/appointments/{everything}",
 "DownstreamScheme": "https",
 "DownstreamHostAndPorts": [
 {
 "Host": "localhost:7206",
 "Port": 443
 }
 ],
 "UpstreamPathTemplate": "/appointments/{everything}",
 "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
 },
 {
 "Key": "patientsRoute",
 "DownstreamPathTemplate": "/api/v1/patients/{everything}",
 "DownstreamScheme": "https",
 "DownstreamHostAndPorts": [
 {
 "Host": "patients-svc",
 "Port": 443
 }
 ],
 "UpstreamPathTemplate": "/patients/{everything}",
 "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
 },
 {
 "Key": "appointmentSummary",
 "UpstreamPathTemplate": "/portal/appointments/{appointmentId}",
 "UpstreamHttpMethod": [ "GET" ],
 "Aggregator": "AppointmentSummaryAggregator"
 }
 ],
 "Aggregates": [
 {
 "Aggregator": "AppointmentSummaryAggregator",
 "RouteKeys": [
 "appointmentsRoute",
 "patientsRoute",
 "doctorsRoute"
 ]
 }
 ],
 "GlobalConfiguration": {
 "BaseUrl": "https://gateway.microservices-health.com",
 "RequestIdKey": "x-correlation-id",
 "QoSOptions": {
 "ExceptionsAllowedBeforeBreaking": 3,
 "DurationOfBreak": 3000,
 "TimeoutValue": 10000
 }
 }
}
```

Đọc từng phần:

- **Routes**: mục cha, nơi khai báo cấu hình upstream/downstream. `DownstreamPathTemplate` là địa chỉ thật của microservice; `DownstreamScheme` là giao thức dùng để giao tiếp; `DownstreamHostAndPorts` là host/port cụ thể. `UpstreamPathTemplate` là đường dẫn phơi ra cho client — gọi vào đây, Ocelot tự động định tuyến sang `DownstreamPathTemplate` tương ứng (bạn thậm chí có thể đổi tên route, ví dụ endpoint gốc tên "Customers" nhưng client chỉ biết nó qua tên "Patients"). `UpstreamHttpMethod` định nghĩa các method hợp lệ.

- **QoSOptions**: đây chính là chỗ nối lại Polly — thư viện resilience đã học kỹ ở Chapter 9 và 10 — vào routes/aggregate của Ocelot:
  - `ExceptionsAllowedBeforeBreaking: 3` — cho phép tối đa 3 lỗi liên tiếp (network fail, response 5xx, timeout) trước khi circuit breaker "trip" sang trạng thái Open cho destination đó; từ đó các lời gọi tiếp theo bị chặn ngay lập tức, tiết kiệm độ trễ cho client và giảm tải cho backend đang gặp sự cố.
  - `DurationOfBreak: 3000` — circuit ở trạng thái Open trong 3 giây, sau đó Polly chuyển sang Half-Open và cho một request thử "dò đường". Nếu thành công, circuit về Closed (hoạt động bình thường); nếu thất bại, breaker mở lại và đồng hồ reset.
  - `TimeoutValue: 10000` — deadline cứng cho mỗi request; quá 10 giây, Ocelot hủy task HTTP và tính đó là một lỗi, cộng dồn vào `ExceptionsAllowedBeforeBreaking` — nghĩa là circuit sẽ trip nhanh khi service chỉ đang chậm chứ chưa hẳn đã chết hẳn.

### Đăng ký trong Program.cs

```csharp
using Ocelot.DependencyInjection;
using Ocelot.Middleware;
using Ocelot.Provider.Polly;

var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddJsonFile("ocelot.json", optional: false,
 reloadOnChange: true);
builder.Services.AddOcelot(builder.Configuration)
 .AddPolly();

var app = builder.Build();
await app.UseOcelot();
app.UseHttpsRedirection();
app.Run();
```

Sau khi cấu hình xong, chỉ cần gọi base URL gateway kèm `/appointments`, request sẽ được Ocelot chuyển tiếp xuống đúng downstream service — client hoàn toàn không biết địa chỉ thật phía sau.

### Aggregator trong Ocelot — Aggregator Pattern ở Chapter 5 tái xuất, lần này "miễn phí" nhờ hạ tầng

Cấu hình `Aggregates` ở trên đã khai báo một endpoint gộp (`appointmentSummary`) kết hợp 3 route: appointments, patients, doctors. Phần logic gộp được viết trong một class implement `IDefinedAggregator`:

```csharp
using Gateway.Ocelot.Models;
using Newtonsoft.Json;
using Ocelot.Middleware;
using Ocelot.Multiplexer;
using System.Net;
using System.Text;

namespace Gateway.Ocelot.Agggregtors;

public sealed class AppointmentSummaryAggregator : IDefinedAggregator
{
 public async Task<DownstreamResponse>
 Aggregate(List<HttpContext> contexts)
 {
    var appointment = await contexts[0].Items.DownstreamResponse()
     .Content.ReadFromJsonAsync<AppointmentDetailsDto>();
    // Service calls for Doctor and Patient info
    var summary = new
     {
     appointment.AppointmentId,
     appointment.StartTime,
     Doctor = new { doctor.DoctorId, doctor.FirstName,
     doctor.LastName, doctor.Specialty },
     Patient = new { patient.PatientId, patient.FirstName,
     patient.LastName, patient.Gender }
     };
    var json = JsonConvert.SerializeObject(summary);
    var headers = new List<Header> { new("Content-Type",
     ["application/json"]) };
    var httpContent = new StringContent(json, Encoding.UTF8,
     "application/json");
    return new DownstreamResponse(httpContent,
     HttpStatusCode.OK,
     headers,
     "OK");
 }
}
```

Khi client gọi vào URL aggregator, Ocelot tự động biến MỘT request thành NHIỀU lời gọi downstream song song, rồi gộp kết quả lại — client hoàn toàn không biết có sự "phân tách" (fan-out) diễn ra phía sau. Nếu một downstream call thất bại trước khi code aggregator chạy tới, Ocelot tự tiêm vào một response lỗi với status code tương ứng, và chính đoạn code aggregator sẽ quyết định: trả về dữ liệu một phần (partial data), một giá trị fallback, hay lan truyền lỗi đó lên trên.

Đây chính là điểm khác biệt lớn so với việc tự viết Aggregator Service ở Chapter 5: logic gộp giờ chạy NGAY trong gateway, không cần một service riêng biệt để bảo trì, và tận dụng luôn cơ chế QoS/circuit breaker đã cấu hình sẵn ở tầng route.

**Bài học cốt lõi: Ocelot không thay thế tư duy Aggregator Pattern — nó chỉ chuyển nơi bạn đặt logic đó, từ một service độc lập sang ngay lớp hạ tầng gateway, giúp bạn tận dụng luôn resilience và routing đã có sẵn.**

---

## Phần 7: Backend for Frontend (BFF) — khi "một API cho tất cả" không còn đủ tốt

Khi hệ thống microservices phục vụ nhiều loại client khác nhau — mobile app, web app, tích hợp bên thứ ba — một API tổng quát chung cho tất cả dần trở nên bất lực trong việc đáp ứng đúng nhu cầu riêng của từng loại client. Đây chính là lúc pattern Backend for Frontend (BFF) trở thành chiến lược thiết kế mấu chốt: thay vì ép mọi client theo cùng một hợp đồng, mỗi loại client tương tác với một BFF service riêng, được thiết kế xoay quanh đúng nhu cầu của nó.

### Những vấn đề BFF giải quyết

- **Over-fetching và under-fetching**: mỗi BFF chỉ trả về đúng dữ liệu frontend đó cần, giảm băng thông và độ phức tạp khi parse dữ liệu (mobile không cần tải những field chỉ web mới dùng, và ngược lại).
- **UI-specific aggregation**: BFF tự điều phối gọi nhiều microservice và định dạng dữ liệu theo đúng cấu trúc tối ưu cho client đó.
- **Authentication complexity**: quản lý session và token tập trung ngay trong BFF, giúp frontend nhẹ đi.
- **Frontend-driven business logic leakage**: những logic chỉ để phục vụ UI được đóng gói trong BFF, không làm "bẩn" microservice hay client app.
- **Versioning và change management**: mỗi BFF tiến hóa độc lập theo nhịp riêng của frontend nó phục vụ, giảm thiểu ảnh hưởng chéo giữa các client không liên quan.

Nói cách khác, BFF cho phép ta cung cấp một API "đo ni đóng giày" theo từng loại thiết bị, dựa trên chính trải nghiệm người dùng mà ta muốn đạt được trên UI đó.

### Analogy: Trợ lý riêng cho từng phòng ban, thay vì một lễ tân chung cho cả tòa nhà

Ở Phần 1 mình đã ví API Gateway như quầy lễ tân chung của cả tòa nhà. Giờ hãy hình dung, phòng kế toán cần một trợ lý biết rành về số liệu tài chính, trong khi phòng pháp lý cần một trợ lý rành về hợp đồng — nếu chỉ có MỘT lễ tân chung phải trả lời mọi câu hỏi chuyên sâu của mọi phòng ban, người đó sẽ quá tải và trả lời hời hợt. BFF giống như việc mỗi phòng ban có một trợ lý riêng, hiểu rõ đúng những gì phòng đó cần, dù tất cả trợ lý này vẫn cùng làm việc với chung một tập hồ sơ gốc (các microservice) phía sau.

### Triển khai BFF bằng nhiều instance Ocelot qua Docker

Cách thực dụng nhất để triển khai BFF là dùng nhiều instance Ocelot, mỗi instance mang một file cấu hình riêng — Docker là ứng viên lý tưởng để host việc này, cho phép ta dùng chung một base image gateway nhưng sinh ra các instance khác nhau tùy theo BFF cần.

**Dockerfile cho Gateway.Ocelot:**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Gateway.Ocelot/Gateway.Ocelot.csproj", "Gateway.Ocelot/"]
RUN dotnet restore "./Gateway.Ocelot/Gateway.Ocelot.csproj"
COPY . .
WORKDIR "/src/Gateway.Ocelot"
RUN dotnet build "./Gateway.Ocelot.csproj" -c $BUILD_CONFIGURATION
 -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Gateway.Ocelot.csproj" -c $BUILD_CONFIGURATION
 -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Gateway.Ocelot.dll"]
```

Đây là build kiểu multi-stage (4 giai đoạn) để tạo ra một image Ocelot gateway gọn nhẹ, chạy được trên .NET 9 — build project web rồi publish file build vào thư mục app mà container sẽ dùng.

**docker-compose.yml — spin up hai BFF, một cho web, một cho mobile:**

```yaml
version: "3.8"
services:
 web-bff:
 image: healthcare/ocelotgw:latest
 build:
 context: ./Chapter 11/Gateway.Ocelot
 container_name: web-bff
 environment:
 - ASPNETCORE_ENVIRONMENT=Production # or Development
 ports:
 - "8080:80" # expose Web BFF
 volumes:
 - ./Chapter 11/Gateway.Ocelot/ocelot.WebPortal.json:/app/ocelot.json
 mobile-bff:
 image: healthcare/ocelotgw:latest
 build:
 context: ./Chapter 11/Gateway.Ocelot
 container_name: mobile-bff
 environment:
 - ASPNETCORE_ENVIRONMENT=Production
 ports:
 - "8081:80" # expose Mobile BFF
 volumes:
 - ./Chapter 11/Gateway.Ocelot/ocelot.MobilePortal.json:/app/ocelot.json
 appointments-svc:
 image: appointments-api:latest
 # … your existing service definition
 patients-svc:
 image: patients-api:latest
 # …
 doctors-svc:
 image: doctors-api:latest
 # …
```

Chú ý cách hai service `web-bff` và `mobile-bff` dùng CHUNG một image (`healthcare/ocelotgw:latest`), nhưng mount vào hai file cấu hình khác nhau (`ocelot.WebPortal.json` và `ocelot.MobilePortal.json`) thông qua volume. Đây chính là kỹ thuật cốt lõi: một base image, nhiều "bản sắc" cấu hình khác nhau — mỗi bản sắc có thể tùy chỉnh riêng về aggregator, rate limit, hay tập route được phép truy cập.

---

## Phần 8: Bảo mật và Rate Limiting tại Gateway — điểm chặn tập trung cho toàn hệ thống

API security không phải một tính năng tùy chọn — nó là một yêu cầu thiết kế bắt buộc. Trong kiến trúc microservices, các API chính là mô liên kết (connective tissue) của toàn hệ thống, và bất kỳ lỗ hổng nào trong đó cũng có thể trở thành điểm yếu tác động lớn. API vừa đóng vai trò lớp truy cập, vừa là ranh giới tin cậy (trust boundary) giữa các hệ thống — vì vậy bảo mật API không phải là lựa chọn, mà là điều kiện tiên quyết.

Hai cơ chế bảo mật hiệu quả nhất là **API key** và **JWT-based authentication**, có thể dùng kết hợp tùy nhu cầu. API key là token đơn giản, thường truyền qua header hoặc query string, phù hợp cho giao tiếp server-to-server hoặc ứng dụng đơn giản. JWT là token nhỏ gọn, tự chứa (self-contained), đại diện cho danh tính người dùng và các claim, thường dùng trong hệ thống hướng người dùng cuối, tận dụng OAuth 2.0 hoặc OpenID Connect.

Khi có gateway, nó trở thành điểm chặn (chokepoint) lý tưởng để thực thi bảo mật và rate limiting — tập trung hóa các mối quan tâm xuyên suốt này, giải phóng các service phía sau khỏi việc phải tự cài đặt lặp lại.

### Xác thực API key trên YARP

Vì YARP được xây trên nền middleware ASP.NET Core, mọi hành vi routing của nó đều có thể được tăng cường bằng chính pipeline xác thực/ủy quyền sẵn có của ASP.NET Core. Ta có thể chèn một middleware kiểm tra API key ngay trước khi request được forward:

```csharp
// Service registrations
var app = builder.Build();

// Custom middleware to validate API key
app.Use(async (context, next) =>
{
 const string apiKeyHeader = "X-Api-Key";
 var validApiKey = builder.Configuration["ApiKey"]; // load from
 // appsettings or environment
 if (!context.Request.Headers.TryGetValue(apiKeyHeader,
 out var providedKey) ||
 providedKey != validApiKey)
 {
 context.Response.StatusCode = StatusCodes.Status401Unauthorized;
 await context.Response.WriteAsync("Unauthorized: API key is
 missing or invalid.");
 return;
 }
 await next.Invoke();
});
app.MapReverseProxy();
app.Run();
```

Middleware này chặn request TRƯỚC KHI nó được forward tới route của proxy, đảm bảo có một API key hợp lệ mới cho đi tiếp. Cấu hình key tương ứng có thể trông như sau:

```json
{
 "ApiKeys": {
 "appointments": "key-for-appointments",
 "patients": "key-for-patients",
 "doctors": "key-for-doctors"
 },
 ...
}
```

Lưu ý: ví dụ trên chỉ để minh họa, trong thực tế giá trị API key phải được lưu an toàn trong biến môi trường hoặc secrets manager — không bao giờ hardcode như trên trong production.

### JWT authentication trên YARP

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

```csharp
builder.Services.AddAuthentication("Bearer")
 .AddJwtBearer("Bearer", options =>
 {
 options.Authority = "https://auth.healthmanagement.com";
 options.Audience = "api-gateway";
 options.RequireHttpsMetadata = true;
 });
builder.Services.AddAuthorization();
builder.Services.AddReverseProxy(.LoadFromConfig
 (builder.Configuration.GetSection("ReverseProxy"));
```

```csharp
var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapReverseProxy();
app.Run();
```

Với cấu hình này, mọi request tới gateway đều bị xác thực bằng JWT. Token phải nằm trong header `Authorization: Bearer <token>`. Nếu token thiếu, hết hạn, sai định dạng, hoặc thiếu audience/claim yêu cầu, request bị từ chối với 401 hoặc 403 — và downstream service hoàn toàn không bị chạm tới.

### JWT authentication trên Ocelot — khai báo ngay trong ocelot.json

```json
{
 "Routes": [
 {
 "Key": "appointmentsRoute",
 "DownstreamPathTemplate": "/api/v1/appointments/{everything}",
 "DownstreamScheme": "https",
 "DownstreamHostAndPorts": [
 { "Host": "localhost", "Port": 7206 }
 ],
 "UpstreamPathTemplate": "/appointments/{everything}",
 "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
 "AuthenticationOptions": {
 "AuthenticationProviderKey": "Bearer",
 "AllowedScopes": [ "appointments.read", "appointments.write" ]
 }
 }
 ]
}
```

Cấu hình này đảm bảo mọi request tới `/appointments` phải mang một JWT bearer token hợp lệ, chứa ít nhất một trong các scope yêu cầu. Nếu token thiếu, hết hạn, hoặc không hợp lệ, request bị từ chối với 401 Unauthorized hoặc 403 Forbidden trước khi chạm tới downstream service. Program.cs tương ứng:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddJsonFile("ocelot.json", optional: false,
 reloadOnChange: true);
builder.Services.AddOcelot(builder.Configuration)
 .AddPolly();
var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
await app.UseOcelot();
app.UseHttpsRedirection();
app.Run();
```

Mặc định, Ocelot sẽ chuyển tiếp header Authorization xuống downstream service, cho phép microservice tự kiểm tra ủy quyền bổ sung hoặc ghi log hoạt động phục vụ audit nếu cần. Nếu downstream không cần token, hay cần biến đổi/loại bỏ nó, ta có thể cấu hình header transformation ngay trong routing rule của Ocelot.

### Rate Limiting — kiểm soát tần suất, chống lạm dụng và DoS

Rate limiting là kỹ thuật kiểm soát số lượng request một client được phép gửi trong một khoảng thời gian nhất định — cơ chế điều tiết traffic để đảm bảo dùng tài nguyên công bằng, bảo vệ hệ thống khỏi quá tải, và giảm thiểu lạm dụng hay tấn công DoS.

**Bốn thuật toán rate-limiting phổ biến:**

| Thuật toán | Cách hoạt động | Đặc điểm |
|---|---|---|
| Fixed window | Đếm số request trong một khoảng cố định (VD: 1 phút), hết hạn mức thì chặn hết phần còn lại trong khoảng đó | Đơn giản nhưng có thể bị "burst" ngay đầu mỗi window |
| Sliding window | Cửa sổ đếm trượt theo thời gian thay vì reset theo mốc cố định | Cho ước lượng tần suất chính xác hơn, mượt hơn |
| Token bucket | Token được nạp vào "xô" theo tốc độ cố định; hết token thì request bị trì hoãn/từ chối | Chịu được traffic burst tốt hơn fixed window |
| Leaky bucket | Xử lý request theo tốc độ cố định bất kể burst lớn cỡ nào; request dư bị xếp hàng hoặc drop khi buffer đầy | Làm mượt traffic đầu ra |

Khi client vượt hạn mức, server nên trả về **HTTP 429 Too Many Requests**, kèm header **Retry-After** để báo khi nào client được thử lại.

### Rate limiting trên YARP bằng AspNetCoreRateLimit

```bash
dotnet add package AspNetCoreRateLimit
```

Lưu ý quan trọng: `AspNetCoreRateLimit` KHÔNG phải một phần nội tại của YARP — nó là một middleware độc lập, có thể dùng chung với YARP hoặc bất kỳ ứng dụng ASP.NET Core nào khác (YARP thực tế cũng có khả năng rate-limiting riêng, nhưng cấu hình `IpRateLimiting` dưới đây là dành cho thư viện `AspNetCoreRateLimit`).

```json
"IpRateLimiting": {
 "EnableEndpointRateLimiting": true,
 "StackBlockedRequests": false,
 "RealIpHeader": "X-Forwarded-For",
 "GeneralRules": [
 {
 "Endpoint": "*:/appointments/*",
 "Period": "1s",
 "Limit": 10
 },
 …
 ]
}
```

Giải nghĩa từng field:

- `EnableEndpointRateLimiting: true` — bật rate limiting theo TỪNG endpoint riêng biệt, thay vì áp một hạn mức chung cho toàn hệ thống. Nhờ đó, ta có thể siết chặt hơn với các route nhạy cảm/tốn tài nguyên, và nới lỏng hơn với các route ít rủi ro.
- `StackBlockedRequests: false` — các request bị chặn KHÔNG được tính vào bộ đếm hạn mức của window đó, tránh hiệu ứng "khóa dây chuyền" kéo dài thời gian bị chặn. Nếu set `true`, request bị chặn vẫn tiếp tục bị đếm, có thể kéo dài thời gian bị từ chối.
- `RealIpHeader: "X-Forwarded-For"` — chỉ định lấy IP thật của client từ header này, cần thiết khi ứng dụng chạy sau một reverse proxy/load balancer (nếu không, server có thể lấy nhầm IP của chính proxy làm IP client để tính rate limit).
- `GeneralRules` — danh sách luật rate limit theo từng pattern route, mỗi luật định nghĩa một window thời gian và số request tối đa cho phép trong window đó theo IP.

```csharp
builder.Services.AddMemoryCache();
builder.Services.Configure<IpRateLimitOptions>
 (builder.Configuration.GetSection("IpRateLimiting"));
builder.Services.AddInMemoryRateLimiting();
builder.Services.AddSingleton<IRateLimitConfiguration,
 RateLimitConfiguration>();

var app = builder.Build();
app.UseIpRateLimiting();
app.MapReverseProxy();
```

Điểm cần chú ý: `AddMemoryCache()` là bắt buộc — middleware rate limiting cần một nơi lưu và cập nhật metadata về số lượng request mỗi client (theo IP, API key, hay token) đã thực hiện trong từng time window. Không có cache, middleware sẽ không có cách nào "nhớ" client đã gọi bao nhiêu lần, và do đó không thể thực thi hạn mức.

### Rate limiting có sẵn trong Ocelot

```json
{
 "Routes": [
 {
 "Key": "limitedPatientsRoute",
 "DownstreamPathTemplate": "/api/v1/patients/{everything}",
 "DownstreamScheme": "https",
 "DownstreamHostAndPorts": [{ "Host": "patients-svc", "Port": 443 }],
 "UpstreamPathTemplate": "/patients/{everything}",
 "UpstreamHttpMethod": [ "GET" ],
 "RateLimitOptions": {
 "EnableRateLimiting": true,
 "Period": "10s",
 "Limit": 5
 }
 }
 ]
}
```

Route `limitedPatientsRoute` chỉ chấp nhận GET, mọi POST/PUT/DELETE bị gateway từ chối ngay — đây là một hình thức "quản trị API theo method" (method-level API governance). Phần quan trọng nhất là `RateLimitOptions`: cho phép tối đa 5 request mỗi client trong 10 giây; vượt ngưỡng, Ocelot trả về ngay 429 Too Many Requests mà KHÔNG chuyển tiếp request xuống downstream service — tức là service phía sau không hề bị "chạm" tới, được bảo vệ hoàn toàn ở lớp gateway.

---

## Phần 9: Chọn công nghệ gateway — những tiêu chí thực chiến

Khi lựa chọn công nghệ API gateway cho hệ thống thật, cần cân nhắc các yếu tố sau:

1. **Chọn đúng công nghệ gateway**: các lựa chọn phổ biến gồm managed cloud gateway (Amazon API Gateway, Azure API Management), hoặc Kong. Đánh giá dựa trên nhu cầu cụ thể của hệ thống: khả năng mở rộng, mức độ dễ tích hợp, tính năng bảo mật, chi phí, và trải nghiệm dev.
2. **Định nghĩa rõ luật routing và aggregation**: gateway nên orchestrate nhiều service call và gộp kết quả thành một response duy nhất — giảm số round-trip mạng, đơn giản hóa logic phía client.
3. **Tối ưu hiệu năng và khả năng mở rộng**: dùng caching, xử lý bất đồng bộ, và các chiến lược load-balancing để tăng hiệu năng, scale ngang hiệu quả, giảm độ trễ, giữ tính sẵn sàng cao.
4. **Monitoring và observability**: tích hợp đầy đủ monitoring, logging, tracing ngay trong gateway để theo dõi thời gian thực, chẩn đoán sự cố nhanh, và ra quyết định dựa trên số liệu cụ thể.

---

## Tổng kết chương

Chương này đưa ta đi từ một vấn đề rất cụ thể — client phải tự "nhớ mặt" từng microservice — đến giải pháp kinh điển của microservices architecture: một lớp gateway đứng ra làm mặt tiền thống nhất cho toàn hệ thống. Ta đã thấy API gateway không chỉ đơn thuần route traffic, mà còn tập trung hóa hàng loạt mối quan tâm xuyên suốt: xác thực, rate limiting, caching, monitoring, và service discovery — biến client thành thứ đơn giản nhất có thể trong toàn bộ kiến trúc, đúng như vai trò nó nên có.

Ta cũng học được rằng gateway không phải viên đạn bạc: nếu không thiết kế cho khả năng chịu lỗi, nó trở thành single point of failure; nếu để domain logic rò rỉ vào, nó phình to thành "thick gateway" đi ngược tinh thần phi tập trung của microservices; và nếu chọn sai công nghệ, ta có thể dính vendor lock-in. Việc phân biệt rõ API Gateway (quản lý traffic bên ngoài) với Service Mesh (quản lý traffic nội bộ giữa các service) cũng là một ranh giới quan trọng cần giữ vững khi thiết kế hệ thống lớn.

Về mặt công nghệ, ta đã đi qua hai lựa chọn bổ trợ nhau trong hệ sinh thái .NET: YARP — một reverse proxy dạng middleware, linh hoạt, lập trình được bằng C# thuần, phù hợp khi cần kiểm soát routing ở mức thấp; và Ocelot — một API gateway đầy đủ tính năng, tích hợp sẵn cache, JWT authentication, aggregation, rate limiting, và circuit breaker qua Polly, phù hợp khi hệ thống cần nhiều cross-cutting concern được quản lý tập trung ngay tại lớp cấu hình JSON. Ta cũng đã thấy cách Aggregator Pattern học ở Chapter 5 không hề biến mất — nó chỉ "chuyển nhà" vào bên trong gateway, tận dụng luôn hạ tầng resilience đã xây ở Chapter 9 và 10.

Cuối cùng, BFF Pattern mở rộng tư duy gateway thêm một bước: thay vì MỘT gateway chung cho mọi client, mỗi loại client (web, mobile) có một BFF riêng, được "đo ni đóng giày" theo đúng nhu cầu của nó — triển khai thực tế bằng cách chạy nhiều instance Ocelot qua Docker, mỗi instance mang một file cấu hình riêng biệt.

**Bài học cốt lõi: Gateway không phải là một tầng "thêm cho có" — nó là nơi ta chuyển toàn bộ gánh nặng điều phối, bảo mật, và kiểm soát traffic ra khỏi cả client lẫn từng microservice riêng lẻ, miễn là ta giữ được kỷ luật để nó luôn "mỏng" và không biến thành một monolith trá hình ở lớp hạ tầng.**

Giờ khi đã có một lớp cổng thống nhất phục vụ đúng nhu cầu từng loại client, chương tiếp theo sẽ đi sâu vào chính bản thân các frontend đó — làm sao xây dựng nhiều frontend bằng các framework và công nghệ khác nhau sao cho ăn khớp với từng microservice.
