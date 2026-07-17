# Bài học: Aggregator Pattern — Khi UI không nên "chạy vòng quanh" gọi từng service

> Chương 5 — Working with the Aggregator Pattern
> Ví dụ xuyên suốt: Blazor WebAssembly Dashboard bệnh nhân gọi tới AggregatorService, service này tổng hợp dữ liệu từ Patient, Appointment, Medical History, Billing service

---

## Phần 1: Vấn đề thực tế — UI đang phải làm việc của một "nhạc trưởng"

Khi tương tác giữa các service rời rạc trong ứng dụng microservices trở nên phức tạp, thách thức xử lý việc lấy và tổng hợp dữ liệu (data retrieval and aggregation) ngày càng phức tạp hơn.

Hãy hình dung kịch bản một client application cần thông tin trải dài qua nhiều service, như hồ sơ người dùng, lịch sử đơn hàng, và gợi ý sản phẩm. Không có chiến lược thống nhất, client sẽ cần khởi tạo các request riêng biệt tới từng service, quản lý response, và xử lý mọi lỗi phát sinh. Cách tiếp cận này tăng độ phức tạp của logic phía client và dẫn tới sự kém hiệu quả, bao gồm tăng độ trễ và network overhead.

Hệ thống quản lý phòng khám của chúng ta sẽ truy cập dữ liệu từ nhiều service với mỗi request UI. Hãy hình dung một UI cho phép bác sĩ lâm sàng truy cập hồ sơ toàn diện của bệnh nhân, bao gồm thông tin cá nhân, bệnh sử, thuốc đang dùng, lịch hẹn sắp tới, và chi tiết thanh toán. Mỗi phân đoạn dữ liệu này có thể nằm ở service riêng biệt:

- **Patient service**: quản lý thông tin cá nhân và nhân khẩu học
- **Medical history service**: lưu trữ hồ sơ y tế quá khứ và lịch sử điều trị
- **Appointment service**: xử lý lịch trình và chi tiết lịch hẹn
- **Billing service**: hóa đơn và thông tin thanh toán

Khi người dùng truy cập patient dashboard, UI khởi tạo request cho từng service. Các request này phải xảy ra song song hoặc tuần tự, tùy vào kiến trúc, để tránh trì hoãn. Ví dụ, luồng request mà UI thực hiện có thể trông như thế này:

1. Gửi GET request tới Patient Service API (`/api/patient/{id}`)
2. Gửi GET request tới Medical History Service API để lấy hồ sơ y tế của bệnh nhân (`/api/medical-history/{patientId}`)
3. Gửi GET request tới Appointments Service API để lấy lịch hẹn của bệnh nhân (`/api/appointments/{patientId}`)
4. Gửi GET request tới Billing Service API để lấy thông tin thanh toán của bệnh nhân (`/api/billing/{id}`)

Cũng cần nhớ rằng mỗi request có thể yêu cầu token xác thực, header, và query parameter cụ thể phù hợp với service đang truy cập. Pattern này gợi nhớ tới pattern giao tiếp đồng bộ đã khám phá ở chương trước. Do số lượng network roundtrip, mỗi cuộc gọi service đưa vào một độ trễ nhất định, và tất cả cộng dồn lại sẽ ảnh hưởng tới hiệu năng của UI.

### Ví dụ thực chiến: Blazor WebAssembly không có Aggregator

Ta dùng ứng dụng Blazor WebAssembly (Blazor WASM) để minh họa cách UI phải tương tác với nhiều service. Blazor là framework frontend hiện đại tận dụng HTML, CSS, và C# để tăng tốc phát triển ứng dụng web, cho phép tạo component tái sử dụng chạy trên cả client và server.

```bash
dotnet new blazorwasm -n HealthcareDashboard
cd HealthcareDashboard
dotnet add package Microsoft.Extensions.Http
```

Cấu hình endpoint trong `wwwroot/appsettings.json`:

```json
{
  "ApiEndpoints": {
    "AppointmentsApi": "https://appointments.api/",
    "PatientsApi": "https://patients.api/",
    "MedicalHistoryApi": "https://medicalhistory.api/",
    "BillingApi": "https://billingdetails.api/"
  }
}
```

Implement service client class cho từng microservice để thúc đẩy khả năng bảo trì, mở rộng, và module hóa:

```csharp
public record Patient(
    Guid PatientId,
    string FirstName,
    string LastName,
    DateTime DateOfBirth,
    string Gender,
    string ContactNumber,
    string Email
);

public class PatientServiceClient(HttpClient httpClient)
{
    private readonly HttpClient _httpClient = httpClient;

    public async Task<Patient?> GetPatientAsync(Guid patientId)
    {
        return await _httpClient.GetFromJsonAsync<Patient>($"/{patientId}");
    }
}
```

Đăng ký `HttpClient` cho từng service trong `Program.cs`:

```csharp
var config = builder.Configuration.GetSection("ApiEndpoints");
builder.Services.AddHttpClient<PatientServiceClient>(client =>
    client.BaseAddress = new Uri(config["ApiEndpoints:PatientsApi"]));
// Other service client registrations
```

Component `Dashboard.razor`:

```csharp
@page "/dashboard/{PatientId:Guid}"
@inject PatientServiceClient PatientService
@* Other services to inject *@

<h3>Patient Dashboard</h3>
@if (isLoading)
{
    <p>Loading data...</p>
}
else
{
    <div>
        <h4>Patient Full Name</h4>
        <p>@patient?.FirstName @patient?.LastName</p>
        <h4>Patient Profile</h4>
        <p><strong>Gender:</strong> @patient?.Gender</p>
        <p><strong>Email:</strong> @patient?.Email</p>
        <p><strong>Contact:</strong> @patient?.ContactNumber</p>
        @* Additional profile details *@
    </div>
}

@code {
    [Parameter]
    public Guid PatientId { get; set; }
    private Patient? patient;
    private bool isLoading = true;

    protected override async Task OnParametersSetAsync()
    {
        try
        {
            isLoading = true;
            patient = await PatientService.GetPatientAsync(PatientId);
            @* More synchronous service calls via service clients *@
        }
        catch (Exception ex)
        {
            throw;
        }
        finally
        {
            isLoading = false;
        }
    }
}
```

Khi tham số được phát hiện, method `OnParametersSetAsync` thực thi nhiều cuộc gọi API để lấy thông tin từ nhiều API. UI sẽ loading cho tới khi toàn bộ thao tác hoàn tất. Như đã đề cập, nếu một trong các service gặp vấn đề, toàn bộ UI sẽ bị đơ một lúc.

Một điều khác cần cân nhắc là UI có thể đang chịu trách nhiệm cho quá nhiều thứ. Ta cần cân nhắc UI nên xử lý thế nào nếu một trong các cuộc gọi service thất bại. Ta fail toàn bộ thao tác load hay hoàn tất một phần load?

Mối quan tâm chính khác là UI đang coupling chặt tới đâu với các service. Ta sẽ cần sửa nhiều khía cạnh của code UI với mỗi lần sửa đổi hoặc thêm service. Hãy hình dung nếu mỗi service cần header bổ sung và có yêu cầu bảo mật riêng.

Giới thiệu một Aggregator service có thể gánh vác các trách nhiệm này.

---

## Phần 2: Giới thiệu Aggregator Pattern

UI sẽ gọi aggregator để xử lý toàn bộ tương tác microservice, tổng hợp dữ liệu, và trả về một response thống nhất. Điều này đơn giản hóa logic UI, giảm độ trễ, và tập trung hóa xử lý lỗi, khiến hệ thống hiệu quả và dễ bảo trì hơn. Đây là bước đầu tiên trong việc implement Aggregator pattern, trừu tượng hóa độ phức tạp của tương tác đa service và cung cấp một kênh giao tiếp tinh gọn, hiệu quả.

Phương pháp này giảm số lượng tương tác client-server và tập trung hóa xử lý lỗi cũng như định dạng dữ liệu, dẫn tới hệ thống hiệu quả và thân thiện với người dùng hơn. UI gửi một request duy nhất tới aggregator service cho một patient dashboard tổng hợp. UI không cần biết endpoint, định dạng dữ liệu, hay cơ chế xác thực của từng service riêng lẻ.

### Analogy: Aggregator giống một trợ lý hành chính tổng hợp hồ sơ

Hãy hình dung một bác sĩ trưởng khoa cần xem hồ sơ đầy đủ của bệnh nhân trước ca mổ. Thay vì tự mình chạy đi hỏi từng phòng ban — phòng xét nghiệm, phòng dược, phòng tài chính — bác sĩ chỉ cần gọi cho một trợ lý hành chính. Trợ lý này sẽ tự đi hỏi tất cả các phòng ban cùng lúc, tổng hợp thành một bộ hồ sơ hoàn chỉnh, rồi đưa lại cho bác sĩ.

### Bốn nhóm lợi ích cụ thể

**Security benefits (Lợi ích bảo mật)**
- UI chỉ giao tiếp với aggregator, giảm thiểu tiếp xúc trực tiếp với microservice.
- Authentication và authorization có thể tập trung hóa, đảm bảo áp dụng nhất quán chính sách bảo mật.
- Credential hoặc token nhạy cảm cần thiết cho từng service được giấu khỏi UI.
- Ít endpoint API được truy cập trực tiếp từ UI hơn, giảm bề mặt tấn công tổng thể của hệ thống.
- Aggregator có thể sanitize và validate dữ liệu nhận từ microservice trước khi gửi tới UI.
- End-to-end encryption có thể được enforce cho luồng dữ liệu giữa UI, aggregator, và microservice.

**Optimization benefits (Lợi ích tối ưu hóa)**
- Giảm network overhead vì UI thực hiện ít request hơn tới các service khác nhau. Một request duy nhất tới aggregator service giảm số lượng network round-trip.
- Request tới microservice có thể được xử lý song song bởi aggregator, giảm thời gian chờ phản hồi.
- Caching có thể được implement trong aggregator, phục vụ dữ liệu lặp lại hoặc được yêu cầu thường xuyên nhanh hơn so với truy vấn từng service riêng lẻ.

**Failure management benefits (Lợi ích quản lý lỗi)**
- Aggregator có thể phát hiện lỗi ở microservice riêng lẻ và trả về dữ liệu một phần cho UI, đảm bảo ứng dụng vẫn hoạt động ngay cả khi một số service đang down. Vì aggregator chặn dữ liệu trước khi chuyển tiếp, logic chất lượng và sanitization bổ sung có thể được thêm vào để đánh giá và điều chỉnh sự không nhất quán trong dữ liệu.
- UI không cần xử lý lỗi từng service riêng lẻ. Aggregator tổng hợp xử lý lỗi và cung cấp response thống nhất với thông tin lỗi chi tiết khi cần.
- Cơ chế retry có thể được tập trung hóa trong aggregator để xử lý lỗi tạm thời.
- Aggregator cung cấp một điểm duy nhất để theo dõi và log request, lỗi, và metric hiệu năng, đơn giản hóa việc debug và ứng phó sự cố.

**Data transformation benefits (Lợi ích biến đổi dữ liệu)**
- Aggregator có thể biến đổi dữ liệu từ nhiều service thành một response cố kết phù hợp với nhu cầu của UI. Ví dụ: kết hợp response của patient, appointment, và billing service thành một object `PatientDashboard` đơn giản hóa việc render UI.
- Aggregator che chắn UI khỏi thay đổi trong schema của từng microservice. Nếu một microservice cập nhật API, aggregator xử lý sự thích ứng, duy trì tính nhất quán cho UI. Ví dụ: nếu Medical History service đổi một field từ `diagnosis` thành `condition`, aggregator dịch lại cho UI, tránh gián đoạn.
- Aggregator có thể thêm các field tính toán hoặc làm phong phú response với thông tin bổ sung. Ví dụ: thêm field `fullName` bằng cách nối `firstName` và `lastName` từ Patient service.

---

## Phần 3: Cái giá phải trả — Aggregator có thể trở thành điểm nghẽn mới

Dù đây là những lợi ích hấp dẫn của việc dùng aggregator, cần lưu ý rằng aggregator này có thể trở thành single point of failure. Nếu aggregator down, mọi client phụ thuộc vào nó sẽ mất quyền truy cập vào mọi nguồn dữ liệu, bất kể các nguồn đó có còn khỏe mạnh hay không.

Dù aggregator đưa vào rủi ro tiềm ẩn, nó mở ra khả năng mở rộng đáng kể khi được thiết kế đúng cách. Đây là một số yếu tố thiết kế cần cân nhắc:

- **Horizontal scaling**: dùng frontend load balancer phân phối đều traffic đến giữa nhiều node aggregator, tự động điều chỉnh theo sức khỏe và năng lực của instance. Định nghĩa quy tắc scaling động dựa trên metric thời gian thực: CPU usage, request latency, và queue depth. Điều này đảm bảo tầng aggregation phát triển để đáp ứng nhu cầu, sau đó co lại trong lúc ít traffic để tối ưu chi phí.

- **Asynchronous processing**: tận dụng message queue cho các thao tác chạy dài hoặc không nhạy cảm về độ trễ. Request được đưa vào queue và xử lý bởi worker pool, làm mượt các đợt tăng traffic và ngăn request storm áp đảo API downstream.

- **Sharding and partitioning**: nếu aggregator phải duy trì tracking in-flight hoặc trạng thái correlation, partition workload theo khách hàng, khu vực địa lý, hoặc loại API. Mỗi shard scale độc lập và giảm thiểu tranh chấp.

Bằng cách kết hợp redundancy, quản lý trạng thái cẩn thận, và kỹ thuật load-shedding thông minh, một API aggregator có thể tiến hóa từ một bottleneck tiềm ẩn thành lõi mạnh mẽ, có khả năng mở rộng cao của hệ thống phân tán.

---

## Phần 4: Triển khai thực tế — Xây dựng Aggregator Service trong .NET

### Bước 1: Khởi tạo project

Aggregator service sẽ đóng vai trò điểm truy cập duy nhất cho client UI, nghĩa là nó chỉ đơn giản là một API khác gọi tới các API khác. Các cuộc gọi này sẽ xảy ra đồng bộ.

```bash
dotnet new webapi -o AggregatorService
cd AggregatorService
dotnet add package Microsoft.Extensions.Http
dotnet add package Newtonsoft.Json
```

Cấu hình `appsettings.json`:

```json
{
  "ApiEndpoints": {
    "AppointmentsApi": "https://appointments.api/",
    "PatientsApi": "https://patients.api/",
    "MedicalHistoryApi": "https://medicalhistory.api/",
    "BillingApi": "https://billingdetails.api/"
  }
}
```

### Bước 2: Định nghĩa DTO tổng hợp

```csharp
namespace AggregatorService.Models;

public class PatientDashboard
{
    public PatientDto? Patient { get; set; }
    public List<AppointmentDto>? Appointments { get; set; }
    // Additional details
}

public class PatientDto
{
    public Guid Id { get; set; }
    public string Fullname { get; set; } = string.Empty;
    public DateTime DateOfBirth { get; set; }
    public string Gender { get; set; } = string.Empty;
    public string Address { get; set; } = string.Empty;
    public string ContactNumber { get; set; } = string.Empty;
}

public class AppointmentDto
{
    public Guid Id { get; set; }
    public DateTime AppointmentDate { get; set; }
    public string DoctorName { get; set; } = string.Empty;
}
```

### Bước 3: Service Client gọi tới service gốc

```csharp
public class AppointmentsServiceClient(HttpClient httpClient)
{
    private readonly HttpClient _httpClient = httpClient;

    public async Task<List<AppointmentDto>?> GetAppointmentsAsync(int patientId)
    {
        var appointments = await _httpClient.GetFromJsonAsync<List<AppointmentDto>?>(
            $"api/appointmentsforpatient/{patientId}");
        return appointments;
    }
}
```

Endpoint tương ứng cần bổ sung vào `AppointmentsController`:

```csharp
// GET: api/Appointments/GetAppointmentsByPatientId/5
[HttpGet("GetAppointmentsByPatientId/{patientId}")]
public async Task<ActionResult<IEnumerable<AppointmentByPatientId>>> GetAppointmentsByPatientId(Guid patientId)
{
    var appointments = await _context.Appointments
        .Where(a => a.PatientId == patientId)
        .Select(q => new AppointmentByPatientId(q.AppointmentId, string.Empty, q.Slot.Start))
        .ToListAsync();

    // Lấy thông tin bác sĩ cho mỗi lịch hẹn — chạy song song
    var tasks = appointments.Select(async appointment =>
    {
        var doctor = await _doctorsApiClient.GetDoctorAsync(appointment.AppointmentId);
        appointment.DoctorName = doctor.LastName;
    });
    await Task.WhenAll(tasks);

    return appointments;
}

public class AppointmentByPatientId(Guid appointmentId, string doctorName, DateTime date)
{
    public Guid AppointmentId { get; set; } = appointmentId;
    public string DoctorName { get; set; } = doctorName;
    public DateTime Date { get; set; } = date;
}
```

**Lưu ý kỹ thuật**: DTO này dùng `class` chứ không phải `record`, vì cần cho phép mutability sau khi khởi tạo — với `record`, giá trị không thể thay đổi sau khi object được tạo, nên cần cẩn trọng chọn đúng kiểu dữ liệu tùy theo mục đích sử dụng.

Đăng ký service client trong `Program.cs`:

```csharp
builder.Services.AddHttpClient<AppointmentsServiceClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiEndpoints:AppointmentsApi"]);
});
```

### Bước 4: Controller — điều phối các cuộc gọi song song

```csharp
[ApiController]
[Route("api/[controller]")]
public class AggregatorController(
    PatientServiceClient _patientService,
    AppointmentsServiceClient _appointmentService) : ControllerBase
{
    [HttpGet("patient-dashboard/{patientId}")]
    public async Task<ActionResult<PatientDashboard>> GetPatientDashboard(int patientId)
    {
        var patientTask = _patientService.GetPatientAsync(patientId);
        var appointmentsTask = _appointmentService.GetAppointmentsAsync(patientId);
        // Additional tasks

        await Task.WhenAll(patientTask, appointmentsTask /*, Additional tasks */);

        var patient = await patientTask;
        if (patient is null) return NotFound();
        var appointments = await appointmentsTask;

        var dashboardData = new PatientDashboard
        {
            Patient = patient,
            Appointments = appointments
            // Other property assignments
        };
        return Ok(dashboardData);
    }
}
```

Implementation `PatientServiceClient` — biến đổi dữ liệu ngay tại tầng aggregator (kết hợp `FirstName` + `LastName` thành `Fullname`):

```csharp
public record Patient(
    Guid PatientId,
    string FirstName,
    string LastName
    /* Other properties */
);

public class PatientServiceClient
{
    private readonly HttpClient _httpClient;

    public PatientServiceClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<PatientDto?> GetPatientAsync(int patientId)
    {
        var patient = await _httpClient.GetFromJsonAsync<Patient>($"api/patients/{patientId}");
        if (patient is null) return null;

        var patientDto = new PatientDto
        {
            Id = patient.PatientId,
            Fullname = $"{patient.FirstName} {patient.LastName}",
            // Other property assignments
        };
        return patientDto;
    }
}
```

### Bước 5: Kết nối UI với Aggregator Service

Cấu hình lại `appsettings.json` của UI — giờ chỉ cần địa chỉ của aggregator service:

```json
{
  "ApiEndpoints": {
    "AggregatorServiceApi": "https://AggregatorService.api/"
  }
}
```

Định nghĩa model tổng hợp phía UI:

```csharp
namespace HealthcareDashboard.UI.Models;

public class PatientDashboardModel
{
    public PatientDto? Patient { get; set; }
    public List<AppointmentDto>? Appointments { get; set; }
    // Additional properties
}

public class PatientDto
{
    public Guid Id { get; set; }
    public string Fullname { get; set; } = string.Empty;
    public DateTime DateOfBirth { get; set; }
    public string Gender { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string ContactNumber { get; set; } = string.Empty;
}

public class AppointmentDto
{
    public Guid Id { get; set; }
    public DateTime AppointmentDate { get; set; }
    public string Doctor { get; set; } = string.Empty;
}
```

Service client mới cho Aggregator Service:

```csharp
public class AggregatorServiceClient(HttpClient httpClient)
{
    private readonly HttpClient _httpClient = httpClient;

    public async Task<PatientDashboardModel?> GetPatientDashboardAsync(Guid patientId)
    {
        return await _httpClient.GetFromJsonAsync<PatientDashboardModel?>(
            $"/aggregator/patient-dashboard/{patientId}");
    }
}
```

Đăng ký trong `Program.cs` (có thể xóa các đăng ký service riêng lẻ cũ):

```csharp
builder.Services.AddHttpClient<AggregatorServiceClient>(client =>
    client.BaseAddress = new Uri(config["ApiEndpoints:AggregatorServiceApi"]));
```

Refactor component `Dashboard.razor` để chỉ gọi một service duy nhất:

```csharp
@page "/dashboard/{PatientId:Guid}"
@inject AggregatorServiceClient AggregatorService

<h3>Patient Dashboard</h3>
@if (isLoading)
{
    <p>Loading data...</p>
}
else if (patientProfile is null)
{
    <p>No data found for the patient</p>
}
else
{
    <div>
        <h4>Patient Full Name</h4>
        <p>@patientProfile?.Patient?.Fullname</p>
        <h4>Patient Profile</h4>
        <p><strong>Gender:</strong> @patientProfile?.Patient?.Gender</p>
        <p><strong>Email:</strong> @patientProfile?.Patient?.Email</p>
        <p><strong>Contact:</strong> @patientProfile?.Patient?.ContactNumber</p>
        <h4>Appointments</h4>
        <ul>
            @foreach (var appointment in patientProfile?.Appointments)
            {
                <li>@appointment.AppointmentDate.ToShortDateString() with @appointment.Doctor</li>
            }
        </ul>
    </div>
}

@code {
    [Parameter]
    public Guid PatientId { get; set; }
    private PatientDashboardModel? patientProfile = new PatientDashboardModel();
    private bool isLoading = true;

    protected override async Task OnParametersSetAsync()
    {
        try
        {
            isLoading = true;
            patientProfile = await AggregatorService.GetPatientDashboardAsync(PatientId);
        }
        catch (Exception ex)
        {
            throw;
        }
        finally
        {
            isLoading = false;
        }
    }
}
```

Giờ component chỉ cần thực hiện một cuộc gọi service duy nhất và chỉ cần thuộc tính cho một object trả về. Hoạt động này kết luận rằng giới thiệu một aggregator có thể dẫn tới giao tiếp UI-tới-microservice gọn gàng hơn và ít coupling hơn. Tuy nhiên, bản thân aggregator có thể trở nên phức tạp vì nó giờ phải gánh vác trách nhiệm điều phối các cuộc gọi service hiệu quả.

---

## Phần 5: Performance Optimization và Best Practices

Aggregator service được giới thiệu để tối ưu cách UI tương tác với microservice của ứng dụng. Ta không bao giờ được quên rằng hệ thống chỉ mạnh bằng mắt xích yếu nhất. Vì service mới này là mắt xích giữa UI và phần còn lại của các service, nó cần được xây dựng tốt và có khả năng xử lý nhiều kịch bản.

### 1. Asynchronous Processing

Lập trình bất đồng bộ cho phép chương trình thực hiện thao tác không chặn (non-blocking), cho phép nhiều tác vụ chạy đồng thời. Paradigm này có lợi khi xử lý các thao tác chờ đợi, như network request tới API.

Trong chương trình đồng bộ, các tác vụ được thực thi tuần tự. Với một tác vụ như cuộc gọi API, chương trình dừng thực thi cho tới khi cuộc gọi hoàn tất. Hành vi chặn này khiến chương trình rảnh rỗi trong lúc chờ phản hồi.

Lập trình bất đồng bộ cho phép chương trình khởi tạo một tác vụ, tiếp tục thực thi tác vụ khác trong lúc chờ, và tiếp tục tác vụ ban đầu khi phản hồi đã sẵn sàng. Điều này đạt được bằng các cấu trúc như `async`/`await` trong các ngôn ngữ lập trình hiện đại như C#.

Ta dùng điều này trong implementation `AggregatorController.cs`, nơi ta thực thi đồng thời các task cuộc gọi API, giảm đáng kể thời gian hoàn tất các cuộc gọi. Một thread duy nhất có thể xử lý hàng nghìn request đồng thời vì nó không bị chặn trong lúc chờ thao tác I/O.

### 2. Caching

Caching lưu trữ một bản sao dữ liệu trong tầng lưu trữ tạm thời (cache) để phục vụ request tương lai nhanh hơn. Thay vì lặp lại việc lấy cùng dữ liệu từ nguồn (VD: microservice), aggregator service lấy nó từ cache.

Caching hoạt động bằng cách phục vụ dữ liệu từ nguồn ở request đầu tiên. Khi dữ liệu được lấy và thêm vào cache store với thời gian hết hạn, cho các request tiếp theo, nếu dữ liệu cached chưa hết hạn, nó được dùng để hoàn tất request mà không cần gọi nguồn.

Phương pháp đơn giản để implement caching là in-memory caching, nơi dữ liệu được cache trong bộ nhớ của ứng dụng. Phương pháp này dễ mất (volatile), và dữ liệu sẽ mất khi ứng dụng restart, vì vậy không phải giải pháp dài hạn.

```csharp
[HttpGet("patient-dashboard/{patientId}")]
public async Task<ActionResult<PatientDashboard>> GetPatientDashboard(int patientId)
{
    var cacheKey = $"PatientDashboard_{patientId}";

    if (_cache.TryGetValue(cacheKey, out PatientDashboard cachedDashboard))
    {
        return Ok(cachedDashboard);
    }

    // Nếu không có trong cache, lấy dữ liệu từ microservice
    // Lưu dữ liệu vào cache với thời gian hết hạn
    _cache.Set(cacheKey, dashboard, TimeSpan.FromMinutes(5));
    return Ok(dashboard);
}
```

Kỹ thuật caching mạnh mẽ hơn liên quan tới distributed caching, nơi cache không dễ mất — sẽ được xem xét kỹ hơn ở chương sau về xây dựng microservices có khả năng phục hồi.

### 3. Compression

Compression là kỹ thuật giảm kích thước dữ liệu được truyền qua mạng. Nó cải thiện hiệu năng và hiệu quả của ứng dụng, đặc biệt là web service và API. Nén response payload cho phép ứng dụng tiết kiệm băng thông, giảm độ trễ, và tăng cường trải nghiệm người dùng tổng thể.

.NET cung cấp hỗ trợ built-in cho nhiều thuật toán nén:

- **Gzip**: thuật toán phổ biến cho ứng dụng web nhờ cân bằng giữa tỷ lệ nén và tốc độ
- **Brotli**: thuật toán mới hơn với tỷ lệ nén tốt hơn Gzip, đặc biệt cho tài nguyên tĩnh
- **Deflate**: thuật toán khác, tương tự Gzip nhưng implementation hơi khác

```bash
dotnet add package Microsoft.AspNetCore.ResponseCompression
```

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
});

var app = builder.Build();
app.UseResponseCompression();
app.MapControllers();
app.Run();
```

Điều này đặc biệt hữu ích cho aggregator service trả về payload lớn sau khi tổng hợp dữ liệu từ nhiều service. Sau khi implement compression, payload có thể giảm kích thước lên tới 200%.

`ResponseCompressionMiddleware` tuân theo pipeline sau nội bộ:

1. **Client request**: client gửi header `Accept-Encoding` chỉ rõ phương thức nén nào nó hỗ trợ (VD: gzip, br, deflate)
2. **Middleware match**: middleware kiểm tra MIME type của response có khớp danh sách cho phép không, header `Accept-Encoding` có hiện diện và được hỗ trợ không, HTTP method có đủ điều kiện không, có path/content type nào bị loại trừ không
3. **Compression provider selection**: `ResponseCompressionMiddleware` duyệt qua các compression provider đã cấu hình (theo thứ tự) và khớp encoding được hỗ trợ đầu tiên với `Accept-Encoding` của client
4. **Compression stream wrapping**: bọc stream `HttpResponse.Body` với compression stream tương ứng (`GZipStream`, `BrotliStream`, `DeflateStream`)
5. **Header injection**: set header response `Content-Encoding` (VD: gzip) và `Vary: Accept-Encoding` (quan trọng cho caching proxy)
6. **Response execution**: dữ liệu ghi vào response body được định tuyến qua compression stream trước khi gửi tới client

### 4. Error Handling

Xử lý lỗi là yếu tố then chốt để xây dựng hệ thống đáng tin cậy và hiệu quả, đặc biệt trong aggregator service. Service này tích hợp dữ liệu từ nhiều microservice, mỗi cái có thể gặp lỗi. Không có cơ chế xử lý lỗi vững chắc, một lỗi đơn lẻ có thể lan truyền tới Aggregator service, ảnh hưởng tới khả năng cung cấp kết quả cho client.

Graceful degradation đảm bảo khi một microservice lỗi, aggregator service vẫn có thể trả về kết quả một phần hoặc dữ liệu fallback, duy trì một mức độ chức năng nhất định. Các pattern nâng cao hơn để xử lý retry hoặc dừng thực thi lỗi một cách graceful bao gồm retry pattern và circuit breaker pattern, sẽ được thảo luận chi tiết ở chương sau.

Với cuộc gọi API, ta có thể mong đợi một trong ba loại lỗi:

- **Transient errors (Lỗi tạm thời)**: các vấn đề tạm thời như network timeout hoặc rate limiting. Thường có thể giải quyết bằng cách retry.
- **Permanent errors (Lỗi vĩnh viễn)**: các vấn đề như 404 Not Found hoặc 401 Unauthorized. Retry thường không cần thiết.
- **Unexpected errors (Lỗi không lường trước)**: các điều kiện không lường trước, như lỗi deserialize hoặc null reference.

Bất kể lý do chính của lỗi, một exception được throw khi `HttpClient` nhận response lỗi. Lỗi này có thể được catch, và một giá trị mặc định có thể được trả về. Bằng cách này, lỗi từ một service không ảnh hưởng tới các cuộc gọi service khác.

Quan trọng là thống nhất một phương pháp xử lý lỗi chuẩn với team và làm điều tốt nhất cho giải pháp. Có thể có trường hợp một lỗi nên chấm dứt toàn bộ tiến trình, và có lúc ta nên graceful và tiếp tục với các cuộc gọi service thiết yếu hơn, ngay cả khi một hoặc nhiều cuộc gọi thất bại.

### 5. Logging and Monitoring

Logging và monitoring là thực hành nền tảng để đảm bảo độ tin cậy, hiệu năng, và khả năng bảo trì của bất kỳ hệ thống phần mềm nào. Các thực hành này cho phép developer theo dõi hành vi hệ thống, phát hiện và chẩn đoán vấn đề, và đưa ra quyết định thông tin để tối ưu service.

Logging là thực hành ghi lại sự kiện và message hệ thống trong quá trình thực thi chương trình. Log cung cấp bản ghi theo trình tự thời gian của các thao tác, lỗi, và cảnh báo, mang lại insight giá trị về hành vi và hiệu năng của ứng dụng.

.NET cung cấp framework logging mạnh mẽ và có thể mở rộng, tích hợp trong namespace `Microsoft.Extensions.Logging`. Cơ chế built-in này hỗ trợ structured logging và nhiều logging provider, có thể tùy chỉnh cao, phù hợp cho ứng dụng ở mọi quy mô.

```csharp
ILogger<ClassName> logger
```

```csharp
logger.LogInformation("I am a log message");
```

Monitoring là việc quan sát liên tục hiệu năng, khả dụng, và sức khỏe của một hệ thống. Công cụ monitoring thu thập và phân tích metric, cho phép chủ động nhận diện và giải quyết vấn đề tiềm ẩn.

---

## Tổng kết chương

Chương này xem xét Aggregator pattern, một cách tiếp cận thiết kế quan trọng để quản lý tương tác giữa các microservice. Chương làm nổi bật các thách thức của giao tiếp trực tiếp client-tới-microservice, như tăng độ trễ, logic phía client phức tạp, và phụ thuộc coupling chặt, đặc biệt trong các hệ thống như ứng dụng y tế. Những vấn đề này được giảm thiểu bằng cách giới thiệu một aggregator service, và độ phức tạp được chuyển từ UI sang một service trung tâm, tinh gọn việc tổng hợp dữ liệu và tương tác.

Ta đã đi qua lý thuyết đằng sau pattern, cách triển khai thực tế với ASP.NET Core (bao gồm gọi song song bằng `Task.WhenAll`, biến đổi dữ liệu tại tầng aggregator), và các best practice quan trọng: xử lý bất đồng bộ, caching, compression, xử lý lỗi graceful degradation, và logging/monitoring tập trung.

Aggregator pattern cũng có các biến thể phổ biến — API Gateway (thêm routing, rate limiting, auth tập trung) và Backend for Frontend/BFF (aggregator riêng cho từng loại client) — đều dùng chung triết lý gom dữ liệu và giấu độ phức tạp bên dưới khỏi client.

Chương tiếp theo sẽ tiếp tục khám phá các pattern thiết kế khác giúp xây dựng hệ thống microservices vững chắc hơn.
