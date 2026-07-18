# Bài học: Building Resilient Microservices — Khi lỗi không phải là ngoại lệ, mà là điều chắc chắn sẽ xảy ra

> Chương 10 — Building Resilient Microservices
> Ví dụ xuyên suốt: Hệ thống quản lý phòng khám — Patient Service, Appointment Service, Notification Service, Integration Service (kết nối bên thứ ba như bảo hiểm)

---

## Phần 1: Vấn đề thực tế — thất bại là điều chắc chắn, không phải bất thường

Chương trước khám phá Saga pattern, cho phép ta điều phối các business transaction chạy dài xuyên suốt nhiều service mà không thỏa hiệp tính nhất quán. Saga cung cấp một lớp fault tolerance thiết yếu cho business workflow, nhưng chúng chỉ là khởi đầu khi thiết kế hệ thống thực sự resilient.

Ứng dụng hiện đại phải được xây dựng với kỳ vọng rằng **lỗi sẽ xảy ra**. Trong các hệ thống như vậy, lỗi không phải là bất thường mà là một sự chắc chắn mang tính vận hành. Dù là network timeout, một downstream service crash, một trục trặc hạ tầng tạm thời, hay một API bên thứ ba không phản hồi, ứng dụng của bạn phải được thiết kế để xử lý lỗi một cách graceful.

**Resilience (khả năng phục hồi)** đề cập tới khả năng của một hệ thống phục hồi từ lỗi và tiếp tục hoạt động, ngay cả ở trạng thái suy giảm. Trong microservices, resilience vượt xa việc xử lý exception. Nó bao gồm timeout, retry, circuit breaker, bulkhead isolation, failover strategy, monitoring, caching, và các tính năng liên quan khác. Các cơ chế này phối hợp với nhau để đảm bảo lỗi được kiểm soát và đường phục hồi được tự động hóa, định nghĩa rõ ràng.

**Lưu ý quan trọng**: các biện pháp resiliency ta thảo luận ở đây được hỗ trợ ở **tầng ứng dụng**. Dù môi trường cloud (Azure, AWS, Google Cloud) cung cấp nhiều tính năng resiliency ở tầng hạ tầng (auto-retry, connection pool, service mesh, failover), chúng thường **không thay thế** nhu cầu về pattern ở tầng ứng dụng như những gì được thảo luận trong chương này. Ví dụ: Azure Application Gateway hay AWS API Gateway có thể cung cấp retry logic và timeout ở tầng networking, nhưng điều này không bảo vệ bạn khỏi lỗi logic hoặc lỗi tạm thời xảy ra bên trong logic ứng dụng hoặc giữa các service coupling chặt.

### Analogy: Resilience giống hệ miễn dịch, không phải áo giáp

Áo giáp (defense đơn giản) chỉ ngăn một loại tấn công cụ thể. Hệ miễn dịch của cơ thể thì khác — nó không ngăn mọi vi khuẩn xâm nhập, mà xây dựng cơ chế **phát hiện, cô lập, và phục hồi** khi có sự cố xảy ra. Một hệ thống resilient không cố ngăn 100% lỗi xảy ra (điều bất khả thi trong hệ phân tán) — nó xây dựng cơ chế để phát hiện lỗi nhanh (timeout), thử lại thông minh (retry), ngắt kết nối khi cần (circuit breaker), và tiếp tục hoạt động ở chế độ suy giảm thay vì sụp đổ hoàn toàn.

---

## Phần 2: Bốn microservice trong hệ thống phòng khám và điểm chạm lỗi

- **Patient service**: quản lý nhân khẩu học và hồ sơ bệnh nhân.
- **Appointment service**: đặt lịch hẹn với bác sĩ.
- **Notification service**: gửi email và cập nhật SMS.
- **Integration service**: kết nối tới dịch vụ bên thứ ba như API bảo hiểm để xác định điều kiện và ủy quyền.

Mỗi service này thực hiện tập thao tác riêng, và có lúc một thao tác cần gọi tới một web service khác. Nhiều thứ có thể sai, và điều thiết yếu là lập kế hoạch và phát triển với các điểm lỗi tiềm ẩn trong đầu. Service giao tiếp đồng bộ hoặc bất đồng bộ, cả hai phương pháp đều có ưu/nhược điểm và điểm lỗi tiềm ẩn riêng.

---

## Phần 3: Lỗi trong giao tiếp đồng bộ và cách xử lý

### Timeout Failure

Hãy mô phỏng một cuộc gọi giữa appointment service và patient service để lấy chi tiết của bệnh nhân đang đặt lịch. Do tải xử lý cao, request mất nhiều thời gian hơn dự kiến và vượt quá timeout đã cấu hình. Điều này có thể dẫn tới:

- Client nhận lỗi 504 Gateway Timeout.
- Bệnh nhân không thể đặt lịch, gây bực bội.
- Nếu không được xử lý, có thể kích hoạt retry và khuếch đại tải.

Đây là lý do phổ biến gây lỗi trong cuộc gọi service-tới-service, và về lý thuyết, ta có thể giảm thiểu bằng timeout tường minh và hợp lý được cấu hình trong code gọi service. Không có cách trực tiếp để giải quyết timeout, nhưng ta có thể chọn xử lý hậu quả của cuộc gọi service thất bại theo cách graceful.

### Service Unavailability

Một lý do khác có thể gây timeout là service unavailability — do target service đang bảo trì hoặc tạm thời không khả dụng, hoặc do vấn đề nội bộ với target microservice, như database trở nên không thể truy cập trong lúc request. Thêm cache có thể, ở một mức độ nào đó, giúp giảm khả năng database outage thường xuyên và tăng tốc thời gian phản hồi, cũng giảm rủi ro timeout.

Ta có thể implement **circuit breaker** để ngăn các cuộc gọi tiếp theo cho tới khi hệ thống ổn định. Có các package hỗ trợ implementation này, như **Polly** (thư viện .NET cho transient-fault handling và resilience), cung cấp retry và circuit breaker policy. Điều này cho phép ta cấu hình timeout và retry logic để xử lý các vấn đề như vậy. Retry logic trở nên đặc biệt hữu ích cho lỗi tạm thời, vì nó cho phép ta đảm bảo chính xác hơn rằng một lần thử gọi service sẽ thành công ở các lần gọi tiếp theo.

Rủi ro tăng lên khi service bên thứ ba được giới thiệu, vì ta không thể truy cập hoặc thấy lý do tiềm ẩn cho một outage. Tương tự, ta phải implement cơ chế fallback để log một task xác minh thủ công, áp dụng retry policy, và đặt giới hạn timeout thực tế. Ghi lại các thao tác degraded mode để reconciliation sau này cũng sẽ hữu ích.

### Poor Authentication Handling

Đây là một điểm lỗi tiềm ẩn đôi khi bị bỏ qua. Authentication là tuyến phòng thủ đầu tiên trong bất kỳ hệ thống bảo mật nào. Trong kiến trúc microservices, mỗi service hoạt động độc lập, và các service khác nhau có thể cần chiến lược authentication khác nhau. Nếu ta xem đây là một điểm lỗi, ta thấy giao tiếp liên service đôi khi có thể thất bại giữa các service, dẫn tới response code 401 hoặc 403. Điều này thường do calling service thất bại khi đính kèm API key hoặc authentication token cần thiết. Trong các trường hợp khác, có thể token đã đính kèm đã hết hạn, và service cần lấy một token mới rồi retry request ban đầu.

---

## Phần 4: Lỗi trong giao tiếp bất đồng bộ và cách xử lý

Như đã thảo luận trước đó, giao tiếp bất đồng bộ thiết yếu để decouple service, tăng cường khả năng mở rộng, và cải thiện responsiveness, đặc biệt trong hệ thống phải chịu đựng tải xử lý biến đổi hoặc hoạt động qua workflow phức tạp. Chế độ giao tiếp này đưa vào một lớp kịch bản lỗi khác biệt đáng kể so với API đồng bộ.

### Message Loss

Hãy xem xét appointment service gửi message tới notification service sau khi đặt lịch thành công. Có rủi ro rõ ràng về mất message, có thể xảy ra nếu broker fail hoặc cấu hình sai, message không bền vững do dùng non-durable queue, hoặc consumer crash trước khi acknowledge message. Một cách giải quyết là dùng **durable queue** (VD: RabbitMQ với persistence), đảm bảo pattern acknowledgment đúng chuẩn (Complete, Abandon, hoặc Dead Letter), hoặc lưu trữ message quan trọng vào một transactional outbox trước khi enqueue.

### Timeout và Duplicate Processing

Cũng có rủi ro timeout, vì lỗi mạng có thể dẫn tới giao tiếp kém hoặc gián đoạn. Retry là hợp lý, nhưng có thể dẫn tới mất acknowledgment và khiến consumer xử lý lại cùng message nhiều lần. Đảm bảo idempotency trong message handler và thêm deduplication token, như message ID và correlation ID. Bằng cách này, ta có thể lưu trữ ID message đã xử lý trong database hoặc cache. Ta đã thấy điều này hữu ích ra sao khi implement choreography saga (Chapter 9).

### Outbox Pattern — giải quyết Dual Write Problem

Outbox pattern là một phương pháp phổ biến khác để xử lý mất message của hệ thống. Nó đảm bảo rằng thay đổi trạng thái ứng dụng và việc gửi message được xử lý atomic. Thay vì publish message trực tiếp tới message broker trong cùng transaction với database update, outbox pattern lưu message vào một bảng outbox chuyên biệt trong database local như một phần của transaction. Một background worker riêng biệt, thường gọi là outbox processor hoặc dispatcher, định kỳ quét bảng outbox và publish message tới message broker thực sự.

Pattern này giải quyết **dual write problem** — rủi ro ghi vào database và message broker riêng biệt, nơi một thao tác có thể thành công còn thao tác kia thất bại, dẫn tới dữ liệu không nhất quán.

---

## Phần 5: Implementing Fault-Handling Policies trong Synchronous Communication

### Cài đặt Polly

Polly là framework cho phép ta thêm một lớp resilience vào ứng dụng. Nó hoạt động như một lớp giữa hai service, lưu trữ chi tiết của request đã khởi tạo, và giám sát thời gian phản hồi và/hoặc response code. Nó có thể được cấu hình với tham số xác định điều gì cấu thành lỗi, và ta có thể cấu hình thêm loại hành động ta muốn thực hiện — có thể dưới dạng retry hoặc hủy request.

```bash
dotnet add package Polly
dotnet add package Microsoft.Extensions.Http.Polly
```

### Định nghĩa Retry Policy

```csharp
using Polly;
using Polly.Extensions.Http;

builder.Services.AddHttpClient<PatientsApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiEndpoints:PatientsApi"]);
}).AddPolicyHandler(GetRetryPolicy());
```

```csharp
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions.HandleTransientHttpError()
        .OrResult(r => !r.IsSuccessStatusCode)
        .Or<HttpRequestException>()
        .WaitAndRetryAsync(5, retryAttempt =>
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            (exception, timeSpan, context) => {
                // Thêm logic thực thi trước mỗi lần retry, như logging hoặc re-authentication
            });
}
```

Đầu tiên, ta định nghĩa kiểu trả về của method là `IAsyncPolicy<HttpResponseMessage>`, tương ứng với kiểu trả về của cuộc gọi từ HTTP client. Sau đó ta cho phép policy quan sát lỗi HTTP tạm thời, mà framework có thể xác định mặc định, và ta có thể mở rộng logic đó với thêm điều kiện, như quan sát giá trị `IsSuccessStatusCode`, hoặc thậm chí liệu `HttpRequestException` có được trả về không.

Vài tham số này bao trùm các kịch bản tệ nhất tổng quát của một HTTP response. Sau đó ta đặt tham số cho retry — muốn retry tối đa 5 lần nữa, mỗi lần retry được thực hiện ở khoảng thời gian tăng dần bắt đầu 2 giây sau cuộc gọi trước đó. Đây là khái niệm **backoff**. Cuối cùng, ta có thể định nghĩa hành động ta muốn thực hiện giữa mỗi lần retry — có thể bao gồm error handling hoặc logic re-authentication.

**Cảnh báo quan trọng**: retry policy có thể ảnh hưởng tiêu cực tới hệ thống, dẫn tới concurrency cao và resource contention. Ta phải đảm bảo một policy vững chắc và định nghĩa delay/retry hiệu quả. Một retry policy được cấu hình cẩu thả có thể mở ứng dụng cho vấn đề hiệu năng đáng kể.

### Adding a Circuit Breaker Policy

Ta nên xử lý lỗi mất nhiều thời gian hơn để giải quyết và định nghĩa một policy từ bỏ retry call tới một service khi ta đã kết luận nó không phản hồi lâu hơn mong đợi. Điều ta không muốn làm là tiếp tục retry service call mà không có điều kiện thoát nào. Điều này sẽ giống implement một vòng lặp vô hạn nếu target service vẫn không phản hồi, và có thể vô tình gây ra một cuộc tấn công denial-of-service lên chính service của bạn. Vì lý do này, ta implement **circuit breaker pattern**, đóng vai trò như một orchestrator cho các cuộc gọi service của ta.

Ta có thể giới thiệu một circuit breaker giữa client và server. Ban đầu, nó cho phép mọi cuộc gọi đi qua — gọi là trạng thái **closed**. Nếu có lỗi hoặc phản hồi trễ xảy ra, circuit breaker mở ra (**open**). Một khi circuit breaker mở, các cuộc gọi tiếp theo sẽ fail nhanh hơn — rút ngắn thời gian chờ phản hồi. Nó sẽ chờ một khoảng timeout đã cấu hình rồi cho phép cuộc gọi lại nếu target service đã phục hồi. Nếu không có cải thiện, circuit breaker sẽ ngắt truyền tải.

```csharp
builder.Services.AddHttpClient<PatientsApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiEndpoints:PatientsApi"]);
}).AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());
```

```csharp
static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
}
```

Policy này định nghĩa một circuit breaker mở circuit khi có 5 lần retry thất bại liên tiếp. Một khi đạt ngưỡng đó, circuit sẽ ngắt trong 30 giây và tự động fail mọi cuộc gọi tiếp theo, được diễn giải như HTTP failure response.

Với hai policy này, bạn có thể điều phối retry service và giảm thiểu đáng kể tác động của outage không có kế hoạch lên ứng dụng và trải nghiệm người dùng cuối. Circuit breaker policy cũng bảo vệ khỏi bất kỳ tác dụng phụ tiềm ẩn nào của retry policy.

---

## Phần 6: Adding a Caching Layer

Caching có thể là một phần giá trị của chiến lược resiliency. Ta có thể implement một chiến lược caching fallback về cache này nếu một service không khả dụng. Nghĩa là ta có thể dùng caching layer như một fallback data source và tạo ảo giác cho người dùng cuối rằng mọi service đều đang chạy. Cache này sẽ được cập nhật định kỳ và duy trì mỗi khi dữ liệu trong database của source service bị sửa đổi.

Tất nhiên, với chiến lược này, ta cần chấp nhận hệ quả của việc có khả năng dữ liệu bị lỗi thời (stale). Nếu source service offline và database hỗ trợ đang được cập nhật, có thể bởi job khác, thì cache cuối cùng sẽ trở thành nguồn dữ liệu lỗi thời. Càng nhiều biện pháp ta implement để đảm bảo tính "tươi mới", ta càng đưa vào nhiều độ phức tạp cho ứng dụng.

### Redis — Distributed Cache

Cách hiệu quả nhất để implement caching layer là dưới dạng **distributed cache** — mọi hệ thống trong kiến trúc có thể truy cập cache trung tâm, tồn tại như một service độc lập, bên ngoài. Implementation này có thể tăng tốc độ và hỗ trợ scale.

Redis là công nghệ database caching phổ biến. Nó là in-memory data store mã nguồn mở, cũng có thể được dùng làm message broker. Nó là key-value store dùng một key duy nhất để index một giá trị, không có hai giá trị nào cùng key. Giá trị có thể lưu ở kiểu dữ liệu đơn giản như string, number, list, nhưng JSON được dùng phổ biến cho kiểu object phức tạp hơn.

```bash
docker pull redis
docker run --name redis-cache -d -p 6379:6379 redis
docker exec -it redis-cache redis-cli
```

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Đăng ký RedisCache service
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("RedisConnection");
});
```

```json
"ConnectionStrings": {
  "RedisConnection": "localhost:6379,abortConnect=false"
}
```

### CacheProvider — wrapper interface

```csharp
public interface ICacheProvider
{
    Task ClearCache(string key);
    Task<T> GetFromCache<T>(string key) where T : class;
    Task SetCache<T>(string key, T value, DistributedCacheEntryOptions options) where T : class;
}

public class CacheProvider : ICacheProvider
{
    private readonly IDistributedCache _cache;
    public CacheProvider(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<T> GetFromCache<T>(string key) where T : class
    {
        var cachedResponse = await _cache.GetStringAsync(key);
        return cachedResponse == null ? null : JsonSerializer.Deserialize<T>(cachedResponse);
    }

    public async Task SetCache<T>(string key, T value, DistributedCacheEntryOptions options) where T : class
    {
        var response = JsonSerializer.Serialize(value);
        await _cache.SetStringAsync(key, response, options);
    }

    public async Task ClearCache(string key)
    {
        await _cache.RemoveAsync(key);
    }
}
```

### Áp dụng vào AppointmentsController

```csharp
public class AppointmentsController(
    // Các service khác
    IDistributedCache _cache) : ControllerBase
```

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<Appointment>> GetAppointment(Guid id)
{
    var cacheKey = $"appointment-details:{id}";
    var cached = await _cache.GetStringAsync(cacheKey);
    if (!string.IsNullOrEmpty(cached))
    {
        var cachedDetails = JsonSerializer.Deserialize<AppointmentDetails>(cached);
        return Ok(cachedDetails);
    }

    var appointment = await _context.Appointments.FindAsync(id);
    if (appointment == null)
    {
        return NotFound();
    }

    var patient = await _patientsApiClient.GetPatientAsync(id);
    var doctor = await _doctorsApiClient.GetDoctorAsync(id);

    using var channel = GrpcChannel.ForAddress(_configuration["GrpcEndpoints:DocumentService"]);
    var client = new DocumentService.DocumentServiceClient(channel);
    var documents = await client.GetAllAsync(new PatientId { Id = patient.PatientId.ToString() });

    var appointmentDetails = new AppointmentDetails(
        id, patient, doctor,
        appointment.Slot.Start, appointment.Slot.End,
        appointment.Location.RoomNumber, appointment.Location.Building,
        documents
    );

    var serialized = JsonSerializer.Serialize(appointmentDetails);
    await _cache.SetStringAsync(cacheKey, serialized,
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) // TTL
        });

    return Ok(appointmentDetails);
}
```

Lợi ích ở đây rất đáng kể vì dù ta đặt TTL là 10 phút, con số này có thể lâu hơn nhiều cho dữ liệu ít thay đổi thường xuyên. Thêm nữa, vì cuộc gọi này dùng ba nguồn dữ liệu, thêm cache sẽ nâng cao hiệu năng của các query tiếp theo cho chi tiết cùng bản ghi lịch hẹn.

---

## Phần 7: Reducing Authentication Errors

Bảo mật web service là thực hành thông thường, đây là kỳ vọng cho cả API nội bộ lẫn bên thứ ba. Mỗi microservice nên tự xác thực hoặc hành động thay mặt cho một identity người dùng/hệ thống.

### API Key qua Delegating Handler

```csharp
public class ApiKeyHandler(IConfiguration _configuration) : DelegatingHandler
{
    private string _headerName = "x-api-key";
    private string _apiKey = _configuration["ApiKey"];

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        if (!request.Headers.Contains(_headerName))
        {
            request.Headers.Add(_headerName, _apiKey);
        }
        return base.SendAsync(request, cancellationToken);
    }
}
```

```csharp
builder.Services.AddTransient<ApiKeyHandler>();
builder.Services.AddHttpClient<ExternalApiService>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ThirdPartyApi:BaseUrl"]);
})
.AddHttpMessageHandler<ApiKeyHandler>();
```

```csharp
public class ExternalApiService(HttpClient _httpClient)
{
    public async Task<string> GetSecureDataAsync()
    {
        var response = await _httpClient.GetAsync("/secure-data");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```

### JWT — phương pháp phổ biến hơn cho service nội bộ

JWT (JSON Web Token) là token gọn nhẹ, an toàn cho URL, dùng cho authentication và authorization. Nó được cấp cho client sau một lần login thành công. JWT hoàn hảo cho microservices vì API stateless và không track session xác thực.

```csharp
using System.Net.Http.Headers;

public class JwtAuthHandler(IConfiguration _configuration) : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var token = _configuration["JwtSettings:Token"];
        if (!string.IsNullOrEmpty(token))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        }
        return base.SendAsync(request, cancellationToken);
    }
}
```

```csharp
builder.Services.AddTransient<JwtAuthHandler>();
builder.Services.AddHttpClient<DoctorsApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiEndpoints:DoctorsApi"]);
})
.AddHttpMessageHandler<JwtAuthHandler>();
```

### Hai kịch bản lỗi bổ sung cần xử lý

- **Expired JWT**: token được cấp với thời hạn giới hạn. Nếu token hiện diện nhưng hết hạn, code phải được viết sao cho một token mới được lấy và dùng ở cuộc gọi tiếp theo — theo cách này, lỗi ban đầu không ảnh hưởng tới người dùng cuối.
- **Forbidden (403)**: token xác thực đúng (nhận diện đúng service/client gọi), nhưng app không có quyền cần thiết để truy cập resource yêu cầu. Có rất ít điều có thể làm để ngăn điều này ảnh hưởng tới service khác.

---

## Phần 8: Implementing Fault-Handling Policies trong Asynchronous Communication

### Retry Policy cho Message Consumer với MassTransit

Giao tiếp bất đồng bộ không xảy ra trực tiếp giữa các service mà đòi hỏi một hệ thống chuyển giao message trung gian, như message broker. Message broker được thiết kế để bền vững trong lúc truyền tải message và có cơ chế hỗ trợ lưu giữ message tới khi consumer xử lý chúng.

```csharp
services.AddMassTransit(x =>
{
    x.AddConsumer<AppointmentCreatedConsumer>(cfg =>
    {
        cfg.UseMessageRetry(r =>
        {
            r.Interval(3, TimeSpan.FromSeconds(5)); // Retry 3 lần, cách nhau 5 giây
        });
    });
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ConfigureEndpoints(context);
    });
});
```

Retry policy này đảm bảo nếu consumer throw exception, message sẽ được retry 3 lần với khoảng cách 5 giây trước khi được coi là thất bại.

### Dead-Letter Queue (DLQ)

Vì message delivery không được đảm bảo trong hệ thống bất đồng bộ, DLQ là một queue đặc biệt được thiết kế để bắt lấy message không thể xử lý thành công, ngay cả sau nhiều lần retry. Chúng đóng vai trò lưới an toàn, cho phép operator và developer kiểm tra, phân tích, và có thể xử lý lại message thất bại mà không mất thông tin quan trọng hoặc làm crash hệ thống.

Với cấu hình trước đó, MassTransit tự động chuyển một message thất bại sang error/DLQ. Ví dụ, lỗi trong `AppointmentCreatedConsumer` sẽ dẫn tới message được chuyển hướng tới queue `appointment-created_error`. Queue này sau đó có thể được monitor hoặc rút cạn cho mục đích chẩn đoán.

### Khác biệt giữa các Message Broker

MassTransit được thiết kế cung cấp abstraction thống nhất trên nhiều message broker. Do đó, cấu hình retry, dead-lettering, outbox, và consumer hoạt động trên nhiều transport provider, nhưng hành vi thay đổi tùy theo broker và khả năng của nó:

- **Azure Service Bus**: retry và DLQ được map khác biệt. Azure có sẵn DLQ native dưới dạng subqueue `$DeadLetterQueue`, nhưng MassTransit cũng mô phỏng `_error` queue. Message TTL và session support được map tới construct của Azure.
- **Amazon SQS**: MassTransit hoạt động tốt với SQS và basic retry policy, nhưng SQS không hỗ trợ một số pattern nâng cao. DLQ phải được cấu hình thủ công trong AWS, không được tự động tạo bởi MassTransit. Visibility timeout và max receive count do AWS quản lý, không phải MassTransit retry configuration.

Nói chung, quan trọng là hiểu công nghệ message broker/queue đã chọn hoạt động ra sao và xác định liệu fault-tolerance có thể được enforce trong code hay phải được cấu hình thủ công trên nền tảng.

---

## Phần 9: Using the Outbox Pattern

Outbox pattern là kỹ thuật đảm bảo giao tiếp đáng tin cậy và nhất quán giữa các service. Nó tận dụng database local như một buffer bền vững cho message cần publish. Pattern này đảm bảo tính atomic: hoặc cả database update và message write cùng xảy ra, hoặc không cái nào xảy ra.

### Định nghĩa OutboxMessage Entity

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string EventType { get; set; } = default!;
    public string Payload { get; set; } = default!;
    public DateTime OccurredOnUtc { get; set; }
    public bool Processed { get; set; }
    public DateTime? ProcessedOnUtc { get; set; }
}
```

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<OutboxMessage> OutboxMessages { get; set; } = default!;
    // ... các DbSet và cấu hình khác
}
```

### Sửa POST Method để dùng Outbox

```csharp
[HttpPost]
public async Task<ActionResult<Appointment>> PostAppointment(Appointment appointment)
{
    // Code hiện có để thêm bản ghi vào database
    // Rút gọn để súc tích

    var outboxMessage = new OutboxMessage
    {
        Id = Guid.NewGuid(),
        EventType = nameof(AppointmentCreated),
        Payload = JsonSerializer.Serialize(appointmentCreatedEvent),
        OccurredOnUtc = DateTime.UtcNow,
        Processed = false
    };
    _context.OutboxMessages.Add(outboxMessage);
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();

    return CreatedAtAction("GetAppointment", new { id = appointment.AppointmentId }, appointment);
}
```

Toàn bộ thao tác này được bọc trong một transaction để đảm bảo mọi thứ hoặc thành công hoặc thất bại toàn bộ.

### OutboxProcessor — Background Service

```csharp
public class OutboxProcessor(IServiceProvider _serviceProvider, ILogger<OutboxProcessor> _logger)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _serviceProvider.CreateScope();
            // Khởi tạo các service cần thiết, như db context

            var messages = await dbContext.OutboxMessages
                .Where(m => !m.Processed)
                .OrderBy(m => m.OccurredOnUtc)
                .Take(20)
                .ToListAsync(stoppingToken);

            foreach (var message in messages)
            {
                try
                {
                    await messagePublisher.Publish(message.Payload);
                    message.Processed = true;
                    message.ProcessedOnUtc = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    // Tùy chọn implement retry logic hoặc chuyển tới dead-letter queue
                }
            }
            await dbContext.SaveChangesAsync();
        }
    }
}
```

```csharp
builder.Services.AddHostedService<OutboxProcessor>();
```

Ta có thể đảm bảo thay đổi database và publish event luôn nhất quán. Nếu message broker tạm thời không khả dụng, message vẫn ở trong outbox và được retry sau. Bảng outbox cũng đóng vai trò như một log của event, hỗ trợ monitoring và debugging.

---

## Tổng kết chương

Chương này khám phá kỹ thuật và công cụ để nâng cao fault tolerance, độ ổn định, và hiệu năng của hệ thống phân tán. Dùng hệ thống quản lý phòng khám làm domain, chương đặt bối cảnh cho các kịch bản thực tế nơi lỗi trong giao tiếp service — dù đồng bộ hay bất đồng bộ — có thể gây hệ quả nghiêm trọng.

Ta bắt đầu bằng việc nhấn mạnh tính tất yếu của lỗi trong hệ phân tán, làm rõ ứng dụng hiện đại phải được thiết kế để phục hồi graceful từ lỗi thay vì giả định vận hành liên tục không gián đoạn. Ta phân loại các lỗi này: timeout, unavailability, và authentication breakdown — và minh họa cách thiết kế resilient giảm thiểu tác động của chúng bằng Polly (transient fault handling), Redis (distributed caching), và outbox pattern (event propagation đáng tin cậy).

Với giao tiếp đồng bộ, ta học retry policy dùng Polly xử lý lỗi tạm thời với exponential backoff, và circuit breaker pattern bảo vệ khỏi retry storm hoặc vòng lặp vô hạn. Ta khám phá tầm quan trọng của caching cho endpoint nặng về đọc, implement chi tiết với Redis và một service layer tái sử dụng được. Ta cũng học cách API key và JWT có thể được tự động inject vào HTTP request qua custom delegating handler, tập trung hóa và bảo mật identity propagation.

Với giao tiếp bất đồng bộ, ta học retry policy cho message consumer qua MassTransit, dead-letter queue như lưới an toàn, và sự khác biệt hành vi giữa các message broker (RabbitMQ, Azure Service Bus, Amazon SQS). Cuối cùng, outbox pattern được implement chi tiết từng bước — từ định nghĩa entity, thêm vào DbContext, tới publish message và cập nhật trạng thái — giải quyết triệt để "dual write problem" kinh điển.

**Bài học cốt lõi**: xây dựng resilient microservices không phải là ngăn chặn mọi lỗi (điều bất khả thi trong hệ phân tán), mà là thiết kế hệ thống có khả năng phát hiện lỗi nhanh, cô lập chúng, và phục hồi tự động — kết hợp retry strategy, circuit breaker, distributed caching, authentication có trách nhiệm, và đảm bảo messaging bất đồng bộ để tạo ra hệ thống suy giảm graceful và phục hồi nhanh chóng, đúng như những gì microservices hiện đại đòi hỏi.

Chương tiếp theo sẽ chuyển sang implement API Gateway pattern.
