# Bài học: Giao tiếp Đồng bộ giữa các Microservices — Khi "gọi hàm" trở thành "gọi mạng"

> Chương 3 — Synchronous Communication between Microservices
> Ví dụ xuyên suốt: AppointmentsApi (RESTful) gọi PatientsApi/DoctorsApi (RESTful) và DocumentsService (gRPC) trong hệ thống quản lý phòng khám

---

## Phần 1: Bản chất của giao tiếp đồng bộ — và cái giá của nó

Sau Chapter 2, ta biết rằng dữ liệu sẽ được chia sẻ giữa các microservice, đòi hỏi giao tiếp để trao đổi thông tin liên quan. Mỗi microservice thường quản lý một phạm vi chức năng riêng, thường bao gồm một kho dữ liệu tách biệt. Tại một thời điểm nào đó, một microservice sẽ cần dữ liệu từ microservice khác — đây là lúc ta cần giao tiếp liên service.

**Giao tiếp đồng bộ (synchronous communication)** là mô hình mà một service gọi trực tiếp service khác và chờ phản hồi trước khi tiếp tục xử lý — giống hệt các lệnh gọi hàm truyền thống, nơi bên gọi bị block cho tới khi bên được gọi trả về kết quả.

Ba đặc điểm cốt lõi:

- **Immediate response**: bên gọi mong đợi phản hồi ngay lập tức, dẫn tới tương tác ràng buộc chặt (tightly coupled).
- **Blocking behavior**: bên gọi bị dừng lại cho tới khi nhận được phản hồi, có thể ảnh hưởng hiệu năng nếu bên được gọi chậm hoặc không phản hồi.
- **Simplified error handling**: lỗi có thể được phát hiện và xử lý ngay lập tức, vì bên gọi biết kết quả tức thì.

Có hai loại giao tiếp lớn: đồng bộ và bất đồng bộ. Nếu thao tác cần phản hồi ngay lập tức, ta dùng kỹ thuật đồng bộ; với các tiến trình chạy dài không nhất thiết cần phản hồi ngay, ta chọn bất đồng bộ.

### Ví dụ thực tế trong hệ thống phòng khám

Một truy vấn đơn giản từ frontend cần được thực hiện đồng bộ. Giả sử cần hiển thị danh sách toàn bộ bác sĩ trong hệ thống cho người dùng — ta cần gọi trực tiếp tới microservice API bác sĩ, lấy dữ liệu từ database và trả về. Đây là thao tác đơn giản có thể thực hiện bởi bất kỳ hệ thống nào, ở bất kỳ ngôn ngữ nào — nhưng cần lưu ý tới nhược điểm tiềm ẩn: cách triển khai này khiến các service phụ thuộc lẫn nhau, làm hệ thống kém linh hoạt và khó scale hơn. Nếu service được gọi bị chậm, bên gọi cũng bị chậm theo, ảnh hưởng tới hiệu năng tổng thể.

---

## Phần 2: Ba mối quan tâm bắt buộc khi thiết kế giao tiếp đồng bộ

### 2.1. Performance — Độ trễ cộng dồn (Cumulative Latency)

Ta luôn cân nhắc hiệu năng khi phát triển giải pháp — không chỉ ở từng service riêng lẻ mà cả trong kịch bản giao tiếp giữa chúng. Khi một service gọi service khác, cần đảm bảo cuộc gọi được thực hiện theo cách hiệu quả nhất có thể. Vì phần lớn dùng RESTful API, giao tiếp HTTP là lựa chọn mặc định; gRPC cũng là lựa chọn để đạt throughput cao hơn và độ trễ thấp hơn.

Một mối lo ngại quan trọng với giao tiếp đồng bộ là **độ trễ cộng dồn** khi các cuộc gọi service xuyên qua mạng. Mỗi cuộc gọi tới service downstream tạo ra một network hop. Mỗi hop bao gồm thời gian truyền tải mạng, độ trễ xử lý ở service đích, và khả năng bị xếp hàng chờ nếu service đang chịu tải cao. Dù mỗi cuộc gọi chỉ thêm vài mili-giây, tổng độ trễ có thể tăng nhanh khi càng nhiều service tham gia vào chuỗi xử lý. Thiết kế đường đi giao tiếp với số hop tối thiểu, cùng các cơ chế fallback hoặc timeout, trở nên thiết yếu để duy trì hiệu năng và độ tin cậy.

### 2.2. Resilience — Hai pattern sống còn

Ta phải đảm bảo các cuộc gọi service được thực hiện qua các kênh bền vững, vì phần cứng có thể hỏng hoặc mạng có thể gián đoạn ngay khi cuộc gọi đang thực thi. Hai pattern chính giúp service trở nên resilient:

- **Retry Pattern**: Lỗi tạm thời (transient failure) là các lỗi ngắn hạn có thể làm gián đoạn việc hoàn tất một thao tác, nhưng thường tự biến mất. Ta ưu tiên retry thao tác vài lần theo cấu hình thay vì fail hoàn toàn ngay lập tức; nếu vẫn không thành công, ta trigger timeout. **Cần đặc biệt cẩn trọng với các thao tác ghi dữ liệu**: request có thể đã được gửi đi và lỗi tạm thời chỉ ngăn cản việc nhận phản hồi — không có nghĩa là thao tác chưa hoàn tất, và việc retry mù quáng có thể dẫn tới kết quả không mong muốn (thao tác bị thực hiện trùng lặp).

- **Circuit Breaker Pattern**: Pattern này giới hạn số lần ta cố thử gọi một service. Nhiều cuộc gọi có thể fail vì lỗi tạm thời mất nhiều thời gian để tự khắc phục, hoặc vì số lượng request gửi tới service gây nghẽn tài nguyên hệ thống sẵn có. Ta có thể cấu hình pattern này để giới hạn thời gian ta dành cho việc cố gọi một service.

### 2.3. Tracing and Monitoring

Một thao tác đơn lẻ giờ đây có thể trải dài qua nhiều service — điều này đặt ra thách thức mới cho việc giám sát và theo dõi hoạt động xuyên suốt các service từ một điểm khởi đầu. Ta cần công cụ phù hợp để xử lý distributed logging và tổng hợp log vào một nơi trung tâm để dễ theo dõi và truy vết vấn đề. Các công cụ như OpenTelemetry và Azure Monitor có thể hỗ trợ nhiệm vụ này.

### Lợi ích của giao tiếp đồng bộ

Giao tiếp đồng bộ dễ implement và dễ hiểu nhờ bản chất request-response đơn giản, trực tiếp. Ta cũng được hưởng lợi từ việc lấy/xử lý dữ liệu thời gian thực, giúp biết ngay liệu thao tác thành công hay thất bại. Trong trường hợp lỗi, luồng thực thi tuyến tính giúp việc truy vết và xử lý sự cố dễ dàng hơn.

---

## Phần 3: HTTP — nền móng của mọi giao tiếp đồng bộ hiện đại

HTTP hoạt động theo mô hình request-response, nơi client khởi tạo request và server xử lý, trả về response tương ứng. Mỗi HTTP request độc lập, không có "ký ức" cố hữu về các tương tác trước đó — bản chất stateless này đơn giản hóa thiết kế server, nhưng đòi hỏi cơ chế như cookie hoặc session khi cần duy trì trạng thái.

HTTP sở hữu thiết kế đơn giản, định dạng dựa trên văn bản, dễ implement và debug — góp phần vào việc được áp dụng rộng rãi. Nó cũng hỗ trợ nhiều method (GET, POST, PUT, DELETE) định nghĩa các hành động cụ thể trên resource.

### HTTP Headers — không chỉ là metadata vặt vãnh

Trong hệ thống dựa trên microservices, header thường được dùng cho:

- **Authentication and authorization**: Header như `Authorization` mang theo token (JWT, API key, OAuth token) cho phép service xác thực danh tính và quyền hạn của bên gọi.
- **Content type declaration**: `Content-Type` chỉ định media type của request/response body (VD: `application/json`), giúp bên nhận parse đúng payload. Tương tự, `Accept` cho server biết client mong đợi định dạng phản hồi nào.
- **Correlation and tracing**: Header như `X-Request-ID`, `traceparent`, `baggage` (dùng trong distributed tracing) giúp lan truyền trace/correlation ID qua các service, phục vụ observability và debug tốt hơn.
- **Caching and compression**: Header như `Cache-Control`, `ETag`, `Content-Encoding` ảnh hưởng tới chính sách cache và nén phản hồi để cải thiện hiệu năng.
- **Custom metadata**: Microservice có thể định nghĩa và trao đổi header tùy chỉnh để mang dữ liệu theo ngữ cảnh mà không cần thay đổi request body.

### HTTP Status Code — 5 nhóm cần nhớ

- **1xx (Informational)**: truyền tải thông tin cấp giao thức
- **2xx (Success)**: request được chấp nhận, không có lỗi trong quá trình xử lý
- **3xx (Redirection)**: cần đi theo một đường khác để hoàn tất request ban đầu
- **4xx (Client Error)**: lỗi phát sinh từ phía request, VD dữ liệu sai định dạng (400) hoặc địa chỉ sai (404)
- **5xx (Server Error)**: server thất bại trong việc hoàn thành tác vụ vì lý do không lường trước

---

## Phần 4: SOAP vs REST — vì sao ngành công nghiệp chuyển dịch

REST (Representational State Transfer) và SOAP (Simple Object Access Protocol) là hai paradigm nổi bật cùng dùng HTTP cho web service. Dù cả hai đều cho phép trao đổi dữ liệu qua mạng, chúng khác biệt đáng kể về nguyên tắc thiết kế, định dạng dữ liệu, và trường hợp sử dụng.

### SOAP

SOAP được phát triển cuối những năm 1990 để trao đổi thông tin có cấu trúc và triển khai web service. Nó dựa trên XML cho định dạng message, thường dùng HTTP/HTTPS để thương lượng và truyền tải message. SOAP cũng hỗ trợ bảo mật (WS-Security) và transaction (cho phép rollback thao tác), phù hợp cho ứng dụng cấp doanh nghiệp.

Ví dụ SOAP request lấy thông tin lịch hẹn:

```xml
<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:app="http://www.example.com/appointment">
  <soap:Header/>
  <soap:Body>
    <app:GetAppointmentDetails>
      <app:AppointmentId>123e4567-e89b-12d3-a456-426614174000</app:AppointmentId>
    </app:GetAppointmentDetails>
  </soap:Body>
</soap:Envelope>
```

Web service có thể đạt được giao tiếp độc lập nền tảng (platform-independent), tuân thủ nghiêm ngặt cấu trúc message đã thỏa thuận bằng cách dùng SOAP với XML schema định nghĩa rõ ràng. SOAP vẫn là lựa chọn phổ biến nhờ tính chuẩn hóa cao, độ tin cậy, và các tính năng cấp doanh nghiệp — hoạt động trên hợp đồng chính thức (formal contract), đảm bảo cấu trúc message nghiêm ngặt và validate kiểu dữ liệu, đặc biệt hữu ích trong các ngành được quản lý chặt như tài chính và y tế. SOAP cũng hỗ trợ sẵn xử lý lỗi, quản lý transaction, WS-Security, và mã hóa cấp message.

**Hạn chế của SOAP**:
- **Verbosity**: message dựa trên XML có thể rất dài dòng, dẫn tới kích thước message lớn hơn và tốn băng thông hơn.
- **Complexity**: tính chất mở rộng của đặc tả khiến việc implement và bảo trì trở nên thách thức.
- **Performance**: parse XML có thể tốn nhiều tài nguyên, ảnh hưởng hiệu năng.

### REST

REST là framework hiệu quả hơn, đã thay thế phần lớn SOAP, mang đến cách tiếp cận đơn giản và nhẹ nhàng hơn cho web service. REST tận dụng giao thức HTTP chuẩn, dễ implement và tiêu thụ; tính stateless và cơ chế caching giúp tăng khả năng mở rộng.

Lý do lựa chọn REST:

- **Flexibility in data formats**: khác với SOAP chỉ giới hạn ở XML, REST hỗ trợ nhiều định dạng dữ liệu bao gồm JSON, XML, HTML, plain text. JSON nhẹ và dễ parse hơn, cải thiện hiệu năng và tương thích với ứng dụng web.
- **Performance and scalability**: tính stateless và hỗ trợ caching của REST cải thiện hiệu năng và khả năng mở rộng, giảm tải server và đơn giản hóa việc scale.
- **Lower bandwidth consumption**: việc dùng JSON giúp REST tiêu thụ ít băng thông hơn message XML của SOAP — hiệu quả này rất quan trọng với ứng dụng lưu lượng cao và thiết bị di động.
- **Better support for web clients**: khả năng tương thích với công nghệ web và các thao tác stateless khiến REST phù hợp hơn với client web, bao gồm trình duyệt và ứng dụng di động.

Cùng ví dụ appointment, response JSON của REST:

```json
{
  "Appointment": {
    "AppointmentId": "123e4567-e89b-12d3-a456-426614174000",
    "PatientId": "987e6543-e21b-34d3-c456-426614174111",
    "DoctorId": "654e3210-e54b-56d3-d456-426614174222",
    "StartTime": "2024-12-01T10:00:00",
    "EndTime": "2024-12-01T11:00:00",
    "RoomNumber": "101",
    "Building": "Main",
    "Purpose": "General Consultation"
  }
}
```

Cấu trúc JSON này mang lại định dạng ngắn gọn và dễ đọc hơn cho chi tiết lịch hẹn, so với XML dài dòng của SOAP.

---

## Phần 5: Xây dựng một RESTful Microservice trong .NET — quy trình thực chiến

RESTful API expose các method cho phép truy cập chức năng bên dưới thông qua các HTTP call chuẩn. Một call/request thường bao gồm: URL/endpoint (địa chỉ resource cần tương tác), verb/method (GET để lấy dữ liệu, POST để tạo record, PUT để cập nhật, DELETE để xóa), và data (đi kèm khi cần để hoàn tất request).

### Khởi tạo project

```bash
dotnet new webapi -o AppointmentsApi
cd AppointmentsApi
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

### Định nghĩa Model (kế thừa tinh thần DDD từ Chapter 2)

```csharp
[Owned]
public record TimeSlot(DateTime Start, DateTime End);
[Owned]
public record Location(string RoomNumber, string Building);

public class Appointment
{
    public Guid AppointmentId { get; set; }
    public Guid PatientId { get; set; }
    public Guid DoctorId { get; set; }
    public TimeSlot Slot { get; set; }
    public Location Location { get; set; }
    public string Purpose { get; set; }

    public void Reschedule(TimeSlot newSlot)
    {
        Slot = newSlot;
    }

    public void ChangePurpose(string newPurpose)
    {
        Purpose = newPurpose;
    }
}
```

Attribute `[Owned]` được thêm vào các value object — đây là yêu cầu của EF Core để mô hình hóa các entity type chỉ xuất hiện trên navigation property của entity type khác.

### Database Context

```csharp
public class AppointmentContext : DbContext
{
    public AppointmentContext(DbContextOptions<AppointmentContext> options)
        : base(options) { }

    public DbSet<Appointment> Appointments { get; set; }
}
```

Đăng ký trong `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddDbContext<AppointmentContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

### Migration

```bash
dotnet tool install --global dotnet-ef
dotnet ef migrations add InitialCreate
dotnet ef database update
```

### Scaffold Controller

ASP.NET Core có 2 kiểu template API: **Controller-based** (dùng class chuyên biệt đóng gói HTTP/RESTful logic) và **Minimal API** (không cần class riêng, viết trực tiếp trong `Program.cs`, nhẹ hơn nhưng đường cong học tập dốc hơn để làm quen với pattern).

```bash
dotnet tool install --global dotnet-aspnet-codegenerator
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet aspnet-codegenerator controller -name AppointmentsController -async \
  -api -m Appointment -dc AppointmentContext -outDir Controllers -dbProvider sqlite
```

Controller được sinh ra:

```csharp
[Route("api/[controller]")]
[ApiController]
public class AppointmentsController : ControllerBase
{
    private readonly AppointmentContext _context;

    public AppointmentsController(AppointmentContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Appointment>>> GetAppointments()
    {
        return await _context.Appointments.ToListAsync();
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Appointment>> GetAppointment(Guid id)
    {
        var appointment = await _context.Appointments.FindAsync(id);
        if (appointment == null)
        {
            return NotFound();
        }
        return appointment;
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> PutAppointment(Guid id, Appointment appointment)
    {
        if (id != appointment.AppointmentId)
        {
            return BadRequest();
        }
        _context.Entry(appointment).State = EntityState.Modified;
        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!AppointmentExists(id)) return NotFound();
            else throw;
        }
        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteAppointment(Guid id)
    {
        var appointment = await _context.Appointments.FindAsync(id);
        if (appointment == null)
        {
            return NotFound();
        }
        _context.Appointments.Remove(appointment);
        await _context.SaveChangesAsync();
        return NoContent();
    }
}
```

**Ý nghĩa các attribute**:
- `[Route("api/[controller]")]`: thiết lập pattern routing, ánh xạ HTTP request tới `api/appointments`.
- `[ApiController]`: bật các tính năng như tự động model validation và binding.
- `ControllerBase`: base class cung cấp chức năng RESTful/HTTP thiết yếu.

Controller dùng dependency injection để truy cập `AppointmentContext` — đảm bảo tham chiếu (như kết nối database) được dispose đúng cách sau khi dùng cho một tác vụ cụ thể.

### Testing với REST Client (VS Code extension)

Tạo file `api-requests.http`:

```http
# Retrieve all appointments
GET https://{API_ADDRESS}/api/appointments
Accept: application/json

###
# Create a new appointment
POST https://{API_ADDRESS}/api/appointments
Content-Type: application/json

{
  "patientId": "987e6543-e21b-34d3-c456-426614174111",
  "doctorId": "654e3210-e54b-56d3-d456-426614174222",
  "slot": {
    "start": "2024-12-01T10:00:00",
    "end": "2024-12-01T11:00:00"
  },
  "location": {
    "roomNumber": "101",
    "building": "Main"
  },
  "purpose": "General Consultation"
}
```

Kết quả trả về HTTP 201 — mã chuẩn cho thao tác POST thành công tạo resource mới. Test thủ công có thể trở nên nặng nề với ứng dụng lớn hơn — unit testing tự động hóa là hướng tiếp cận phổ biến hơn.

---

## Phần 6: HttpClient — cách một service gọi service khác

Khi nói về giao tiếp API-tới-API đồng bộ, API thực hiện cuộc gọi đóng vai trò là client. .NET cung cấp thư viện `HttpClient` với các method ánh xạ tới từng HTTP verb.

### Cấu hình endpoint

```json
"ApiEndpoints": {
  "DoctorsApi": "DOCTORS_API_ENDPOINT",
  "PatientsApi": "PATIENTS_API_ENDPOINT"
}
```

### Đăng ký Typed Client trong Program.cs

```csharp
builder.Services.AddHttpClient<PatientsApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiEndpoints:PatientsApi"]);
});

builder.Services.AddHttpClient<DoctorsApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiEndpoints:DoctorsApi"]);
});
```

Cách tiếp cận này cho phép đăng ký các typed client — logic của mỗi client được đóng gói trong class riêng tương ứng với code cần để tương tác với API đó, thúc đẩy separation of concerns. Cách này cũng tập trung hóa cấu hình để logic có thể được mock hoặc thay thế trong unit test, tăng khả năng testability.

### Implement Client

```csharp
public record Patient(
    Guid PatientId, string FirstName, string LastName,
    DateTime DateOfBirth, string Gender, string ContactNumber, string Email
);

public class PatientsApiClient
{
    private readonly HttpClient _httpClient;
    public PatientsApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<Patient> GetPatientAsync(Guid patientId)
    {
        var response = await _httpClient.GetAsync($"/api/patients/{patientId}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Patient>();
    }
}
```

Cả hai implementation (`PatientsApiClient`, `DoctorsApiClient`) chia sẻ đặc điểm chung:

- **HttpClient injection**: vì typed client được đăng ký với base URL tương ứng, `HttpClient` có thể được inject an toàn và tự động được dispose đúng cách sau thao tác. Điều này quan trọng để giảm nguy cơ **socket exhaustion** — hệ quả của việc kết nối `HttpClient` được mở nhưng không đóng đúng cách.
- **Get method**: dùng `HttpClient` đã inject để gọi GET tới endpoint, tự deserialize JSON trả về thành object C#.
- **Models**: cả hai implementation dùng record đóng vai trò model/data transfer object, khớp theo tên và kiểu dữ liệu C# phù hợp với response JSON mong đợi.

### Sử dụng trong Controller

```csharp
public class AppointmentsController : ControllerBase
{
    private readonly AppointmentContext _context;
    private readonly PatientsApiClient _patientsApiClient;
    private readonly DoctorsApiClient _doctorsApiClient;

    public AppointmentsController(AppointmentContext context,
        PatientsApiClient patientsApiClient, DoctorsApiClient doctorsApiClient)
    {
        _context = context;
        _patientsApiClient = patientsApiClient;
        _doctorsApiClient = doctorsApiClient;
    }

    public async Task<ActionResult<Appointment>> GetAppointment(Guid id)
    {
        var appointment = await _context.Appointments.FindAsync(id);
        if (appointment == null)
        {
            return NotFound();
        }

        // Gọi đồng bộ tới 2 service khác để lấy thông tin bổ sung
        var patient = await _patientsApiClient.GetPatientAsync(id);
        var doctor = await _doctorsApiClient.GetDoctorAsync(id);

        AppointmentDetails appointmentDetails = new AppointmentDetails(
            id, patient, doctor,
            appointment.Slot.Start, appointment.Slot.End,
            appointment.Location.RoomNumber, appointment.Location.Building
        );
        return Ok(appointmentDetails);
    }
}
```

Method này gọi đồng bộ tới API bác sĩ và bệnh nhân để lấy chi tiết về các entity đó, sau đó tổng hợp thành một object đầy đủ thông tin hơn.

---

## Phần 7: gRPC — khi REST không đủ nhanh

RPC (Remote Procedure Call) xuất hiện từ những năm 1960-1970, cho phép chương trình thực thi một thủ tục trên server từ xa như thể nó chạy cục bộ. Tuy nhiên, các implementation RPC ban đầu thường dẫn tới tight coupling giữa client và server, gây khó khăn khi thay đổi hoặc cập nhật mà không ảnh hưởng cả hai đầu. Một số thách thức đáng lưu ý:

- **Tight coupling**: RPC thường khiến việc thay đổi/cập nhật gần như bất khả thi mà không ảnh hưởng cả hai đầu.
- **Platform dependency**: các implementation RPC ban đầu thường đặc thù theo nền tảng, cản trở khả năng liên thông hệ thống.
- **Scalability issues**: bản chất đồng bộ của RPC có thể dẫn tới nghẽn hiệu năng, đặc biệt trên mạng độ trễ cao.

**gRPC**, framework RPC hiệu năng cao do Google phát triển, mang lại hiệu năng cải thiện, hỗ trợ đa nền tảng và giải quyết nhiều hạn chế của RPC truyền thống.

### Lợi ích cụ thể của gRPC

- **Compactness**: tạo ra payload nhỏ hơn, giảm sử dụng băng thông mạng.
- **Faster serialization and deserialization**: định dạng nhị phân của Protocol Buffers (Protobuf) cho phép xử lý nhanh hơn so với JSON dạng văn bản của REST.
- **Server streaming**: server có thể gửi nhiều phản hồi cho một request duy nhất từ client — hữu ích khi cung cấp luồng dữ liệu liên tục.
- **Client streaming**: client gửi các request tới server, server xử lý và trả về một phản hồi duy nhất.
- **Bidirectional streaming**: cả client và server trao đổi message đồng thời, cho phép giao tiếp thời gian thực.

gRPC được ASP.NET Core hỗ trợ đầy đủ, là ứng viên tốt cho giải pháp microservices dựa trên .NET Core. Với bản chất contract-based, nó tự nhiên enforce các tiêu chuẩn và kỳ vọng cụ thể mà ta thường cố mô phỏng khi tạo REST API service class bằng interface.

gRPC cũng có khả năng liên thông đa ngôn ngữ (cross-language interoperability) — tự động sinh code client/server ở nhiều ngôn ngữ lập trình bao gồm C#, Java, Go, Python, Node.js, Ruby, Kotlin, và Swift. Điều này cho phép các team xây dựng microservice bằng ngôn ngữ khác nhau trong khi vẫn duy trì một hợp đồng nhất quán, type-safe xuyên suốt toàn hệ thống — lựa chọn tuyệt vời cho kiến trúc microservices đa ngôn ngữ (polyglot).

### HTTP/2 — nền tảng của gRPC

gRPC hoạt động trên HTTP/2, một bước tiến đáng kể so với HTTP/1.1:

- **Multiplexing**: cho phép nhiều request/response được gửi đồng thời qua một kết nối TCP duy nhất, giảm độ trễ và cải thiện throughput.
- **Binary framing and compression**: dùng định dạng nhị phân gọn nhẹ cho truyền tải dữ liệu, giúp parse nhanh hơn và giảm băng thông so với JSON dạng văn bản.

### Tạo gRPC Microservice — DocumentsService

```bash
dotnet new grpc -o DocumentsService
```

Định nghĩa contract bằng file `.proto` (đóng vai trò Interface Definition Language — IDL):

```protobuf
syntax = "proto3";
option csharp_namespace = "DocumentsService.Protos";
package DocumentSearch;

service DocumentService {
  rpc GetAll (PatientId) returns (DocumentList);
  rpc Get (DocumentId) returns (Document);
}

message Empty{}
message Document {
  string patientId = 1;
  string name = 2;
  string Id = 3;
}
message DocumentList {
  repeated Document documents = 1;
}
message DocumentId {
  string Id = 1;
}
message PatientId {
  string Id = 1;
}
```

Cấu hình trong `.csproj`:

```xml
<ItemGroup>
  <Protobuf Include="Protos\documents.proto" GrpcServices="Server" />
</ItemGroup>
```

Implementation class:

```csharp
public class DocumentServiceImpl : DocumentService.DocumentServiceBase
{
    private static readonly List<Document> Documents = new List<Document>
    {
        new Document { Id = "1", PatientId = "123", Name = "Document1" },
        new Document { Id = "2", PatientId = "456", Name = "Document2" }
    };

    public override Task<DocumentList> GetAll(PatientId request, ServerCallContext context)
    {
        var documentList = new DocumentList();
        documentList.Documents.AddRange(Documents.Where(q => q.Id == request.Id));
        return Task.FromResult(documentList);
    }

    public override Task<Document> Get(DocumentId request, ServerCallContext context)
    {
        var document = Documents.Find(doc => doc.Id == request.Id);
        if (document == null)
        {
            throw new RpcException(new Status(StatusCode.NotFound, "Document not found"));
        }
        return Task.FromResult(document);
    }
}
```

Đáng chú ý: các model như `Document` chưa từng được tạo tường minh — trong gRPC service, cấu trúc dữ liệu và service contract được định nghĩa dựa trên nội dung file `.proto`. Message `Document` sẽ tự động được chuyển thành class C# tương ứng trong quá trình build, bao gồm các property phản chiếu field đã định nghĩa, cùng các interface/method cần thiết cho việc serialize/deserialize.

Đăng ký service trong `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddGrpc();
var app = builder.Build();
app.MapGrpcService<DocumentServiceImpl>();
app.Run();
```

### Gọi gRPC service từ RESTful API

```bash
dotnet add package Grpc.Net.Client
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools
```

Cấu hình `.csproj` để sinh client code:

```xml
<ItemGroup>
  <Protobuf Include="Protos\documents.proto" GrpcServices="Client" />
</ItemGroup>
```

Cấu hình endpoint động qua `appsettings.json` (best practice cloud-native, không hardcode URL):

```json
"GrpcEndpoints": {
  "DocumentService": "https://localhost:5170"
}
```

```csharp
var grpcAddress = _configuration["GrpcEndpoints:DocumentService"];
using var channel = GrpcChannel.ForAddress(grpcAddress);
var client = new DocumentService.DocumentServiceClient(channel);
var documents = await client.GetAllAsync(new PatientId { Id = patient.PatientId.ToString() });
```

Cách tiếp cận này tuân theo best practice cloud-native, cho phép cấu hình theo từng environment mà không cần sửa codebase, đảm bảo microservice luôn portable, scalable, và dễ triển khai qua các giai đoạn khác nhau của vòng đời phần mềm.

### Luồng giao tiếp gRPC — 4 bước

1. **Serialization**: client-side stub serialize request message thành định dạng nhị phân bằng Protobuf.
2. **Transmission**: message đã serialize được truyền qua mạng tới server, thường qua HTTP/2 — mang lại lợi ích độ trễ thấp hơn và hỗ trợ multiplexing request song song.
3. **Server processing**: server nhận message, deserialize, xử lý request, và chuẩn bị response.
4. **Response handling**: server serialize response message và gửi lại cho client, nơi stub deserialize thành object có thể sử dụng.

---

## Phần 8: Khi nào chọn REST, khi nào chọn gRPC?

| Tiêu chí | REST | gRPC |
|---|---|---|
| Đối tượng gọi | Trình duyệt, mobile app, client bên ngoài | Service-tới-service nội bộ |
| Định dạng | JSON (dễ đọc, dễ debug) | Protobuf nhị phân (nhanh, nhỏ gọn) |
| Giao thức | HTTP/1.1 (phổ biến hơn) | HTTP/2 (multiplexing, nhanh hơn) |
| Hợp đồng dữ liệu | Lỏng lẻo, dễ sai lệch nếu không kỷ luật | Chặt chẽ, contract-first (`.proto`) |
| Streaming | Không hỗ trợ tự nhiên | Hỗ trợ đầy đủ (server/client/bidirectional) |
| Đa ngôn ngữ | Tốt (ai cũng gọi được HTTP) | Rất tốt, tự sinh code nhiều ngôn ngữ |

---

## Tổng kết chương

Chương này đi sâu vào các sắc thái của giao tiếp trong kiến trúc microservices, tập trung vào sự cần thiết và cách triển khai giao tiếp đồng bộ — nơi một service yêu cầu dữ liệu từ service khác và chờ phản hồi ngay lập tức, tương tự lệnh gọi hàm truyền thống. Dù cách tiếp cận này đơn giản và có lợi cho việc lấy dữ liệu thời gian thực, nó cũng đưa vào những phức tạp như tight coupling, nghẽn hiệu năng tiềm ẩn, và lỗi dây chuyền nếu một service trở nên không phản hồi.

Chương cũng khám phá các công nghệ giao tiếp đồng bộ như HTTP/RESTful và gRPC, cùng vai trò của chúng trong việc xây dựng API vững chắc. HTTP, với sự đơn giản, khả năng mở rộng, và bản chất stateless, là nền tảng của giao tiếp web, phù hợp tốt với nguyên tắc REST. RESTful API tận dụng HTTP mang lại sự linh hoạt qua định dạng JSON, khả năng mở rộng nhờ caching, và tiêu thụ băng thông thấp. Chương hướng dẫn từng bước tạo một RESTful microservice trong .NET với ASP.NET Core, bao gồm tích hợp database, thiết lập controller, và phương pháp testing bằng công cụ như VS Code REST Client extension.

Sau đó, chương chuyển sang gRPC, nêu bật các ưu điểm so với REST: hỗ trợ HTTP/2 cho multiplexing, serialize nhị phân gọn nhẹ qua Protobuf, và các pattern giao tiếp nâng cao (streaming, giao tiếp hai chiều).

Chương tiếp theo sẽ khám phá giao tiếp bất đồng bộ giữa các microservice — cách xử lý các tiến trình không cần phản hồi ngay lập tức.
