# Chapter 12 — Micro Frontends: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| Frontend / Presentation Layer | Phần ứng dụng người dùng trực tiếp thấy và tương tác, render UI, xử lý input |
| Service-Oriented Architecture (SOA) | Kiến trúc chia ứng dụng thành lớp service riêng biệt giao tiếp qua HTTP/REST/SOAP |
| Macrofrontend | Toàn bộ lớp frontend được xây và deploy như một khối monolith duy nhất |
| Micro Frontend | Module UI nhỏ, tập trung, gắn với một bounded context, do một đội sở hữu, deploy độc lập |
| Shell Application | Ứng dụng gốc cung cấp điều hướng toàn cục, xác thực chung, design token, điều phối load micro-UI |
| Integration Contract | Hợp đồng tích hợp giữa domain UI và shell (JS/WASM entry point, web component tag, Razor component) |
| Module Federation | Kỹ thuật lắp ráp frontend từ các module xây/deploy độc lập, load động tại runtime |
| Blazor WebAssembly (WASM) | Framework .NET chạy client-side trong trình duyệt qua biên dịch C# sang WebAssembly |
| Blazor Server | Chạy tương tác trên server qua SignalR, browser chỉ nhận giao diện tối thiểu |
| Razor Class Library (RCL) | Thư viện chia sẻ component Razor giữa nhiều ứng dụng Blazor/MAUI |
| LazyAssemblyLoader | Service Blazor dùng JS interop để tải assembly qua mạng khi cần (lazy loading) |
| AOT (Ahead-of-Time) Compilation | Biên dịch code C# sang WASM trước runtime, thay vì diễn giải IL trong trình duyệt |
| Tree Shaking (PublishTrimmed) | Loại bỏ IL/metadata không dùng đến khỏi binary cuối cùng |
| Error Boundary | Component Blazor cô lập lỗi để tránh lan ra toàn UI |
| State Management | Cách lưu trữ, cập nhật, chia sẻ, duy trì dữ liệu trạng thái ứng dụng |
| UserSessionState (singleton pattern) | Container state dùng chung, có event OnChange, để nhiều component reactively cập nhật |

---

## 2. So sánh Macrofrontend vs Micro Frontend

| Tiêu chí | Macrofrontend | Micro Frontend |
|---|---|---|
| Phạm vi | Toàn bộ lớp frontend là một khối | Module UI nhỏ, gắn một bounded context |
| Sở hữu | Đội trung tâm | Đội cụ thể theo domain |
| Chu kỳ deploy | Chung, một release train | Độc lập theo nhịp riêng |
| Công nghệ | Đồng nhất toàn ứng dụng | Có thể khác nhau theo từng module |
| Độ phức tạp ban đầu | Dễ xây hơn lúc đầu dự án | Cần hoạch định domain/composition kỹ hơn |

## 3. So sánh các framework frontend .NET

| Framework | Render ở đâu | SEO | Tương tác cao | Offline | Phù hợp nhất |
|---|---|---|---|---|---|
| ASP.NET Core MVC | Server | Tốt | Yếu (cần thêm JS) | Không | Back-office, workflow doanh nghiệp |
| Razor Pages | Server | Tốt | Yếu | Không | Form, CRUD, giao diện quản trị |
| Blazor Server | Server (qua SignalR) | Trung bình | Cao | Không | Real-time nội bộ, mạng ổn định |
| .NET MAUI | Native (client) | Không áp dụng | Cao | Có | Mobile/desktop native |
| Blazor WASM | Client (trình duyệt) | Yếu hơn (SPA) | Cao | Có (PWA) | Micro frontend, full-stack .NET |

## 4. So sánh Blazor WASM vs JavaScript Framework

| Tiêu chí | Blazor WASM | React/Angular/Vue |
|---|---|---|
| Ngôn ngữ | C# thuần | JavaScript/TypeScript |
| Tái sử dụng logic với backend .NET | Cao | Thấp |
| Hệ sinh thái | Đang phát triển | Đồ sộ, trưởng thành |
| Tải ban đầu | Từng chậm hơn (cải thiện với AOT) | Thường nhanh hơn |
| Chi phí tổ chức | Một stack, ít cognitive overhead | Dual-stack, cần 2 bộ kỹ năng |
| Resilience library | Không dùng được Polly (phải tự viết retry/timeout) | Thư viện resilience riêng của JS ecosystem |

## 5. So sánh cách xử lý Resilience trên từng nền tảng frontend

| Kỹ thuật | Blazor Server / MVC / MAUI | Blazor WASM |
|---|---|---|
| Retry | Polly (đăng ký trên HttpClient) | Thủ công: try-catch + Task.Delay |
| Timeout / Circuit breaker | Polly | Thủ công: CancellationTokenSource |
| Caching | IMemoryCache / IDistributedCache (Redis), SQLite (MAUI) | localStorage/sessionStorage, service worker (PWA) |
| Observability | ILogger<T>, Application Insights, OpenTelemetry | Custom telemetry service tiêm qua DI |

---

## Checklist kỹ thuật khi triển khai

### Khi thiết kế kiến trúc micro frontend
- [ ] Đã xác định rõ ranh giới domain/bounded context cho từng micro frontend (map 1-1 với microservice nếu có thể)
- [ ] Đã thiết kế shell application: điều hướng toàn cục, xác thực chung, design token dùng chung
- [ ] Đã định nghĩa integration contract rõ ràng cho từng domain UI (JS/WASM entry point, web component, Razor component)
- [ ] Đã tách shared component (button, dialog, form field) thành package dùng chung (NuGet/NPM)
- [ ] Đã xác nhận mỗi domain UI có thể deploy độc lập theo chu kỳ riêng

### Khi dựng Blazor WASM project
- [ ] Đã tạo project bằng `dotnet new blazorwasm`
- [ ] Đã đăng ký HttpClient theo tên (named client) hoặc typed client, gắn đúng BaseAddress
- [ ] Đã kiểm tra null trước khi render dữ liệu tải bất đồng bộ (tránh null exception, cải thiện UX loading)
- [ ] Model class khớp đúng payload trả về từ API

### Khi tích hợp với API Gateway (nối Chapter 11)
- [ ] HttpClient trong frontend trỏ về base address của GATEWAY, không trỏ trực tiếp từng microservice
- [ ] Đã xác nhận nếu backend đổi địa chỉ, chỉ cần sửa cấu hình gateway, không sửa từng frontend

### Khi tối ưu hiệu năng
- [ ] Đã bật `PublishTrimmed` và `PublishAot` trong .csproj cho production build
- [ ] Đã cân nhắc lazy loading cho các module ít dùng (admin tools, domain phụ) qua RCL + LazyAssemblyLoader
- [ ] Đã thêm assembly cần lazy load vào `BlazorWebAssemblyLazyLoad` trong .csproj
- [ ] Đã đo thời gian tải ban đầu trước/sau khi áp dụng AOT + tree shaking

### Khi xây resilience cho frontend
- [ ] Đã bọc các component rủi ro bằng `<ErrorBoundary>`
- [ ] Blazor WASM: đã tự viết retry logic (try-catch + delay) vì không dùng được Polly
- [ ] Blazor WASM: đã dùng `CancellationTokenSource` để giới hạn timeout mỗi request
- [ ] Đã có chiến lược cache cho dữ liệu ít thay đổi (localStorage/sessionStorage hoặc service worker nếu PWA)
- [ ] Đã tích hợp logging/observability tối thiểu (ILogger, Application Insights, hoặc custom telemetry)

### Khi quản lý state dùng chung
- [ ] Đã xác định rõ state nào là local, shared, hay remote
- [ ] Đã tạo singleton state container (VD: UserSessionState) với event OnChange cho state cần chia sẻ nhiều component
- [ ] Component subscribe OnChange trong OnInitialized() và unsubscribe trong Dispose() (tránh memory leak)
- [ ] Không lưu dữ liệu nhạy cảm/dễ đổi (volatile) trong browser storage mà không mã hóa/có chính sách hết hạn

---

## Anti-patterns / Lưu ý cần tránh

1. **Chia backend thành microservices nhưng giữ frontend là monolith**: chỉ giải quyết được nửa vấn đề coupling, các đội vẫn phải chờ nhau để release UI.
2. **Frontend gọi thẳng URL từng microservice**: khóa chặt UI vào service topology, deploy trở nên mong manh, khó quản lý xác thực/routing/service discovery tập trung.
3. **Dùng Polly trong Blazor WASM**: không hoạt động vì chạy trong trình duyệt — phải tự viết retry/timeout thủ công bằng try-catch/CancellationTokenSource.
4. **Không kiểm tra null trước khi render dữ liệu bất đồng bộ**: dễ gây null exception khi component render trước khi API trả kết quả.
5. **Không cô lập lỗi giữa các component**: một component lỗi có thể kéo sập toàn bộ UI nếu không dùng ErrorBoundary.
6. **Lưu dữ liệu nhạy cảm/dễ đổi vào localStorage không mã hóa**: rủi ro bảo mật, đặc biệt với token xác thực hoặc dữ liệu cá nhân.
7. **Không có shared state container khi nhiều component cần cùng dữ liệu session**: dẫn đến trùng lặp logic lấy dữ liệu và UI không đồng bộ giữa các phần.
8. **Bỏ qua AOT/tree shaking cho Blazor WASM production build**: payload lớn không cần thiết, ảnh hưởng tới thời gian tải trên mạng chậm/thiết bị yếu.

---

## Từ vựng chuyên ngành (Anh - Việt)

| Thuật ngữ | Nghĩa |
|---|---|
| Micro Frontend | Vi giao diện, module UI nhỏ độc lập theo domain |
| Macrofrontend | Giao diện dạng khối lớn, xây/deploy như một đơn vị |
| Shell Application | Ứng dụng vỏ bọc, điều phối các micro-UI |
| Bounded Context | Ngữ cảnh giới hạn, ranh giới domain tự trị (từ DDD) |
| Module Federation | Liên kết module, kỹ thuật load module độc lập tại runtime |
| Ahead-of-Time (AOT) Compilation | Biên dịch trước, biên dịch sẵn thay vì diễn giải lúc chạy |
| Tree Shaking | Rung cây, loại bỏ code không dùng đến |
| Lazy Loading | Tải trễ, chỉ tải khi cần |
| Progressive Web App (PWA) | Ứng dụng web tiến bộ, hoạt động gần giống app native, hỗ trợ offline |
| Error Boundary | Ranh giới lỗi, cô lập lỗi trong một vùng UI |
| State Management | Quản lý trạng thái ứng dụng |
| Singleton Pattern | Mẫu thiết kế đảm bảo chỉ có một instance dùng chung |
| Circuit Breaker | Bộ ngắt mạch, ngăn gọi tiếp service đang lỗi |
| Service Worker | Tiến trình nền của trình duyệt, hỗ trợ cache và hoạt động offline |
| Dependency Injection (DI) | Tiêm phụ thuộc |
| Correlation ID | Mã tương quan, dùng để truy vết request xuyên hệ thống |

---

## Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Micro Frontend giải quyết vấn đề gì mà microservices ở backend không giải quyết được?**
→ Microservices tách backend thành các service độc lập, nhưng nếu frontend vẫn là một project monolith do một đội trung tâm quản lý, các đội vẫn phải phối hợp và chờ nhau để release UI — lặp lại đúng vấn đề coupling mà microservices backend đã giải quyết. Micro Frontend mở rộng tư duy bounded context và team autonomy lên tầng UI, cho phép mỗi đội sở hữu cả service lẫn UI của domain mình, deploy độc lập.

**Q2: Phân biệt Macrofrontend và Micro Frontend?**
→ Macrofrontend là khi toàn bộ lớp UI được xây và deploy như một khối monolith duy nhất (dù có thể tiêu thụ nhiều API khác nhau), thường do một đội trung tâm quản lý theo một chu kỳ release chung. Micro Frontend là một module UI có phạm vi nhỏ hơn nhiều — tập trung vào một domain cụ thể, do một đội sở hữu, phát triển và deploy độc lập theo nhịp riêng.

**Q3: Tại sao Blazor WASM phù hợp cho kiến trúc micro frontend hơn Blazor Server?**
→ Blazor WASM chạy hoàn toàn client-side, không cần duy trì kết nối SignalR liên tục tới server như Blazor Server (vốn tốn tài nguyên server và nhạy cảm với độ trễ mạng khi tải cao). Blazor WASM cũng module hóa tốt hơn — có thể build/deploy độc lập, hỗ trợ lazy loading assembly, và hoạt động offline như PWA — những đặc tính rất phù hợp với việc tách UI thành nhiều module độc lập.

**Q4: Tại sao API Gateway lại quan trọng với Micro Frontend?**
→ Nếu không có gateway, mỗi micro frontend phải tự biết địa chỉ, quy tắc xác thực, và chi tiết giao thức của backend service riêng — phá vỡ mục tiêu decoupling. Gateway đóng vai trò lớp trung gian, cung cấp một điểm truy cập thống nhất, giúp mỗi micro frontend hoạt động độc lập với vị trí vật lý/logic của backend, đồng thời tập trung hóa service discovery, load balancing, và bảo mật.

**Q5: Tại sao không dùng Polly được trong Blazor WASM, và giải pháp thay thế là gì?**
→ Blazor WASM chạy hoàn toàn bên trong trình duyệt (sandbox), không có môi trường .NET runtime đầy đủ phía server để Polly hoạt động như thiết kế ban đầu (vốn nhắm vào HttpClient ở môi trường server/desktop). Giải pháp thay thế: tự viết retry logic bằng try-catch kết hợp Task.Delay, và dùng CancellationTokenSource để enforce timeout thủ công.

**Q6: AOT compilation và Tree Shaking khác nhau ở điểm nào, và vì sao cả hai đều cần cho Blazor WASM?**
→ Tree Shaking (PublishTrimmed) loại bỏ code/metadata KHÔNG dùng đến khỏi binary cuối, giảm kích thước payload. AOT compilation (PublishAot) biên dịch trước code C# thành WASM native thay vì diễn giải IL lúc runtime, giúp giảm thời gian khởi động và tải CPU. Cả hai bổ trợ nhau: Tree Shaking giảm KÍCH THƯỚC tải xuống, AOT giảm THỜI GIAN thực thi sau khi đã tải — cùng nhắm tới trải nghiệm khởi động nhanh hơn cho ứng dụng chạy trong trình duyệt.

**Q7: Làm sao chia sẻ state (VD: thông tin đăng nhập) giữa các component Blazor độc lập?**
→ Vì Blazor component mặc định bị cô lập (không share state trừ khi truyền qua parameter hoặc inject service), giải pháp chuẩn là tạo một singleton state container (VD: UserSessionState) đăng ký qua dependency injection, có event OnChange để các component subscribe. Khi state thay đổi, container gọi NotifyStateChanged(), các component đã subscribe tự động gọi lại StateHasChanged() để re-render.

**Q8: Module Federation là gì và nó liên quan gì đến Lazy Loading trong Blazor?**
→ Module Federation là kỹ thuật lắp ráp frontend từ các module được xây dựng và deploy độc lập, load động tại runtime thay vì bundle chung lúc build — khởi nguồn từ Webpack 5 trong hệ sinh thái JavaScript. Trong Blazor, tinh thần này được hiện thực qua Razor Class Library (RCL) kết hợp LazyAssemblyLoader: mỗi domain UI đóng gói thành một RCL riêng, và shell chỉ tải assembly đó khi người dùng thực sự điều hướng tới route tương ứng.

---

## Bài tập thực hành đề xuất

1. Tạo một project Blazor WASM mới với 2 component độc lập (VD: Appointments, Doctors), mỗi component gọi một API giả lập riêng qua named HttpClient.
2. Refactor project trên để gọi qua một gateway URL chung (dùng lại gateway Ocelot/YARP đã dựng ở Chapter 11) thay vì gọi thẳng từng service.
3. Bật `PublishTrimmed` và `PublishAot` trong .csproj, so sánh kích thước file build và thời gian tải trước/sau khi bật.
4. Tách một component (VD: Appointments) ra một Razor Class Library riêng, cấu hình lazy load qua `LazyAssemblyLoader` và `BlazorWebAssemblyLazyLoad`, xác nhận assembly chỉ tải khi điều hướng tới đúng route.
5. Implement một `UserSessionState` singleton với event `OnChange`, dùng nó để đồng bộ trạng thái đăng nhập/đăng xuất giữa component NavMenu và ít nhất một component khác (VD: trang Profile).

---

*Ghi chú: Chương tiếp theo (Chapter 13) sẽ đi vào Securing Microservices — bảo mật toàn diện cho hệ thống, khép lại Part 3 (Resiliency, Security, and Infrastructure Patterns) đã bắt đầu từ Chapter 10.*
