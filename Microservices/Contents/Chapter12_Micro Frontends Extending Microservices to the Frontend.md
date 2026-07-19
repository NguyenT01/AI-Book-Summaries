# Bài học: Khi mặt tiền cũng cần được chia nhỏ như hậu trường

> Chương 12 — Micro Frontends: Extending Microservices to the Frontend
> Ví dụ xuyên suốt: hệ thống quản lý phòng khám — lần này ta xây dựng chính phần giao diện (Appointments, Doctors, Patients) bằng Blazor WebAssembly, ghép vào một shell chung, và kết nối nó với gateway đã dựng ở Chapter 11.

---

## Phần 1: Vấn đề thực tế — khi một đội duy nhất phải "ôm" cả một giao diện khổng lồ

Ở Chapter 11, ta đã dựng xong một lớp gateway đứng trước toàn bộ backend, giúp client chỉ cần biết một địa chỉ duy nhất. Nhưng có một câu hỏi mình cố tình chưa trả lời: chính cái "client" đó — cái giao diện người dùng cuối cùng nhìn thấy — nó được xây dựng như thế nào? Nếu ta đã dày công tách backend thành hàng chục service độc lập, mỗi service một đội sở hữu, một database riêng, một chu kỳ deploy riêng... mà phần frontend vẫn là MỘT project khổng lồ do MỘT đội trung tâm quản lý, thì chẳng phải ta vừa giải quyết được nửa bài toán thôi sao?

Hãy hình dung tình huống này: đội Appointments muốn release một tính năng mới ngay trong ngày để xử lý một edge case khẩn cấp về lịch hẹn trùng. Nhưng vì toàn bộ giao diện nằm trong một project frontend chung, họ phải chờ đội Billing hoàn tất kiểm thử tính năng hóa đơn mới của mình, chờ đội Patient Records duyệt xong bản dịch đa ngôn ngữ, rồi mới được gộp chung vào một lần release toàn hệ thống. Một thay đổi nhỏ ở một góc UI kéo theo việc build lại, test lại, deploy lại TOÀN BỘ ứng dụng — đúng như những gì monolith từng gây ra ở tầng backend, giờ lặp lại y hệt ở tầng frontend.

Về bản chất, frontend (hay lớp trình bày) là phần ứng dụng người dùng trực tiếp tương tác và nhìn thấy — nó chịu trách nhiệm render giao diện, xử lý input người dùng, và điều phối tương tác giữa người dùng với các backend service. Nền tảng của phát triển frontend hiện đại xây trên ba trụ cột quen thuộc: HTML (bộ khung, định nghĩa cấu trúc nội dung), CSS (tạo màu sắc, layout, kiểu chữ, hiệu ứng), và JavaScript (tạo hành vi động, xử lý DOM, giao tiếp với API) — thường đi kèm TypeScript và các framework như React, Angular, Vue.js.

Trong kiến trúc monolith kinh điển, lớp trình bày được xây trực tiếp trên business logic và data access layer, tất cả trong một project. Điều này khiến UI bị khóa chặt vào backend, thường dùng chung code base và chu kỳ deploy — cập nhật độc lập trở nên khó khăn, và việc áp dụng công nghệ mới gần như bất khả thi vì mọi thay đổi đều ảnh hưởng tới toàn hệ thống.

### Analogy: Tòa soạn báo với một phòng biên tập chung, hay nhiều ban biên tập độc lập?

Hãy hình dung một tòa soạn báo. Nếu mọi bài viết — thể thao, kinh tế, giải trí, chính trị — đều phải qua MỘT phòng biên tập duy nhất, mọi quyết định in ấn phải chờ TOÀN BỘ số báo hoàn thiện mới được xuất bản, thì ban thể thao sẽ không thể ra tin nóng ngay sau trận đấu — họ buộc phải chờ ban chính trị hoàn tất bài phân tích dài của mình. Ngược lại, nếu mỗi ban có quyền tự biên tập, tự xuất bản chuyên mục của mình theo nhịp riêng — miễn là tuân theo cùng một khổ giấy, phông chữ, và logo chung của tờ báo — thì ban thể thao có thể tung tin trong vài phút, còn ban điều tra có thể dành hàng tuần cho một bài phân tích sâu. Đó chính là tinh thần micro frontend: mỗi "chuyên mục" (bounded context) có quyền tự chủ hoàn toàn về nhịp độ xuất bản, miễn là vẫn khớp vào một "tờ báo" (shell) có diện mạo thống nhất.

---

## Phần 2: Service-Oriented Architecture — nền móng để tách frontend ra khỏi backend

Trước khi có micro frontend, bước đệm quan trọng là Service-Oriented Architecture (SOA) — kiến trúc chia ứng dụng thành một lớp service riêng biệt, nơi các chức năng nghiệp vụ cụ thể được triển khai, thường qua giao thức HTTP/REST hoặc SOAP. Điều này cho phép chu kỳ phát triển module hóa và lỏng lẻo hơn (loosely coupled), cho phép dùng các công nghệ/framework khác nhau để đáp ứng nhu cầu cả frontend lẫn backend.

Nhờ SOA, frontend có thể phát triển tách biệt, tiêu thụ dữ liệu qua API, và scale độc lập theo tải và pattern sử dụng của riêng nó — một frontend có thể tiêu thụ MỘT hoặc NHIỀU web service khác nhau.

Một điểm cần lưu ý: frontend không nhất thiết phải là web. Khi nghe "frontend", ta thường nghĩ ngay tới ứng dụng web — nhưng thực chất frontend là BẤT KỲ giao diện nào cho phép người dùng tương tác với hệ thống phần mềm: website, mobile app, desktop app, thậm chí trợ lý giọng nói hay command-line interface. Trong một kiến trúc service-oriented phân tán, các frontend client có thể tồn tại trên nhiều nền tảng khác nhau, cùng giao tiếp với chung một tập backend service qua API.

### Macrofrontend vs Micro frontend

Đây là một cặp khái niệm quan trọng cần phân biệt rạch ròi:

| Tiêu chí | Macrofrontend | Micro frontend |
|---|---|---|
| Phạm vi | Toàn bộ lớp frontend, coi như MỘT khối gắn kết | Một module UI nhỏ, tập trung, ứng với một bounded context |
| Ai sở hữu | Thường một đội trung tâm | Một đội cụ thể, gắn liền với domain của họ |
| Chu kỳ deploy | Một chu kỳ release chung cho toàn ứng dụng | Deploy độc lập theo nhịp riêng của từng module |
| Có thể tiêu thụ nhiều API không | Có thể, nhưng vẫn nằm trong MỘT ứng dụng/chu kỳ release | Có, nhưng mỗi module tự quản lý API của domain mình |

Nói ngắn gọn: macrofrontend là những gì ta có khi UI được xây và deploy như một monolith — bất kể backend module hóa tốt tới đâu. Micro frontend đại diện cho một phạm vi nhỏ hơn nhiều: một module UI tập trung, do một đội cụ thể sở hữu, phát triển độc lập, và deploy theo nhịp riêng.

**Bài học cốt lõi: Việc backend đã được chia thành microservices không tự động khiến frontend "microservices hóa" theo — nếu frontend vẫn là một khối monolith, ta chỉ mới giải quyết được một nửa vấn đề coupling.**

---

## Phần 3: Micro Frontend và DDD — cùng một tư duy, áp dụng lên cả hai đầu stack

Nhớ lại từ Chapter 2, Domain-Driven Design khuyến khích ta chia hệ thống doanh nghiệp thành các bounded context — những phân vùng con tự trị, sở hữu model, ngôn ngữ, và vòng đời riêng của chính nó. Khi xây backend, một microservice thường phục vụ đúng MỘT bounded context. Micro frontend ở tầng UI mang trách nhiệm y hệt: nó trình bày phần giao diện của đúng bounded context đó, và tiến hóa theo đúng nhịp của service mà nó đại diện.

Khi mỗi đội cross-functional sở hữu CẢ microservice LẪN micro frontend của mình, ta đạt được:

- **Tính tự chủ của đội (team autonomy)**: mỗi đội ship UI, API, và migration database độc lập, không cần chờ một "đoàn tàu release" chung của toàn frontend.
- **Bám sát nghiệp vụ (business alignment)**: tính năng thay đổi theo đúng nhu cầu kinh doanh; code và đội ngũ được tổ chức theo năng lực (capability) chứ không theo tầng kỹ thuật (layer).
- **Tự do công nghệ (technology freedom)**: mỗi đội chọn framework phù hợp nhất cho bounded context của mình (Blazor, React, Angular...), trong khi shell vẫn tích hợp chúng liền mạch.
- **Deploy-for-value**: các bản deploy nhỏ gọn hơn, bug fix và tính năng mới có thể tung ra hoặc rollback nhanh mà không ảnh hưởng toàn ứng dụng.

Ưu điểm đầu tiên và rõ ràng nhất của micro frontend là nó giúp ta phân vùng ứng dụng đúng theo yêu cầu DDD — chia một domain lớn thành các bounded context. Nhưng phân vùng chỉ là một nửa câu chuyện; phần còn lại là composition — cách ghép các UI này lại một cách có chủ đích, để trải nghiệm người dùng vẫn bám sát mục tiêu domain, đồng thời backend service vẫn giữ được sự tách biệt khỏi công nghệ trình bày.

### Mô hình shell + domain UI

Nếu hình dung một UI dùng chung được thiết kế theo tinh thần này, ta có ba lớp:

1. **Shell application**: cung cấp điều hướng toàn cục tới các phần của ứng dụng (Patients, Appointments, Billing), xử lý luồng xác thực chung, và các design token chuẩn (màu sắc, typography, kiểu form). Shell chịu trách nhiệm đăng nhập/đăng xuất, áp dụng styling tổng thể, và điều phối micro-UI nào cần load khi người dùng điều hướng tới một domain cụ thể. Bên dưới, shell có thể dùng server-side composition (VD: Razor Pages nhúng Blazor component) hoặc client-side composition (Blazor WebAssembly hoặc router kiểu React dynamic import các domain UI).

2. **Domain UI riêng biệt**: mỗi domain có UI riêng, sống trong repository/package của chính nó, phơi ra một "hợp đồng tích hợp" (integration contract) rõ ràng — có thể là entry point JavaScript/WebAssembly, một custom web component tag, hoặc một Razor component render phía server. Bên trong mỗi domain UI, đội sở hữu tự implement form, bảng dữ liệu, và luồng tương tác tiêu thụ API của service mình. Ví dụ: UI đặt lịch hẹn dùng đơn giản HTTP GET/POST (`HttpClient.GetFromJsonAsync<Appointment[]>()`), trong khi UI quản lý tài liệu có thể cần gRPC-Web stream để tải hiệu quả các file scan PDF lớn.

3. **Ánh xạ UI module ↔ bounded context** làm rõ trách nhiệm sở hữu: đội Patient Records có thể tự quyết định cách phân trang danh sách hồ sơ, cache ảnh để xem offline, hay thêm lớp phủ audit-trail — mà không cần phối hợp với chu kỳ release UI của đội Billing. Các mối quan tâm dùng chung (xử lý lỗi toàn cục, token xác thực chuẩn, đo lường telemetry) được cung cấp bởi shell hoặc một thư viện UI dùng chung, đảm bảo tính nhất quán xuyên suốt mà không phá vỡ tính tự chủ của từng đội.

Về mặt deploy, mỗi domain UI có thể theo chu kỳ release riêng — UI Appointment có thể deploy cập nhật hàng giờ để xử lý các xung đột edge-case, trong khi UI Billing cần release hàng tuần theo yêu cầu kiểm duyệt quy định. Dùng module federation hoặc dynamic import, shell tự động lấy phiên bản mới nhất của mỗi domain UI ngay tại runtime, đảm bảo người dùng luôn thấy chức năng mới nhất mà không cần redeploy toàn site.

Cuối cùng, ẩn dưới cấu trúc module hóa này là các thành phần dùng chung (button, dialog, form field) và một hệ thống CSS thống nhất (Bootstrap, token dựa trên Tailwind, hay một thư viện component) giúp thống nhất diện mạo giữa các UI khác nhau — có thể đóng gói thành một NuGet package riêng cho Blazor, hoặc NPM package cho các framework JavaScript.

---

## Phần 4: Micro Frontend ngoài đời thực — Spotify, IKEA, DAZN

Đây không phải một pattern hiếm gặp trên giấy — nhiều tổ chức lớn đã áp dụng thành công:

- **Spotify** thường được nhắc tới như người tiên phong: mỗi tính năng trong web app (playlist, tìm kiếm, hồ sơ người dùng) do một đội riêng quản lý, phát triển như một module frontend tự chứa, được load động và tích hợp vào shell chính. Cách này giúp tăng tốc độ ra tính năng nhờ các đội tự chủ deploy độc lập, thử nghiệm framework khác nhau trong cùng một ứng dụng, và scale cả về phát triển frontend lẫn quyền sở hữu của đội ngũ.

- **IKEA** chuyển đổi từ frontend monolith sang micro frontend để các đội phân tán có thể quản lý các phần khác nhau của nền tảng thương mại điện tử và nội dung. Ban đầu họ dùng iframe, sau đó chuyển sang cách tiếp cận tích hợp hơn — dùng một SPA duy nhất kết hợp module federation.

- **DAZN** áp dụng micro frontend để đẩy nhanh việc phân phối tính năng sản phẩm trên nhiều thị trường quốc tế — họ xây một framework micro frontend nội bộ riêng để xử lý vòng đời, giao tiếp, và tích hợp giữa các thành phần frontend.

---

## Phần 5: Chọn công nghệ frontend trong hệ sinh thái .NET

Việc thiết kế và phát triển micro frontend đòi hỏi hoạch định kỹ lưỡng ranh giới domain, chiến lược ghép UI, giao tiếp liên module, style dùng chung, và mô hình deploy. Trong .NET, có nhiều lựa chọn công nghệ frontend, mỗi lựa chọn có điểm mạnh và đánh đổi riêng.

| Công nghệ | Điểm mạnh | Hạn chế | Phù hợp nhất khi |
|---|---|---|---|
| ASP.NET Core MVC | Render phía server mạnh, tốt cho SEO, accessibility, điều hướng theo trang truyền thống | Khó xử lý partial update, real-time, UI tương tác cao — dễ phải "vá" thêm JavaScript | Ứng dụng enterprise cần back-office workflow, view có thể in, ít tương tác |
| Razor Pages | Gộp controller + view logic vào một page model (.cshtml.cs), gọn nhẹ | Vẫn ràng buộc server-side, không tối ưu cho tương tác client cao hay offline | Form, CRUD, giao diện quản trị nhỏ gọn |
| Blazor Server | Chạy tương tác qua SignalR, truy cập tài nguyên server đầy đủ, tải trang ban đầu nhanh | Nhạy cảm với độ trễ mạng, mỗi session giữ kết nối liên tục → tốn tài nguyên server, khó scale ở tải cao | Ứng dụng real-time, nội bộ, mạng ổn định |
| .NET MAUI | Native UI cho iOS/Android/macOS/Windows, hiệu năng native | Không hướng tới web, mục đích sử dụng khác hẳn | Ứng dụng mobile/desktop cần tích hợp thiết bị sâu |
| Blazor WASM | Chạy client-side hoàn toàn bằng C# biên dịch ra WebAssembly, chia sẻ DTO/service/validation với backend, hỗ trợ module hóa tốt, hoạt động offline như PWA | Payload tải ban đầu từng lớn hơn JS (đã cải thiện nhiều), hệ sinh thái component còn ít hơn JS, một số API trình duyệt cần JS interop | Micro frontend độc lập, team .NET full-stack, cần chia sẻ logic C# giữa client/server |

Blazor WebAssembly (WASM) chiếm một vị trí đặc biệt: nó mang lại khả năng thực thi phía client cho dev .NET bằng cách biên dịch C# sang WebAssembly, cho phép xây dựng SPA tương tác đầy đủ mà không cần JavaScript. Nó cho phép chia sẻ DTO, service, logic validation, và xác thực giữa cả backend lẫn frontend — dev được hưởng cùng bộ công cụ, trải nghiệm debug, và mô hình dependency injection họ đã quen dùng ở server. Các ứng dụng Blazor WASM về bản chất module hóa tốt, hỗ trợ dễ dàng cho kiến trúc micro frontend — có thể build và deploy độc lập, tích hợp API REST hoặc gRPC, thậm chí hoạt động offline như Progressive Web App (PWA).

Dĩ nhiên có đánh đổi: kích thước tải ban đầu của Blazor WASM từng lớn hơn các lựa chọn JavaScript, dù các phiên bản gần đây đã cải thiện đáng kể. Nó cũng chưa có hệ sinh thái component/plugin đồ sộ như JavaScript framework, dù đang cải thiện dần khi cộng đồng và các vendor đầu tư nhiều hơn. Ngoài ra, vì runtime chạy bên trong trình duyệt, một số API và khả năng bị giới hạn hoặc phải truy cập qua JavaScript interop.

**Bài học cốt lõi: Không có công nghệ frontend nào là "đúng" tuyệt đối — lựa chọn phụ thuộc vào mức độ tương tác cần thiết, mạng có ổn định không, và quan trọng nhất là đội ngũ của bạn mạnh ở kỹ năng nào.**

---

## Phần 6: Blazor WASM vs JavaScript Framework — chọn theo team, không chỉ theo công nghệ

Khi quyết định giữa Blazor WebAssembly và các framework JavaScript (React, Angular, Vue), lựa chọn thường phụ thuộc vào sự phù hợp hệ sinh thái, kỹ năng đội ngũ, và ưu tiên kiến trúc — không chỉ thuần túy kỹ thuật.

Blazor WASM nổi bật với dev .NET muốn có trải nghiệm full-stack liền mạch chỉ dùng C#. Nó cho phép tái sử dụng model, logic validation, và hợp đồng service giữa client và server mà không cần chuyển ngữ cảnh sang JavaScript/TypeScript — đặc biệt hữu ích với tổ chức đã đầu tư sâu vào ASP.NET Core, vì Blazor tích hợp mượt mà với API, middleware, và identity provider hiện có.

Ngược lại, JavaScript framework sở hữu hệ sinh thái đã được thiết lập vững chắc và đồ sộ hơn nhiều. React và Angular có cộng đồng lớn, công cụ mạnh, vô số thư viện và component bên thứ ba sẵn có. Chúng thường có thời gian tải ban đầu nhanh hơn và khả năng tương thích tốt hơn với các API trình duyệt cấp thấp — là lựa chọn mạnh cho dự án cần tối ưu hiệu năng, truy cập tính năng client-side nâng cao, hoặc dev đã quen với stack JavaScript/TypeScript.

Tuy nhiên, đánh đổi không chỉ nằm ở mặt kỹ thuật mà còn ở mặt tổ chức. Dùng JavaScript framework trong một hệ sinh thái .NET thường tạo ra độ phức tạp "dual-stack" — đội phải duy trì hai bộ kỹ năng riêng biệt, xây dựng pipeline riêng, và quản lý luồng deploy riêng, làm tăng gánh nặng nhận thức lẫn vận hành. Blazor WASM giúp tích hợp chặt chẽ hơn và dùng chung công cụ, giảm bớt rời rạc giữa đội backend và frontend.

| Tiêu chí | Blazor WASM | JavaScript Framework (React/Angular/Vue) |
|---|---|---|
| Ngôn ngữ | C# thuần | JavaScript/TypeScript |
| Tái sử dụng code với backend .NET | Cao (DTO, validation, service contract dùng chung) | Thấp, cần lớp chuyển đổi/serialize riêng |
| Hệ sinh thái component | Đang phát triển, còn hạn chế hơn | Rất đồ sộ, trưởng thành |
| Thời gian tải ban đầu | Từng chậm hơn, đang cải thiện (AOT) | Thường nhanh hơn |
| Chi phí tổ chức | Một stack duy nhất, giảm cognitive overhead | Dual-stack, cần duy trì 2 bộ kỹ năng |
| Phù hợp nhất | Enterprise system, nội bộ, full-stack .NET (gRPC, SignalR, auth dùng chung) | Sản phẩm hướng người dùng cuối, cần đổi mới UI nhanh, đa dạng thiết bị |

---

## Phần 7: Triển khai thực tế — Blazor WebAssembly Micro Frontend

Bây giờ ta sẽ xây một micro frontend Blazor WASM bao gồm ba bounded context liên quan trong hệ thống quản lý phòng khám: Appointments, Doctors, Patients.

### Khởi tạo project

```bash
dotnet new blazorwasm -o HealthPortal.Frontend --no-https
cd HealthPortal.Frontend
```

Lệnh này scaffold một ứng dụng Blazor client-side chạy hoàn toàn trong trình duyệt qua WASM. Các thành phần quan trọng được sinh ra mặc định:

- **/Pages**: chứa các Razor component đóng vai trò route điều hướng được. Mỗi file `.razor` tương ứng một route. `Index.razor` là trang chủ mặc định; template mặc định còn có `Counter.razor` và `Weather.razor` (mẫu demo). Với bài toán của ta, ta thay bằng: `Appointments.razor`, `Doctors.razor`, `Patients.razor`.
- **/Shared**: chứa các thành phần UI dùng chung nhiều trang. `MainLayout.razor` định nghĩa layout chính (header, sidebar, khu vực nội dung); `NavMenu.razor` chứa link điều hướng.
- **/wwwroot**: chứa tài nguyên tĩnh (ảnh, CSS, JavaScript, service worker cho PWA) — tương tự thư mục `public` ở nhiều framework khác. Template mặc định đã tích hợp sẵn Bootstrap.
- **App.razor**: root component, thiết lập routing qua component `Router`, render đúng trang theo URL.
- **Program.cs**: entry point, cấu hình service (như `HttpClient`), dependency injection, khởi tạo component.
- **wwwroot/index.html**: file HTML host, chứa thẻ `<script>` khởi động Blazor runtime.

### Tạo component và model

```bash
dotnet new razorcomponent -n Appointments -o Pages/Appointments
```

```csharp
public class Appointment
{
    public int Id { get; set; }
    public int PatientId { get; set; }
    public int DoctorId { get; set; }
    public DateTime Date { get; set; }
    public string Status { get; set; }
}
```

### Đăng ký HttpClient — nhắc lại từ Chapter 3

`HttpClient` đã được giới thiệu ở Chapter 3 khi bàn về giao tiếp đồng bộ giữa các microservice. Ở đây ta dùng cách đăng ký đơn giản hơn (thay vì typed client như ở Chapter 3):

```bash
dotnet add package Microsoft.Extensions.Http
```

```csharp
var builder = WebAssemblyHostBuilder.CreateDefault(args);

// Service registrations
builder.Services.AddHttpClient("AppointmentsAPI", client =>
{
    client.BaseAddress = new Uri("https://appointments-service-url/");
});
// other HTTP client registrations

await builder.Build().RunAsync();
```

Mỗi lần đăng ký `HttpClient` tạo ra một alias cho một instance client — nhờ đó ta có được instance ngắn hạn, dùng-xong-hủy cho từng service cụ thể. Mỗi component sẽ inject `IHttpClientFactory` — một lớp trừu tượng của thư viện `HttpClient` cho phép tạo instance client khi cần.

### Component Appointments.razor hoàn chỉnh

```razor
@page "/appointments"
@inject IHttpClientFactory ClientFactory
@using HealthPortal.Frontend.Models

<h3>Appointments</h3>

@if (appointments == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <table>
        <thead>
            <tr><th>ID</th><th>Date</th><th>Status</th></tr>
        </thead>
        <tbody>
            @foreach (var appt in appointments)
            {
                <tr>
                    <td>@appt.Id</td>
                    <td>@appt.Date.ToShortDateString()</td>
                    <td>@appt.Status</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private List<Appointment>? appointments;

    protected override async Task OnInitializedAsync()
    {
        var client = ClientFactory.CreateClient("AppointmentsAPI");
        appointments = await
            client.GetFromJsonAsync<List<Appointment>>(string.Empty);
    }
}
```

Đọc từng phần: `@page "/appointments"` khai báo route ánh xạ component này vào đường dẫn `/appointments`. `@inject IHttpClientFactory ClientFactory` tiêm một instance factory vào component — factory pattern này cho phép ứng dụng tạo các instance `HttpClient` đã cấu hình sẵn, thường kèm base URL hoặc header xác thực riêng theo domain. `@using HealthPortal.Frontend.Models` đưa class `Appointment` vào scope để dùng mà không cần namespace đầy đủ.

Razor component cho phép trộn lẫn HTML và C# qua ký hiệu `@` — component của ta hiển thị một bảng được sinh động dựa trên dữ liệu qua vòng lặp `foreach`.

Cuối cùng, khối `@code { }` định nghĩa logic của component. `private List<Appointment>? appointments` khai báo một list nullable để giữ dữ liệu tải bất đồng bộ từ API — nullable vì tại thời điểm khởi tạo component, dữ liệu chưa tồn tại. `OnInitializedAsync()` là lifecycle method của Blazor, tự động được gọi sau khi component khởi tạo nhưng trước khi render — đây chính là nơi lý tưởng để khởi động các thao tác bất đồng bộ như gọi API. `await client.GetFromJsonAsync<List<Appointment>>(string.Empty)` gửi GET request tới base address của API Appointments và deserialize response JSON thành list `Appointment`.

Cách tiếp cận này áp dụng được cho hầu hết framework frontend của .NET, kể cả Razor, MVC, MAUI. Nhưng dù dùng framework nào, cách này vẫn có một nhược điểm chung: ta nói chung muốn giới hạn mức độ frontend "biết" về các service — càng ít chi tiết về cách gọi từng service được phơi ra cho frontend càng tốt. Đây chính là chỗ gateway phát huy tác dụng để giảm lượng code frontend cần có.

---

## Phần 8: Micro Frontend và API Gateway — nối lại mạch từ Chapter 11

Trong kiến trúc frontend truyền thống, một UI tập trung có thể gọi trực tiếp API backend bằng URL tuyệt đối hoặc logic routing riêng theo domain nhúng ngay trong frontend. Nhưng cách này khóa chặt UI vào cấu trúc topology của service, khiến việc deploy trở nên mong manh và làm tăng độ phức tạp khi quản lý xác thực, routing, và service discovery.

Áp dụng kiến trúc micro frontend càng làm rõ vấn đề này: mỗi module frontend gắn với bounded context và microservice riêng của nó, nghĩa là mỗi micro frontend phải giao tiếp ĐỘC LẬP với backend microservice của mình. Không có gateway, mỗi frontend sẽ phải tự hiểu vị trí, quy tắc xác thực, và chi tiết giao thức của backend service riêng — đúng cái bẫy phá vỡ mục tiêu decoupling và linh hoạt mà ta đang cố xây dựng.

Gateway giải quyết vấn đề này bằng cách đóng vai trò lớp trung gian giữa các micro frontend và service phía sau — cung cấp một điểm truy cập gắn kết và nhất quán cho mọi module frontend, đơn giản hóa service discovery, load balancing, dịch giao thức, và thực thi bảo mật. Với Blazor WebAssembly, nơi client chạy trong môi trường sandbox của trình duyệt và phụ thuộc vào `HttpClient` hoặc gRPC-Web để giao tiếp backend, gateway đảm bảo mọi request — bất kể module nào gọi — đều đi qua một kênh an toàn, chuẩn hóa.

### Cách gateway hỗ trợ micro frontend

Giả sử ba micro frontend Appointments, Doctors, Patients đều cần lấy dữ liệu từ API tương ứng. Có gateway, thay vì mỗi frontend gọi thẳng `https://appointments-api.example.com/api/appointments`, mỗi frontend gọi một endpoint thống nhất như `https://gateway.example.com/api/`. Gateway (cấu hình bằng YARP, Ocelot, hoặc reverse proxy khác — đúng những gì ta đã dựng ở Chapter 11) sẽ định tuyến request tới đúng microservice bằng cơ chế service discovery nội bộ. Nhờ đó, mỗi micro frontend hoạt động độc lập với vị trí vật lý hay logic của backend nó gọi tới.

**Trước khi có gateway:**

```csharp
builder.Services.AddHttpClient("AppointmentsAPI", client =>
{
    client.BaseAddress = new Uri("https://appointments-url/");
});
```

**Sau khi refactor để định tuyến qua gateway:**

```csharp
builder.Services.AddHttpClient("Gateway", client =>
{
    client.BaseAddress = new Uri("https://gateway.example.com/api/");
});
```

Thay đổi này trừu tượng hóa hoàn toàn URL microservice khỏi frontend. Nếu vị trí downstream service thay đổi, ta chỉ cần cập nhật routing ở gateway, KHÔNG cần sửa từng frontend tiêu thụ.

Component `Appointments.razor` giờ refactor lại phần `@code{}`:

```csharp
@code {
    private List<Appointment>? appointments;

    protected override async Task OnInitializedAsync()
    {
        var client = ClientFactory.CreateClient("Gateway");
        // This will resolve to:
        // https://gateway.example.com/api/appointments/
        appointments = await
            client.GetFromJsonAsync<List<Appointment>>("/appointments");
    }
}
```

**Bài học cốt lõi: Gateway ở Chapter 11 không chỉ phục vụ backend-to-backend — nó chính là mảnh ghép giúp micro frontend thực sự "micro" đúng nghĩa, vì mỗi module UI không còn phải tự biết địa chỉ thật, quy tắc xác thực, hay giao thức của từng service riêng lẻ.**

Ta có thể kết hợp chiến lược nhiều framework frontend khác nhau tùy theo domain, để tối đa hóa trải nghiệm người dùng và năng suất dev cho từng bounded context: Blazor WASM cho trải nghiệm client-driven phong phú với logic dùng chung; .NET MAUI cho trải nghiệm native/offline trên mobile/desktop; ASP.NET Core MVC/Razor Pages cho server rendering ổn định, thân thiện quy định; Blazor Server cho trải nghiệm real-time hiệu năng cao.

---

## Phần 9: Tối ưu hiệu năng — Ahead-of-Time Compilation và Tree Shaking

Vì Blazor WASM gửi cả một phiên bản rút gọn của .NET runtime cùng application binary xuống trình duyệt, kích thước tải ban đầu có thể lớn hơn đáng kể so với các framework JavaScript như React hay Vue. Payload này bao gồm không chỉ assembly biên dịch của ứng dụng, mà cả .NET runtime, file localization, satellite assembly, và các package tham chiếu. Payload càng lớn, thời gian tải càng lâu, đặc biệt trên mạng chậm hay thiết bị mobile.

Kỹ thuật Ahead-of-Time (AOT) compilation giúp giảm thiểu điều này: thay vì gửi assembly IL và diễn giải (interpret) chúng tại runtime trong trình duyệt, quá trình build compile trực tiếp code C# thành WASM ngay từ trước. Điều này giảm thời gian khởi động và tải CPU trong trình duyệt, đặc biệt với logic tính toán nặng hoặc chạy thường xuyên.

```xml
<PropertyGroup>
    <PublishTrimmed>true</PublishTrimmed>
    <PublishAot>true</PublishAot>
</PropertyGroup>
```

`PublishTrimmed` loại bỏ IL và metadata không dùng đến trong quá trình build — còn gọi là tree shaking, một dạng dead code elimination. Ví dụ: nếu bạn tham chiếu một NuGet package lớn nhưng chỉ dùng một class/method từ nó, trimming đảm bảo chỉ phần code thực sự cần thiết được đưa vào binary cuối cùng.

`PublishAot` kích hoạt toolchain AOT để compile assembly thành binary WASM native. Một đánh đổi tiềm tàng: AOT làm tăng kích thước artifact build trên đĩa và có thể kéo dài thời gian build; đổi lại, nó giảm sử dụng CPU và cải thiện hiệu năng runtime trong trình duyệt.

---

## Phần 10: Lazy Loading Components — Module Federation kiểu .NET

Code splitting cho phép ứng dụng Blazor chia chức năng thành các bundle/assembly riêng biệt, chỉ tải khi cần — rất quan trọng trong kịch bản micro frontend, nơi người dùng không cần mọi tính năng theo domain cùng lúc. Kỹ thuật này lý tưởng cho các tính năng ít dùng (VD: công cụ admin), các module theo domain trong hệ thống micro frontend dạng shell-hosted, hoặc các thư viện/control bên thứ ba có kích thước lớn.

Khái niệm này gần giống module federation — phương pháp lắp ráp frontend từ các module xây dựng và deploy độc lập, lần đầu xuất hiện trong hệ sinh thái JavaScript qua Webpack 5. Nó cho phép các đội tạo ra các module frontend tự chứa, có thể load động vào ứng dụng host tại runtime, không cần bundle chung tại thời điểm build.

Truyền thống, việc deploy thay đổi ở một phần UI đòi hỏi build lại và deploy lại TOÀN BỘ ứng dụng frontend. Với module federation, mỗi UI theo domain có thể phát triển, kiểm thử, và release độc lập — cải thiện đáng kể tính linh hoạt, đặc biệt quan trọng trong các ngành có quy định chặt hoặc thay đổi nhanh.

### Triển khai lazy loading bằng Razor Class Library (RCL)

```bash
dotnet new razorclasslib -n HealthPortal.Contracts
dotnet sln HealthCare.Microservices.sln add HealthPortal.Contracts/HealthPortal.Contracts.csproj
```

RCL (Razor Class Library) là cách chuẩn để chia sẻ component giữa nhiều ứng dụng Blazor, hoặc giữa web app Blazor và ứng dụng hybrid MAUI render component Blazor.

```bash
dotnet add HealthPortal.Frontend/HealthPortal.Frontend.csproj reference HealthPortal.Contracts/HealthPortal.Contracts.csproj
```

Với demo này, ta chuyển `Appointments.razor` từ project frontend sang project RCL vừa tạo (kèm class model `Appointment` tương ứng, và các NuGet package cần thiết để tránh lỗi thiếu tham chiếu).

Sau khi chuyển, chỉnh sửa `App.razor` để thêm logic load động component RCL khi cần điều hướng tới:

```razor
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using System.Reflection
@inject LazyAssemblyLoader LazyAssemblyLoader

<Router AppAssembly="@typeof(App).Assembly"
    AdditionalAssemblies="assemblies" OnNavigateAsync="NavigateAsync">
    @* Shortened for brevity *@
</Router>

@code {
    private List<Assembly> assemblies = new();

    private async Task NavigateAsync(NavigationContext context)
    {
        if (context.Path.StartsWith("appointments"))
        {
            var assemblyName = context.Path.Substring(6);
            var assembly = await LazyAssemblyLoader.LoadAssembliesAsync(
                new string[] { "HealthPortal.Contracts.wasm" });
            if (assembly != null)
            {
                assemblies.AddRange(assembly);
            }
        }
    }
}
```

Ta bắt đầu bằng cách tiêm service `LazyAssemblyLoader` qua dependency injection — Blazor tự động đăng ký một service singleton cho `LazyAssemblyLoader`. Method `LoadAssembliesAsync` dùng JS interop để tải assembly qua network call. Thêm `AdditionalAssemblies="assemblies" OnNavigateAsync="NavigateAsync"` vào thẻ `Router` để chỉ định các assembly bổ sung sẽ được tải dựa trên một danh sách assembly. Danh sách này được điền động trong method `NavigateAsync` — một event được kích hoạt mỗi khi có nỗ lực điều hướng trong ứng dụng.

Cuối cùng, ta kiểm tra request route cụ thể trong `NavigateAsync`, và nếu pattern khớp với request cho component (ở đây là `appointments`), ta thêm assembly vào danh sách. Tên assembly chính là tên project RCL kèm đuôi `.wasm`. Kỹ thuật này hoàn toàn có thể mở rộng: nhiều thành phần khác nhau có thể được load động theo cùng một request route — đảm bảo chỉ những gì thực sự cần thiết mới được tải cho mỗi request.

Để hoàn tất, cần thêm `ItemGroup` vào file `.csproj` của project frontend:

```xml
<ItemGroup>
    <BlazorWebAssemblyLazyLoad Include="HealthPortal.Contracts.wasm" />
    <!-- Add more assemblies as needed -->
</ItemGroup>
```

Component `Router` của Blazor chỉ định những assembly nào Blazor sẽ tìm kiếm component có thể route tới, và xử lý render component cho route hiện tại — method `OnNavigateAsync` của nó được dùng cùng lazy loading để tải đúng assembly cho endpoint người dùng yêu cầu.

---

## Phần 11: Xây dựng Micro Frontend có khả năng chịu lỗi — nối lại tinh thần Resilience từ Chapter 10

Người dùng tương tác với hệ thống phân tán hiếm khi phân biệt lỗi frontend hay backend — bất kỳ sự cố, độ trễ, hay timeout của service nào cũng bị trải nghiệm như một lỗi của TOÀN BỘ hệ thống. Vì vậy, dù bạn xây frontend bằng Blazor WebAssembly, Blazor Server, ASP.NET Core MVC, Razor Pages, hay .NET MAUI, việc triển khai các pattern giao tiếp có khả năng chịu lỗi là điều bắt buộc để đảm bảo trải nghiệm mượt mà, phản hồi nhanh, và đáng tin cậy.

Resilience trong giao tiếp API không chỉ nằm ở tầng transport — nó phải phản ánh cả trong trải nghiệm người dùng. Cách đơn giản nhất mà ta đã áp dụng ở ví dụ Appointments.razor phía trên: dùng `if` để cập nhật nội dung component động dựa trên sự hiện diện của dữ liệu — tránh được lỗi null exception và kiểm soát được những gì người dùng thấy trong lúc chờ dữ liệu (có thể không bao giờ đến do lỗi API).

### Error Boundary — cô lập lỗi từng component

```razor
<ErrorBoundary>
    <ChildContent>
        <Appointments />
    </ChildContent>
    <ErrorContent>
        <p>An error occurred while loading appointments.</p>
    </ErrorContent>
</ErrorBoundary>
```

Cách này ngăn lỗi ở một component lan ra toàn bộ UI.

### Retry policy và timeout handling — Polly không dùng được trong trình duyệt

Nhắc lại từ Chapter 10 — nơi ta đã học Polly là thư viện resilience mặc định của .NET. Blazor Server, MVC, Razor Pages, và MAUI có thể đăng ký `HttpClient` trực tiếp với Polly như đã làm. Nhưng với Blazor WASM — vì nó chạy TRONG trình duyệt — Polly không hoạt động được. Thay vào đó, logic retry cơ bản phải tự implement thủ công bằng try-catch và async delay:

```csharp
int retries = 3;
while (retries <= 3)
{
    try
    {
        var data = await Http.GetFromJsonAsync<T>("endpoint");
        break;
    }
    catch when (retries > 0)
    {
        await Task.Delay(1000);
    }
}
```

### Circuit breaker và timeout — dùng CancellationTokenSource thay Polly

Mọi request nên có một giới hạn thời gian (timeout boundary) để tránh treo vô hạn. Ở Blazor Server hay MVC, một `HttpClient` phía backend có thể thực thi điều này bằng Polly. MAUI hỗ trợ cả Polly lẫn `CancellationTokenSource`, linh hoạt cho mọi kiểu giao tiếp. Blazor WASM dùng cách thủ công với `CancellationTokenSource` vì không dùng được Polly:

```csharp
using var cts = new CancellationTokenSource(5000);
await Http.GetFromJsonAsync<T>("endpoint", cts.Token);
```

### Caching và chiến lược dữ liệu cũ (stale data)

Giảm số lần gọi backend và mang lại trải nghiệm mượt hơn nếu dữ liệu không tải được từ nguồn chính. Có thể dùng caching trong bộ nhớ hoặc lưu bền cho dữ liệu ít thay đổi (hồ sơ người dùng, danh sách tra cứu tĩnh). MAUI có thể cache vào SQLite hoặc file cục bộ; Blazor Server và MVC có thể dùng memory cache hoặc Redis qua `IMemoryCache`/`IDistributedCache`. Blazor WASM có thể dùng `localStorage`/`sessionStorage` của trình duyệt — nếu chạy như PWA, nó còn tận dụng được các tính năng cache tài nguyên và response tích hợp sẵn, cải thiện đáng kể hiệu năng tải và khả năng hoạt động offline:

```javascript
// Example service worker configuration in wwwroot/service-worker.js
self.addEventListener('fetch', event => {
    event.respondWith(
        caches.match(event.request).then(response => {
            return response || fetch(event.request);
        })
    );
});
```

### Logging và frontend observability

Cần quan sát và theo dõi được ứng dụng frontend — giúp phát hiện response chậm, request thất bại, số lần retry, trạng thái circuit breaker. Các công cụ phổ biến: Application Insights, OpenTelemetry, hoặc logging tùy chỉnh (Serilog, NLog). Mọi frontend .NET đều có thể phát log: MVC log qua `ILogger<T>` và hỗ trợ correlation ID (đã học ở Chapter 9 với Saga Pattern), MAUI và Blazor có thể tiêm custom telemetry service.

**Bài học cốt lõi: Giao tiếp microservice có khả năng chịu lỗi là thách thức chung cho MỌI framework frontend .NET — pattern retry, timeout, fallback, caching, và observability giống nhau về tư tưởng, nhưng cách triển khai khác nhau tùy môi trường thực thi (server vs trình duyệt).**

---

## Phần 12: Quản lý State và giao tiếp giữa các Micro Frontend

State là bất kỳ dữ liệu nào phản ánh trạng thái của ứng dụng hay tương tác của người dùng với nó — từ token xác thực, input form, item được chọn trong danh sách, vị trí cuộn, dữ liệu API đã tải, cho tới preference giao diện như theme. Nó cũng có thể là các cấu trúc phức tạp hơn, như giỏ hàng hay các bước trong một workflow. Việc quản lý state — cách nó được lưu trữ, cập nhật, chia sẻ, và duy trì qua các session, component, thậm chí service — được gọi là quản lý state (state management).

Trong kiến trúc micro frontend, state có thể ở nhiều tầng:

- **Local/component-specific**: giá trị input, tab đang chọn.
- **Shared**: có sẵn cho nhiều component.
- **Remote**: lưu trên server hoặc lấy từ API.

Shared state có thể sống ở:

- Một singleton service tiêm qua dependency injection của .NET
- Browser storage (localStorage/sessionStorage) để duy trì qua các module
- Query string để duy trì qua điều hướng
- Backend cache nếu chia sẻ giữa nhiều thiết bị hoặc nhiều người dùng

Một vài lưu ý quan trọng khi chia sẻ state qua các tầng:

- Dữ liệu dễ thay đổi (volatile) hoặc nhạy cảm không nên lưu trong trình duyệt
- Dữ liệu session có thể cần mã hóa hoặc chính sách hết hạn
- State đặc thù UI (VD: vị trí cuộn) nên reset khi điều hướng nhưng giữ nguyên khi component cập nhật

### Chia sẻ user session state trong Blazor WASM

Một ứng dụng phản hồi tốt và thân thiện "nhớ" hành động của người dùng — giữ input form chưa lưu khi người dùng điều hướng đi, giữ filter trên dashboard, hay giữ ngữ cảnh bệnh nhân xuyên suốt các module y tế.

Blazor component mặc định bị cô lập — chúng đóng gói cả UI lẫn logic riêng, không chia sẻ state trừ khi được truyền tường minh qua parameter hoặc tiêm như một service. Rất phổ biến khi nhiều component (header, dashboard, widget hồ sơ, menu) cần truy cập cùng một dữ liệu session. Không có một container state chung, các component sẽ phải trùng lặp logic lấy dữ liệu session, dẫn tới kém hiệu quả — và thay đổi ở một component không tự động lan tới các component khác, gây ra UI không nhất quán.

Vấn đề này đặc biệt phổ biến với xác thực và quản lý session người dùng. Chức năng login/logout thường được đóng gói trong component riêng, nên việc người dùng đăng nhập không tự động thay đổi state của các component khác trong app. Giải pháp: tạo một singleton state container, `UserSessionState`, mà mọi component có thể tiêm và bind reactively.

```csharp
public class UserSessionState
{
    public string? Username { get; private set; }
    public string? Role { get; private set; }
    public bool IsAuthenticated => !string.IsNullOrEmpty(Username);

    public event Action? OnChange;

    public void SetUser(string username, string role)
    {
        Username = username;
        Role = role;
        NotifyStateChanged();
    }

    public void ClearUser()
    {
        Username = null;
        Role = null;
        NotifyStateChanged();
    }

    private void NotifyStateChanged() => OnChange?.Invoke();
}
```

`SetUser()` cập nhật state và thông báo cho các subscriber (cụ thể là các UI component). `ClearUser()` reset session, được gọi khi logout. `OnChange` là event thông báo cho mọi consumer render lại khi state thay đổi.

Đăng ký singleton trong Program.cs:

```csharp
builder.Services.AddSingleton<UserSessionState>();
```

### Component Login.razor (chỉ mang tính minh họa — bảo mật thật sẽ học ở Chapter 13)

```razor
@page "/login"
@inject UserSessionState Session

<h3>Login</h3>
<input @bind="username" placeholder="Username" />
<input @bind="password" placeholder="password" type="password" />
<button @onclick="LogIn">Log In</button>

@code {
    private string username = string.Empty;
    private string role = string.Empty;

    private void LogIn()
    {
        Session.SetUser(username, role);
    }
}
```

### NavMenu.razor phản ứng theo session state

```razor
@inject UserSessionState Session

<div class="top-row ps-3 navbar navbar-dark">
    <div class="container-fluid">
        <a class="navbar-brand" href="">HealthPortal.Frontend</a>
        <div class="d-flex align-items-center">
            @if (Session.IsAuthenticated)
            {
                <span class="text-white me-3">
                    Welcome, @Session.Username (@Session.Role)</span>
                <button class="btn btn-sm btn-outline-light"
                    @onclick="LogOut">Logout</button>
            }
            else
            {
                <span class="text-white me-3">Not logged in</span>
            }
            <button title="Navigation menu" class="navbar-toggler"
                @onclick="ToggleNavMenu">
                <span class="navbar-toggler-icon"></span>
            </button>
        </div>
    </div>
</div>
@* Existing menu items *@

@code {
    private bool collapseNavMenu = true;
    private string? NavMenuCssClass => collapseNavMenu ? "collapse" : null;

    private void ToggleNavMenu()
    {
        collapseNavMenu = !collapseNavMenu;
    }

    protected override void OnInitialized()
    {
        Session.OnChange += StateHasChanged;
    }

    private void LogOut()
    {
        Session.ClearUser();
    }

    public void Dispose()
    {
        Session.OnChange -= StateHasChanged;
    }
}
```

Ta bắt đầu với `@inject UserSessionState Session` để đưa shared state vào component. Sau đó thêm một khối render có điều kiện thông tin người dùng bên cạnh nút toggle navbar. Nút logout cho phép người dùng xóa dữ liệu session. Trong khối `@code`, ta thêm `Session.OnChange += StateHasChanged` vào `OnInitialized()` để đảm bảo component tự cập nhật phản ứng khi session state thay đổi. `Dispose()` gỡ đăng ký event để tránh rò rỉ bộ nhớ (memory leak).

**Bài học cốt lõi: Vì Blazor component mặc định bị cô lập, việc chia sẻ state xuyên suốt các micro frontend đòi hỏi một container tường minh (singleton service với event) — không có nó, mỗi component sẽ tự trùng lặp logic lấy dữ liệu, và UI dễ rơi vào trạng thái không đồng bộ giữa các phần.**

---

## Tổng kết chương

Chương này mở rộng toàn bộ tinh thần microservices — vốn ta đã áp dụng xuyên suốt cho backend từ Chapter 1 đến Chapter 11 — sang chính lớp giao diện người dùng. Ta bắt đầu bằng việc làm rõ vai trò của frontend trong một ứng dụng hiện đại, giải thích trách nhiệm và nền tảng công nghệ web cốt lõi (HTML, CSS, JavaScript), rồi truy vết sự tiến hóa từ cấu trúc UI monolith sang kiến trúc service-oriented module hóa hơn, đỉnh điểm là sự ra đời của khái niệm macrofrontend và micro frontend. Ta cũng thấy được cách Domain-Driven Design tự nhiên bổ trợ cho mô hình micro frontend, bằng việc cung cấp ranh giới rõ ràng quanh chức năng hướng người dùng, và cách mỗi đội có thể hưởng lợi từ việc tự phát triển, sở hữu, và deploy độc lập micro frontend theo domain của mình.

Ta tập trung vào Blazor WebAssembly như lựa chọn chính, đồng thời ghi nhận các lựa chọn thay thế phù hợp trong hệ sinh thái .NET — sau khi so sánh với ASP.NET Core MVC, Razor Pages, Blazor Server, và .NET MAUI, ta cân nhắc lợi ích và giới hạn của từng lựa chọn trong bối cảnh microservices. Sự phù hợp của Blazor WASM được khẳng định qua khả năng mang lại giao diện tương tác phong phú trong trình duyệt mà không cần JavaScript, tích hợp chặt chẽ với backend .NET, và hỗ trợ tải module động cùng deploy độc lập.

Ta đã đi qua từng bước tạo một project frontend Blazor WASM cho ứng dụng quản lý phòng khám — từ cấu trúc project, tạo component, lấy dữ liệu qua `HttpClientFactory`, cho tới tích hợp với backend service qua routing của gateway (nối trực tiếp mạch kiến thức từ Chapter 11). Ta cũng khám phá lý do có API gateway giúp phát triển frontend dễ dàng hơn nhiều, bằng cách trừu tượng hóa endpoint service, đơn giản hóa giao tiếp, và tập trung hóa các mối quan tâm xuyên suốt.

Về hiệu năng, ta đã áp dụng Ahead-of-Time compilation, tree shaking, và lazy loading — cho thấy việc giảm kích thước payload ban đầu và tận dụng tải assembly tại runtime có thể cải thiện đáng kể hiệu năng khởi động và khả năng mở rộng của ứng dụng Blazor WASM. Những chiến lược này được bổ trợ bởi các thảo luận về giao tiếp frontend có khả năng chịu lỗi — retry policy, circuit breaker, caching, và observability — nối lại trực tiếp tinh thần resilience đã xây ở Chapter 10, đảm bảo các module frontend vẫn phản hồi tốt và chịu lỗi ngay cả khi backend gặp sự cố.

Cuối cùng, ta xem xét quản lý state và giao tiếp liên module — yếu tố thiết yếu trong bất kỳ hệ thống micro frontend nào. Sau khi hiểu rõ khái niệm "state", ta khám phá lý do nó quan trọng với trải nghiệm người dùng và cách quản lý nó xuyên suốt các component được load độc lập, minh họa qua việc triển khai shared session state trong Blazor WASM bằng singleton pattern, cho phép các component tự động cập nhật UI khi có sự kiện đăng nhập/đăng xuất.

**Bài học cốt lõi: Micro frontend không phải là việc "chia nhỏ giao diện cho vui" — nó là sự tiếp nối tất yếu của tư duy bounded context và team autonomy đã xây dựng ở tầng backend, và chỉ thực sự phát huy hiệu quả khi đi kèm một gateway đủ mạnh để trừu tượng hóa service topology, cùng một chiến lược resilience và state management rõ ràng để bù đắp cho sự phân mảnh tất yếu của kiến trúc phân tán.**

Giờ khi cả backend lẫn frontend đều đã được tổ chức theo tinh thần microservices, chương tiếp theo sẽ đi sâu vào khía cạnh còn lại chưa được bàn kỹ: triển khai các biện pháp bảo mật cho toàn bộ ứng dụng microservices.
