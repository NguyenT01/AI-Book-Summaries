# Bài học: Domain-Driven Design — Ngôn ngữ chung giữa Dev và Nghiệp vụ

> Chương 2 — Applying Domain-Driven Design Principles
> Ví dụ xuyên suốt: Hệ thống quản lý phòng khám (Healthcare Management System)

---

## Phần 1: Vấn đề DDD giải quyết — khoảng cách giữa Dev và Nghiệp vụ

Có một tình huống mà bất kỳ dev nào cũng từng gặp: bạn ngồi họp với chuyên gia nghiệp vụ (ví dụ, một y tá trưởng nếu bạn làm hệ thống bệnh viện), họ nói về "hồ sơ bệnh án", "lịch trực", "phác đồ điều trị" — còn trong đầu bạn đang nghĩ về class, interface, inheritance. Hai bên nói hai ngôn ngữ khác nhau, và mọi hiểu lầm nhỏ ở đây, khi build thành code, sẽ trở thành bug nghiệp vụ nghiêm trọng ở production.

**Domain-Driven Design (DDD)**, do Eric Evans đặt nền móng trong cuốn sách kinh điển năm 2003 *"Domain-Driven Design: Tackling Complexity in the Heart of Software"*, giải quyết đúng vấn đề này: thiết kế phần mềm sao cho nó phản chiếu chính xác cách nghiệp vụ đang vận hành, không phải cách lập trình viên nghĩ nghiệp vụ nên vận hành. DDD đòi hỏi sự cộng tác chặt chẽ giữa đội kỹ thuật và chuyên gia nghiệp vụ để xây dựng mô hình phản ánh đúng các quy trình và quy tắc kinh doanh phức tạp.

Khi không có DDD, hai hậu quả rất hay gặp:

- Hệ thống lệch khỏi mục tiêu nghiệp vụ, khó bảo trì, khó thích ứng khi nghiệp vụ thay đổi, dẫn đến giải pháp không đáp ứng đầy đủ yêu cầu người dùng.
- Team thiếu một phương pháp có cấu trúc để đóng gói business logic sẽ gặp khó khi thích ứng phần mềm với yêu cầu nghiệp vụ thay đổi liên tục, dẫn đến hệ thống "cứng", chống lại sự thay đổi.

Với microservices, DDD còn quan trọng hơn nữa vì nó nhấn mạnh việc gắn kiến trúc phần mềm với các domain nghiệp vụ — sự gắn kết này tạo điều kiện xây dựng các service cố kết (cohesive), có khả năng mở rộng (scalable) và dễ bảo trì (maintainable). Nói cách khác: **DDD chính là công cụ để trả lời câu hỏi "cắt service ở đâu"** một cách có hệ thống, thay vì cảm tính.

---

## Phần 2: Bốn khái niệm nền tảng — phải thuộc lòng

### 2.1. Ubiquitous Language (Ngôn ngữ chung)

Đây là ngôn ngữ đặc thù cho một domain model, được toàn bộ thành viên team sử dụng trong mọi hoạt động liên quan tới domain đó. Vì model của một domain cần được phát triển thông qua cộng tác với chuyên gia nghiệp vụ, và vì có sự khác biệt về kỹ năng/góc nhìn giữa hai bên, luôn tồn tại rào cản giao tiếp: dev quen nghĩ và nói về khái niệm lập trình (inheritance, polymorphism), trong khi chuyên gia nghiệp vụ không quan tâm tới điều đó — họ dùng jargon và thuật ngữ riêng mà dev không hiểu.

Ubiquitous Language chính là chiếc cầu nối. Nếu chuyên gia nghiệp vụ gọi là "Patient", code của bạn phải có class tên `Patient`, không phải `Customer` hay `User` chung chung. Ví dụ cụ thể: thay vì có một class `Customer` chung chung, ta tạo các đại diện cụ thể như `Patient` hoặc `Doctor`, mỗi cái có thuộc tính và hành vi riêng biệt.

### 2.2. Bounded Context (Ranh giới ngữ nghĩa)

Đây là khái niệm quan trọng nhất của cả chương. Bounded context định nghĩa ranh giới trong một hệ thống hoặc hệ thống con, xác định phạm vi công việc mà một team cụ thể sẽ đảm nhiệm và trọng tâm nỗ lực của họ trong domain đó. Đây là ranh giới logic của một domain, nơi các thuật ngữ và quy tắc được áp dụng.

Khi định nghĩa một bounded context, việc đánh dấu rõ ràng ranh giới của nó — cả trong code lẫn trong giao tiếp — là điều cực kỳ quan trọng. Sự phân tách này có thể được thể hiện bằng các module/microservice riêng biệt, hoặc bằng API/message contract được định nghĩa rõ ràng. Điều này khiến việc tích hợp giữa các context trở nên có chủ đích và có thể dự đoán được.

Ví dụ kinh điển: khái niệm "Patient" xuất hiện ở cả bounded context **Document Management** lẫn **Appointment Management** — hai context độc lập nhưng có chung một khái niệm được tham chiếu (shared concept). Điều này được biểu diễn qua **context mapping** — một chiến lược phổ biến trong DDD để mô tả quan hệ giữa các bounded context.

Việc thiết lập bounded context là bước chiến lược quan trọng trong DDD, được dùng để scoping các model lớn trong team đông người. Đây chính là nền tảng để bạn vẽ ranh giới microservice: **mỗi bounded context là một ứng viên để trở thành một service riêng**, có mô hình, ngôn ngữ, quy tắc nghiệp vụ và stack công nghệ của riêng nó.

### 2.3. Entity (Thực thể)

Một entity là khối xây dựng nền tảng đại diện cho một đối tượng riêng biệt trong domain, được đặc trưng bởi **identity duy nhất** và một **vòng đời (life cycle)**. Entity rất quan trọng để mô hình hóa các khái niệm cần được phân biệt, ngay cả khi thuộc tính của chúng giống hệt nhau.

Ví dụ: một `Patient` entity trong hệ thống quản lý bệnh viện. Mỗi bệnh nhân có một định danh duy nhất (patient ID) để phân biệt với người khác. Các thuộc tính như tên, địa chỉ, bệnh sử có thể thay đổi, nhưng identity của bệnh nhân luôn không đổi.

Code minh họa base entity trong C#:

```csharp
public abstract class BaseEntity<TId>
{
    public TId Id { get; set; }
}
```

Class được đánh dấu `abstract` để ngăn khởi tạo trực tiếp `BaseEntity`, và dùng generic type `TId` để linh hoạt kiểu định danh cho từng entity kế thừa.

**Về việc chọn kiểu định danh (Guid vs Integer)**: trong kiến trúc microservices, việc chọn đúng kiểu định danh cực kỳ quan trọng cho độ tin cậy và khả năng mở rộng của hệ thống:

- **Global uniqueness**: vì các service hoạt động độc lập và có thể sinh định danh đồng thời, GUID giảm thiểu rủi ro trùng định danh mà không cần một cơ chế điều phối tập trung.
- **Decentralized ID generation**: mỗi service tự sinh GUID cục bộ mà không cần phụ thuộc một nguồn trung tâm, tăng khả năng phục hồi và giảm bottleneck, giữ các service loosely coupled.
- **Simplified data merging**: khi cần gộp dữ liệu từ nhiều service, GUID đảm bảo định danh vẫn duy nhất trên toàn bộ tập dữ liệu.
- **Enhanced security**: GUID khó đoán hơn số nguyên tuần tự, gây khó khăn hơn cho kẻ tấn công muốn đoán định danh hợp lệ (chống enumeration attack).

Một điểm thiết kế rất thực tế: cùng một entity có thể có đại diện khác nhau ở các service khác nhau. Ví dụ, `Patient` trong service quản lý bệnh nhân chứa đầy đủ thuộc tính/hành vi, nhưng `Patient` trong service đặt lịch chỉ cần dữ liệu tối thiểu phục vụ đúng quy trình đặt lịch. Thuộc tính dữ liệu của entity luôn tương đối với yêu cầu của service/bounded context chứa nó.

Về navigation property trong Entity Framework: đôi khi tốt hơn nên **loại bỏ navigation object**, hy sinh một phần "phép màu" của EF, để giữ quyền kiểm soát và tính dự đoán được cao hơn cho cách các model tương tác. Bằng cách chỉ giữ foreign key reference, ta enforce tốt hơn quan hệ aggregate một chiều với các entity không phải root — đây là lý do ta chỉ tham chiếu `PrimaryDoctorId` mà không include navigation property `PrimaryDoctor`, vì `PrimaryDoctor` có thể là một entity ở service hoàn toàn khác.

### 2.4. Value Object (Đối tượng giá trị)

Value object là phần tử domain nền tảng được định nghĩa hoàn toàn bởi **thuộc tính (attributes)**, không phải bởi identity. Khác với entity, chúng không có định danh duy nhất và được coi là bằng nhau nếu tất cả thuộc tính khớp nhau. Chúng **bất biến (immutable)** — trạng thái không thể thay đổi sau khi tạo; mọi thay đổi tạo ra một instance mới. Chính vì vậy, value object có thể hoán đổi cho nhau khi chứa cùng giá trị — lý tưởng để đại diện cho các khái niệm như ngày tháng, tiền tệ, hoặc đơn vị đo lường.

**Ví dụ thực tế về tầm quan trọng của value object**: nói một bệnh nhân "nặng 50" là vô nghĩa nếu không có đơn vị đo — 50 lbs khác hoàn toàn 50 kg. Giá trị số và đơn vị phải được thay đổi cùng nhau, không bao giờ tách rời — đây là lý do value object nên được thiết kế sao cho đơn vị chỉ có thể thay đổi cùng lúc với giá trị số, không thể tự thay đổi riêng lẻ.

Với C# 10 trở lên, `record` type là công cụ lý tưởng cho value object nhờ value-based equality có sẵn — hai record được coi là bằng nhau nếu định nghĩa record giống nhau và mọi field có giá trị bằng nhau.

```csharp
public record TimeSlot(DateTime Start, DateTime End)
{
    public TimeSlot
    {
        if (End <= Start)
        {
            throw new ArgumentException("End time must be after start time.");
        }
    }
}
```

`TimeSlot` đóng gói giờ bắt đầu/kết thúc của một cuộc hẹn. Nhờ dùng record, ta có được value-based equality và tính bất biến miễn phí, đơn giản hóa việc so sánh và đảm bảo khung giờ luôn nhất quán một khi đã tạo.

```csharp
var slot1 = new TimeSlot(
    new DateTime(2024, 11, 23, 14, 0, 0),
    new DateTime(2024, 11, 23, 15, 0, 0)
);
var slot2 = new TimeSlot(
    new DateTime(2024, 11, 23, 14, 0, 0),
    new DateTime(2024, 11, 23, 15, 0, 0)
);

bool areSlotsEqual = slot1 == slot2; // True — so sánh theo giá trị
```

Nguyên tắc bắt buộc: giá trị phải được set qua constructor tại thời điểm khởi tạo, và constructor cũng phải thực hiện toàn bộ validation/invariant check. Dù chọn `record` hay `class`, trạng thái của value object không bao giờ được thay đổi sau khi tạo.

---

## Phần 3: Analogy để nhớ mãi mãi — Entity vs Value Object

Hãy nghĩ về **con người** và **tờ tiền**.

- **Entity giống con người**: mỗi người có một mã định danh duy nhất (CCCD/CMND). Dù bạn cắt tóc, tăng cân, đổi số điện thoại — bạn vẫn là chính bạn, vì cái CCCD đó không đổi. Identity là thứ tồn tại xuyên suốt vòng đời, bất kể thuộc tính thay đổi ra sao.

- **Value Object giống tờ tiền 100 nghìn đồng**: hai tờ 100k bất kỳ, dù là hai tờ giấy vật lý khác nhau, đều **có giá trị y hệt nhau** và **hoán đổi cho nhau được**. Không ai quan tâm "đây là tờ tiền nào" — chỉ quan tâm giá trị 100.000đ. Nếu bạn muốn "đổi" giá trị (ví dụ đổi sang tờ 200k), bạn không sửa tờ cũ — bạn cầm một tờ hoàn toàn mới.

---

## Phần 4: Rich Domain Model vs Anemic Domain Model

Đây là điểm phân biệt code viết theo tinh thần DDD thật sự với code "giả DDD".

**Anemic model**: class chỉ có property, hầu như không có logic/hành vi — logic được đặt ở tầng business logic bên ngoài model. Đây là kiểu lập trình theo phong cách thủ tục (procedural), thường dùng cho các entity con không có logic đặc biệt, đẩy hết behavior lên aggregate root hoặc business logic layer. Rủi ro: dễ dẫn tới spaghetti code, đánh mất lợi thế mà domain model mang lại.

```csharp
public class Patient : BaseEntity<int>
{
    public Patient(string name, string sex, int? primaryDoctorId = null)
    {
        Name = name;
        Sex = sex;
        PrimaryDoctorId = primaryDoctorId;
    }

    private Patient() { } // required for EF

    public string Name { get; private set; }
    public string Sex { get; private set; }
    public int? PrimaryDoctorId { get; private set; }

    public void UpdateName(string name)
    {
        Name = name;
    }
}
```

**Rich model**: class tự chứa method thể hiện hành vi nghiệp vụ và tự validate chính nó — không chỉ chứa dữ liệu.

```csharp
public class Appointment : BaseEntity<Guid>
{
    public Appointment(Guid id, int appointmentTypeId, Guid scheduleId,
        int doctorId, int patientId, int roomId,
        DateTime start, DateTime end, string title,
        DateTime? dateTimeConfirmed = null)
    {
        Id = id;
        AppointmentTypeId = appointmentTypeId;
        ScheduleId = scheduleId;
        DoctorId = doctorId;
        PatientId = patientId;
        RoomId = roomId;
        Start = start;
        End = end;
        Title = title;
        DateTimeConfirmed = dateTimeConfirmed;
    }

    public void UpdateRoom(int newRoomId)
    {
        if (newRoomId == RoomId) return;
        RoomId = newRoomId;
    }

    public void UpdateStartTime(DateTime newStartTime)
    {
        if (newStartTime == Start) return;
        Start = newStartTime;
    }

    public void Confirm(DateTime dateConfirmed)
    {
        if (DateTimeConfirmed.HasValue) return;
        DateTimeConfirmed = dateConfirmed;
    }
}
```

Không phải lúc nào rich model cũng tốt hơn. Với các service CRUD đơn giản, anemic model hoàn toàn hợp lý và tránh over-engineering — quyết định này phải dựa trên mục đích sử dụng và các thao tác thực tế của service, không phải "DDD là chuẩn nên lúc nào cũng phải rich model".

---

## Phần 5: Aggregate — người gác cổng bảo vệ tính nhất quán

Đây là khái niệm thực chiến nhất, và cũng dễ làm sai nhất với junior dev.

**Định nghĩa**: Aggregate là một cụm entity và value object liên quan, được xử lý như một khối thống nhất để duy trì tính nhất quán và enforce các quy tắc nghiệp vụ. Mỗi aggregate có một **aggregate root** — entity duy nhất là điểm truy cập từ bên ngoài. Mọi thay đổi bắt buộc đi qua root, đảm bảo domain invariant luôn được giữ vững và các thay đổi luôn atomic, nhất quán.

Aggregate giúp việc enforce các quy tắc dữ liệu/validation trên nhiều object trở nên dễ dàng hơn — ví dụ, một bệnh nhân có thể có nhiều địa chỉ, nhưng cần ít nhất một địa chỉ để ở trạng thái hợp lệ. Ràng buộc này dễ áp dụng hơn khi kiểm soát từ cấp root.

**Invariant** là quy tắc định nghĩa trạng thái hợp lệ của một domain model và phải luôn được enforce để đảm bảo tính đúng đắn nghiệp vụ. Ví dụ trong service đặt lịch khám: một invariant là "không thể đặt lịch trong quá khứ" — được enforce bằng cách từ chối yêu cầu đặt lịch có start time nhỏ hơn thời điểm hiện tại.

### Ví dụ: Patient Aggregate

- **Aggregate root**: `Patient` entity, định danh bởi patient ID duy nhất
- **Entities**: `MedicalRecord`, `Appointment`, `Prescription` gắn với bệnh nhân
- **Value objects**: `Address`, thông tin liên hệ, thông tin bảo hiểm

### Ví dụ: Appointment Aggregate

- **Aggregate root**: `Appointment` entity, định danh bởi `AppointmentId`
- **Entities**: `Patient` (người nhận khám), `Doctor` (bác sĩ khám)
- **Value objects**: `TimeSlot` (khung giờ), `Location` (địa điểm)

```csharp
public record TimeSlot(DateTime Start, DateTime End)
{
    public TimeSlot
    {
        if (End <= Start)
            throw new ArgumentException("End time must be after start time.");
    }
}

public record Location(string RoomNumber, string Building);

public class Appointment
{
    public Guid AppointmentId { get; }
    public Guid PatientId { get; }
    public Guid DoctorId { get; }
    public TimeSlot Slot { get; }
    public Location Location { get; }
    public string Purpose { get; private set; }

    public Appointment(Guid appointmentId, Guid patientId, Guid doctorId,
        TimeSlot slot, Location location, string purpose)
    {
        AppointmentId = appointmentId;
        PatientId = patientId;
        DoctorId = doctorId;
        Slot = slot;
        Location = location;
        Purpose = purpose;
    }

    public void Reschedule(TimeSlot newSlot)
    {
        // Business rules để check xung đột lịch, v.v.
        Slot = newSlot;
    }

    public void ChangePurpose(string newPurpose)
    {
        Purpose = newPurpose;
    }
}
```

Cách dùng trong code:

```csharp
var appointmentId = Guid.NewGuid();
var patientId = Guid.NewGuid();   // Lấy từ patient records
var doctorId = Guid.NewGuid();    // Lấy từ doctor records

var slot = new TimeSlot(
    new DateTime(2024, 11, 25, 10, 0, 0),
    new DateTime(2024, 11, 25, 11, 0, 0)
);
var location = new Location("Room 101", "Building A");

var appointment = new Appointment(appointmentId, patientId, doctorId,
    slot, location, "Routine Check-up");

// Đổi lịch
var newSlot = new TimeSlot(
    new DateTime(2024, 11, 26, 14, 0, 0),
    new DateTime(2024, 11, 26, 15, 0, 0)
);
appointment.Reschedule(newSlot);

// Đổi mục đích khám
appointment.ChangePurpose("Follow-up Consultation");
```

### Lợi ích của việc mô hình hóa theo aggregate

- **Consistency enforcement**: đóng gói entity và value object liên quan, aggregate đảm bảo mọi business rule/invariant được duy trì trong phạm vi ranh giới của nó.
- **Simplified data access**: component bên ngoài chỉ tương tác qua aggregate root, giảm độ phức tạp khi truy cập/thao tác dữ liệu.
- **Transaction management**: các thao tác thay đổi trạng thái aggregate được xử lý như một transaction duy nhất, đảm bảo tính atomic và nhất quán.

### Analogy: Aggregate giống một cái két sắt ngân hàng

Bạn không thể tự ý mở ngăn kéo bên trong két sắt và lấy đồ ra. Bạn phải đi qua **cửa chính** (aggregate root) — nơi có bảo vệ kiểm tra quy tắc trước khi cho phép bất kỳ thay đổi nào. Nếu ai đó có thể sửa trực tiếp field bên trong entity con mà không qua root, quy tắc nghiệp vụ sẽ bị phá vỡ mà không ai biết.

### Quan hệ trong aggregate — nguyên tắc một chiều

Sách nhấn mạnh rất rõ: dù quan hệ giữa các object có vẻ tự nhiên là hai chiều (bệnh nhân có lịch hẹn, và lịch hẹn cần biết bệnh nhân), DDD khuyến khích **quan hệ một chiều** để giảm coupling. `Appointment` có thể tham chiếu `Patient` qua ID, nhưng `Patient` không cần tham chiếu ngược lại `Appointment`. Cách này giữ cho aggregate tự chứa (self-contained) và đơn giản hóa tương tác.

Câu hỏi để tự kiểm tra khi thiết kế: **"Tôi có thể định nghĩa object này mà không cần object kia không?"** Nguyên tắc: aggregate nên luôn chảy một chiều từ root tới các dependent, không bao giờ ngược lại.

### Quan hệ vượt ranh giới aggregate

Khi quan hệ trải rộng qua nhiều aggregate, chiến lược xử lý bao gồm: tham chiếu aggregate khác bằng identity (không nhúng toàn bộ object), sử dụng domain event, điều phối qua application service, và chấp nhận **eventual consistency** — một chiến lược duy trì tính nhất quán dữ liệu giữa các bounded context/service mà không cần đồng bộ tức thời. Kỹ thuật này giúp tránh việc enforce tính nhất quán chặt chẽ theo thời gian thực trên toàn hệ thống — điều dễ dẫn tới tight coupling và nghẽn hiệu năng — thay vào đó các phần khác nhau của hệ thống được cập nhật bất đồng bộ, thường thông qua domain event.

---

## Phần 6: Cái giá phải trả — DDD không phải cây đũa thần

### Ưu điểm

- **Enhanced communication**: Ubiquitous Language giúp giao tiếp rõ ràng và nhất quán hơn giữa thành viên kỹ thuật và phi kỹ thuật, giảm hiểu lầm. DDD khuyến khích các buổi modeling chung như event-storming workshop, nơi dev và chuyên gia nghiệp vụ cùng khám phá domain.
- **Alignment with business objectives**: tập trung vào core domain giúp thiết kế phần mềm bám sát quy trình nghiệp vụ, giải quyết đúng yêu cầu người dùng.
- **Improved flexibility and maintainability**: cấu trúc module hóa giúp thích ứng dễ dàng hơn với yêu cầu nghiệp vụ thay đổi. Việc đóng gói business rule trong entity/aggregate giúp khoanh vùng thay đổi, giảm rủi ro tác dụng phụ ngoài ý muốn.
- **Cleaner code base**: khuyến khích dùng best practice và design pattern, chia hệ thống phức tạp thành các layer domain/application/infrastructure rõ ràng.

### Nhược điểm

- **Resource intensity**: đòi hỏi đầu tư thời gian và công sức đáng kể để hiểu domain, cần cộng tác chặt chẽ giữa dev và chuyên gia nghiệp vụ.
- **Complexity overhead**: có thể tạo ra độ phức tạp không cần thiết cho domain đơn giản — quá nhiều entity/value object/aggregate cần quản lý, thêm layer trừu tượng (repository, factory, service) có thể khiến hệ thống khó điều hướng, đặc biệt với thành viên mới. Bounded context và aggregate tuy giúp quản lý phức tạp nhưng cũng tạo ra chi phí tích hợp: team phải định nghĩa interface/API/message protocol rõ ràng giữa các context, dễ dẫn tới trùng lặp model và chi phí điều phối.
- **Requires domain expertise**: cần hiểu biết domain sâu, có thể cần đào tạo thêm cho team.
- **Not ideal for every project**: với dự án có độ phức tạp domain thấp hoặc độ phức tạp kỹ thuật cao hơn nghiệp vụ (tối ưu hiệu năng, tích hợp hệ thống, thuật toán phức tạp), DDD có thể gây phức tạp không cần thiết mà không mang lại lợi ích tương xứng.

---

## Phần 7: DDD và Microservices — hai mảnh ghép khớp nhau

Ứng dụng thiết kế theo nguyên tắc DDD được triển khai tốt nhất qua microservices. Kiến trúc microservices thúc đẩy việc chia một ứng dụng monolith thành các service độc lập, tự chứa — vận dụng ý tưởng ranh giới và context, mỗi microservice phục vụ đúng một bounded context của nó, với model, ngôn ngữ, quy tắc nghiệp vụ, và stack công nghệ riêng.

Tuy nhiên, đây không phải viên đạn bạc — sự khớp hoàn hảo giữa một microservice và một bounded context không phải lúc nào cũng chính xác tuyệt đối.

**Ví dụ về bounded context "ẩn"**: hệ thống gửi email/cảnh báo xác nhận lịch khám. Ban đầu, logic này có thể đặt ngay trong web application — khi lịch được xác nhận, gửi email cho bệnh nhân và cảnh báo cho nhân viên. Điều này hợp lý, nhưng ta cũng có thể tạo một service riêng dựa trên **message queue** chỉ để gửi các thông báo này. Cách này giúp web application giảm trách nhiệm, và giảm rủi ro vô tình chỉnh sửa UI logic khi xử lý vấn đề bảo trì email/alert.

Vì DDD khuyến khích tách các context thành component độc lập, kiến trúc microservices là mô hình kiến trúc hoàn hảo để hỗ trợ tham vọng này. DDD đóng vai trò như một hướng dẫn ban đầu để định nghĩa quy tắc nghiệp vụ với ranh giới rõ ràng và phụ thuộc tối thiểu — mỗi microservice được phát triển để hỗ trợ đúng tập quy tắc nghiệp vụ đó, theo cách hiệu quả và loosely coupled nhất.

---

## Tổng kết chương

DDD là một phương pháp phát triển phần mềm nhằm gắn kết hệ thống với các domain nghiệp vụ phức tạp, thông qua việc xây dựng model phản ánh chính xác domain, dựa trên sự cộng tác giữa đội kỹ thuật và chuyên gia nghiệp vụ. Chương này đã đi qua các khái niệm nền tảng: entity, model, ubiquitous language, bounded context, value object, aggregate — và cách chúng tương tác để xây dựng hệ thống thích ứng, dễ bảo trì, bám sát mục tiêu kinh doanh, đặc biệt phù hợp với các domain phức tạp như hệ thống quản lý y tế.

Chương tiếp theo sẽ khám phá giao tiếp đồng bộ (synchronous communication) giữa các microservice — cách các bounded context này thực sự "nói chuyện" với nhau trong một hệ thống phân tán.
