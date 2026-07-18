# Bài học: CQRS Pattern — Khi "đọc" và "ghi" không nên dùng chung một con đường

> Chương 6 — Working with the CQRS Pattern
> Ví dụ xuyên suốt: AppointmentsApi triển khai CQRS với MediatR — `CreateAppointmentCommand`, `GetAppointmentsQuery`, `GetAppointmentByIdQuery`

---

## Phần 1: Vấn đề thực tế — một model không thể phục vụ tốt cả hai nhu cầu trái ngược

Ta giờ đã biết rằng microservices đòi hỏi tầm nhìn xa trong giai đoạn lập kế hoạch, và ta phải áp dụng pattern và công nghệ tốt nhất để hỗ trợ quyết định của mình. Chương này khám phá một pattern khác đã nhận được nhiều lời khen ngợi vì giúp viết code sạch và dễ bảo trì: pattern **Command Query Responsibility Segregation (CQRS)**, một phần mở rộng của pattern CQS.

CQRS là nguyên tắc kiến trúc nền tảng đóng vai trò then chốt trong thiết kế và triển khai microservices hiện đại. Về cốt lõi, CQRS chia trách nhiệm đọc và ghi dữ liệu thành hai đường riêng biệt, cho phép developer tối ưu từng đường độc lập. Sự phân tách này cho phép khả năng mở rộng, bảo trì, và hiệu năng tốt hơn trong các ứng dụng nơi business logic phức tạp và yêu cầu throughput cao là tối quan trọng.

Trong kiến trúc monolithic hoặc layered truyền thống, cùng một data model thường được dùng cho cả việc lấy dữ liệu (query) và sửa đổi dữ liệu (command). Dù cách tiếp cận này hoạt động tốt cho ứng dụng đơn giản hơn, nó có thể dẫn tới sự kém hiệu quả và cứng nhắc khi hệ thống trở nên phức tạp. Ví dụ, một số kịch bản cụ thể có thể cần các query phức tạp dẫn tới over-fetching hoặc under-fetching dữ liệu, trong khi các kịch bản khác đòi hỏi cập nhật trạng thái hệ thống chính xác, atomic. Điều này có nghĩa là một thao tác update hoặc write không nên ảnh hưởng tới toàn bộ các điểm dữ liệu của một bảng và nên được thực hiện theo cách cụ thể, hiệu quả. Cố gắng tối ưu một model duy nhất cho các nhu cầu đa dạng này có thể nhanh chóng dẫn tới thiết kế cồng kềnh, nặng nề.

Tại điểm này, ta bắt đầu dùng các từ như hành vi (behavior) và kịch bản (scenario). Ta bắt đầu cân nhắc cấu trúc code theo cách cho phép cô lập hành vi và nhanh chóng xác định liệu hành vi này chỉ đơn thuần là yêu cầu dữ liệu, hay sẽ tăng cường dữ liệu vào cuối thao tác.

### Analogy: Nhà hàng có một quầy phục vụ cả gọi món lẫn thanh toán

Hãy hình dung một nhà hàng chỉ có một quầy duy nhất vừa nhận order món ăn, vừa xử lý thanh toán, vừa trả lời câu hỏi "món này có gì". Giờ cao điểm, quầy này bị quá tải: người chờ thanh toán phải xếp hàng chung với người đang hỏi menu, và nhân viên phải liên tục chuyển đổi ngữ cảnh giữa "ghi order mới" và "chỉ cần trả lời thông tin". CQRS giống việc tách nhà hàng đó ra: một quầy order/thanh toán riêng (command — thay đổi trạng thái) và một màn hình tra cứu menu điện tử riêng (query — chỉ đọc, không thay đổi gì).

---

## Phần 2: Từ CQS tới CQRS — hiểu nguồn gốc để hiểu bản chất

CQRS pattern bắt nguồn từ pattern CQS, viết tắt của Command-Query Segregation. Pattern này phát biểu rằng mọi method nên thực hiện một hành động (command) hoặc trả về dữ liệu (query), nhưng không bao giờ cả hai. Nguyên tắc này đặt nền móng cho codebase sạch hơn và dễ đoán hơn bằng cách đảm bảo method có một trách nhiệm duy nhất. Tuy nhiên, CQS chủ yếu tập trung vào cải thiện thiết kế ở cấp độ code hơn là giải quyết mối quan tâm kiến trúc trong hệ thống quy mô lớn.

Nhìn kỹ hơn, có thể lập luận rằng CQS chỉ tính đến một data store duy nhất, nghĩa là ta thực hiện thao tác đọc/ghi trên cùng một database.

CQRS mở rộng nguyên tắc CQS lên cấp độ kiến trúc, giải quyết các thách thức đặt ra bởi việc dùng một model duy nhất cho cả query và command. Một model dùng chung thường dẫn tới sự kém hiệu quả trong kiến trúc monolithic hoặc layered truyền thống khi hệ thống scale. Query phức tạp có thể cần over-fetching hoặc under-fetching dữ liệu, trong khi command đòi hỏi thay đổi trạng thái chính xác, atomic. Tối ưu một model duy nhất cho các yêu cầu đa dạng này thường dẫn tới thiết kế nặng nề và dễ vỡ.

Khái niệm over-fetching và under-fetching liên quan tới lượng dữ liệu được xử lý trong một request. Thực hành tốt trong truy vấn database tiêu chuẩn là khi muốn lấy dữ liệu, ta nên cố lấy đúng dữ liệu cần thiết thay vì lấy quá nhiều hoặc quá ít cho ngữ cảnh. Với thực hành tốt này, ta muốn bắt đầu điều này ở cấp độ phần mềm và lấy đúng dữ liệu cần để hoàn tất một request. Lấy nhiều dữ liệu hơn cần thiết đưa vào độ trễ không cần thiết và có thể dẫn tới việc xử lý dữ liệu quá mức.

CQRS pattern giải quyết thách thức này bằng cách giới thiệu sự phân tách rõ ràng các mối quan tâm. Command chịu trách nhiệm cho các hành động thay đổi trạng thái hệ thống, như tạo, cập nhật, hoặc xóa tài nguyên. Query, ngược lại, xử lý việc lấy dữ liệu, cung cấp thông tin cần thiết ở định dạng hiệu quả nhất có thể. Sự phân tách này cho phép developer tùy chỉnh model cơ bản và hạ tầng cho yêu cầu riêng biệt của từng thao tác.

### Kiến trúc CQRS — 2 mô hình

CQRS gợi ý có các data store riêng biệt, một cho thao tác ghi và một cho thao tác đọc. Ví dụ, write model có thể dùng database transactional được tối ưu cho tính nhất quán, do tuân thủ nghiêm ngặt các nguyên tắc Atomicity, Consistency, Isolation, và Durability (ACID). Ngược lại, read model có thể tận dụng cấu trúc denormalized, cache-friendly được thiết kế cho tốc độ, như cache (Redis, Memcached, v.v.) hoặc data store NoSQL (MongoDB, Cosmos DB, v.v.). Việc implement nhiều data store không phải lúc nào cũng là một tùy chọn, cũng không phải là bắt buộc.

CQRS đã nhận được nhiều lời khen ngợi kể từ khi được giới thiệu trong phát triển phần mềm và được ca ngợi như một trụ cột của thiết kế microservice. Tuy nhiên, nó không phải là duy nhất hoặc bắt buộc cho microservices. Nó cũng thêm một tầng phức tạp mới cho nỗ lực phát triển, đòi hỏi nhiều class cụ thể hơn và code phải được viết. Điều này có thể dẫn tới dự án phình to, tăng độ phức tạp code, và overhead trong nỗ lực phát triển.

---

## Phần 3: Ba lợi ích cốt lõi của CQRS

CQRS là về việc chia một model duy nhất thành hai biến thể, một cho đọc và một cho ghi. Mục tiêu cuối cùng, tuy nhiên, là một kiến trúc hệ thống được cải thiện.

### 1. Scalability (Khả năng mở rộng)

Đây là lợi ích đầu tiên của cách tiếp cận này. Đánh giá workload đọc/ghi của hệ thống là điều thiết yếu. Thao tác đọc thường tốn nhiều tài nguyên hơn thao tác ghi vì một lần đọc có thể cần dữ liệu từ nhiều bảng, dẫn tới join và filter bổ sung, làm giảm tốc độ và hiệu quả của việc lấy dữ liệu. Mỗi request có thể có yêu cầu riêng dựa trên hình dạng của dữ liệu. Một trường phái tư duy khuyến khích dùng một data store dành riêng và được tối ưu cho thao tác đọc. Điều này cho phép ta scale thao tác đọc riêng biệt với thao tác ghi.

Ví dụ về một read store dành riêng là data warehouse, nơi một dạng pipeline biến đổi dữ liệu liên tục chuyển đổi dữ liệu từ write data store, một database quan hệ đã chuẩn hóa. Một công nghệ thường được dùng khác là database NoSQL như MongoDB hoặc Cosmos DB. Các construct dữ liệu do data warehouse hoặc database NoSQL cung cấp đại diện cho một phiên bản dữ liệu đã denormalized, chỉ đọc, từ data store quan hệ.

### 2. Performance (Hiệu năng)

Lợi ích thứ hai là hiệu năng. Dù nó song hành cùng khả năng mở rộng, có những động lực khác nhau mà ta cân nhắc. Dùng data store riêng biệt không phải lúc nào cũng là một tùy chọn khả thi, vì vậy có các kỹ thuật khác ta có thể dùng để tối ưu thao tác mà không thể thực hiện được với một unified data model. Ví dụ, ta có thể áp dụng caching cho các query liên quan tới thao tác đọc. Ta cũng có thể dùng các tính năng đặc thù của database và câu lệnh SQL được tinh chỉnh riêng cho các request của mình.

### 3. Simplicity (Tính đơn giản)

Một lợi ích khác ta thu được từ pattern này là tính đơn giản. Điều này nghe có vẻ mâu thuẫn, vì ta đã đề cập tới sự phức tạp trước đó trong chương, nhưng nó phụ thuộc vào góc nhìn ta dùng để đánh giá phần thưởng ta sẽ thu được về lâu dài. Command và query có nhu cầu khác nhau, và dùng một data model để phục vụ cả hai nhu cầu là không hợp lý. CQRS buộc ta phải cân nhắc tạo data model cụ thể cho từng query hoặc command, dẫn tới code dễ bảo trì hơn. Mỗi data model mới chịu trách nhiệm cho một thao tác, và sửa đổi trong đó sẽ có ít hoặc không có tác động tới các khía cạnh khác của chương trình.

Có câu nói rằng khi bạn có một cái búa, mọi thứ trông giống một cái đinh; khi nghe về một pattern mới, bạn có thể cảm thấy cần phải dùng nó ở mọi nơi. Hãy cân nhắc pattern này cẩn thận trước khi dùng và đảm bảo giá trị của nó trong ứng dụng được chứng minh.

---

## Phần 4: Nhược điểm — CQRS không phải viên đạn bạc

Ta đã thấy rằng dùng CQRS trong dự án có một số lợi ích. Nhưng ở đâu có ưu điểm, ở đó có nhược điểm. CQRS giới thiệu nhiều thách thức cần được cân nhắc cẩn thận.

Mọi pattern ta cân nhắc phải được điều tra kỹ lưỡng về giá trị và nhược điểm tiềm ẩn. Ta luôn muốn đảm bảo lợi ích vượt trội hơn nhược điểm và ta sẽ không hối tiếc về những quyết định thiết kế quan trọng này. **CQRS không lý tưởng cho ứng dụng chỉ thực hiện thao tác CRUD đơn giản.**

Như đã nêu, nó là behavior-driven hoặc scenario-driven, vì vậy bạn có thể chứng minh use case trước khi bắt tay vào. Pattern này dựa trên khái niệm thực hiện một command hoặc hoàn thành một query, và trên thực tế, nó sẽ dẫn tới cảm giác trùng lặp code vì nó thúc đẩy ý tưởng phân tách các mối quan tâm và khuyến khích command/query có model dành riêng.

### Câu chuyện thực tế đáng suy ngẫm

Có những trường hợp các development lead với ý định tốt nhất đã implement CQRS trong các dự án không cần nó. Dự án bị phức tạp hóa quá mức, gây bối rối cho developer mới về việc mọi thứ nằm ở đâu.

### Vấn đề data consistency khi có nhiều data store

Như đã thấy, nhiều khu vực lưu trữ dữ liệu được biện minh khi dùng CQRS, có thể xem là implementation hoàn chỉnh nhất của nó. Khi ta giới thiệu nhiều data store, ta giới thiệu các vấn đề về tính nhất quán dữ liệu. Ta phải implement kỹ thuật event-sourcing và bao gồm service-level agreement để thông báo cho người dùng về khoảng cách tiềm ẩn giữa thao tác đọc/ghi của ta.

### Chi phí hạ tầng và vận hành

Ta cũng phải cân nhắc chi phí hạ tầng bổ sung và chi phí vận hành chung khi có nhiều database. Với nhiều database khác nhau đi kèm nhiều điểm lỗi tiềm ẩn và nhu cầu giám sát bổ sung cũng như kỹ thuật fail-safe để đảm bảo hệ thống vận hành trơn tru nhất có thể, ngay cả khi đối mặt với sự cố.

CQRS có thể mang lại giá trị cho một dự án khi được áp dụng hợp lý. Khi dự án của bạn có business logic phức tạp hoặc nhu cầu rõ ràng để tách biệt data store và thao tác đọc/ghi, CQRS sẽ tỏa sáng như một lựa chọn kiến trúc phù hợp.

Pattern này không riêng cho bất kỳ ngôn ngữ lập trình nào, và .NET có hỗ trợ tuyệt vời.

---

## Phần 5: Đánh giá service nào nên/không nên dùng CQRS

Đánh giá nhu cầu và đặc điểm cụ thể của service là điều thiết yếu trước khi implement CQRS pattern. Implement CQRS một cách bừa bãi có thể đưa vào độ phức tạp không cần thiết mà không mang lại lợi ích đáng kể. Để minh họa điều này, hãy xem xét hệ thống quản lý phòng khám ta đã phát triển. Hệ thống này bao gồm microservice quản lý dữ liệu bệnh nhân và bác sĩ, tài liệu, lịch hẹn, thanh toán, và bệnh sử.

Đánh giá service của bạn đảm bảo bạn áp dụng CQRS ở nơi nó mang lại giá trị nhiều nhất. Các yếu tố cần cân nhắc bao gồm độ phức tạp của business logic, tần suất và bản chất của thao tác đọc/ghi, và yêu cầu về khả năng mở rộng của service. Bằng cách đánh giá cẩn thận từng service trong kiến trúc, bạn có thể tránh over-engineering trong khi vẫn thu được lợi ích của CQRS ở đúng ngữ cảnh.

### Ứng viên tốt cho CQRS

Các service như Appointment booking, Patient Record Management, và Medical History là ứng viên mạnh cho CQRS pattern:

- **Appointment booking service** có thể cần xử lý cập nhật liên tục về tình trạng lịch trống, đồng thời cung cấp cho người dùng view real-time, được tối ưu về các khung giờ khả dụng.
- **Patient record management service** phải đảm bảo tính nhất quán khi sửa đổi thông tin y tế nhạy cảm, đồng thời cho phép truy xuất hiệu quả lịch sử bệnh nhân.
- **Medical history service** hưởng lợi từ CQRS bằng cách tách biệt việc tổng hợp dữ liệu real-time khỏi yêu cầu báo cáo intensive-query, cho phép nó scale độc lập cho việc xử lý dữ liệu hiệu năng cao.

### Không nên áp dụng CQRS

Ngược lại, các service xử lý authentication, notification, và document delivery không có khả năng hưởng lợi đáng kể từ CQRS. Các service này thường liên quan tới thao tác đơn giản với nhu cầu tối thiểu về query hoặc thay đổi trạng thái phức tạp:

- **Authentication service** xử lý login và validate token, vốn là thao tác atomic không cần model command và query riêng biệt.
- **Notification service** tập trung vào gửi message dựa trên template định sẵn.
- **Document delivery service** cung cấp file không liên quan tới logic lấy hoặc sửa đổi dữ liệu phức tạp.

Cho implementation của ta, ta sẽ tập trung áp dụng pattern này cho appointment booking service, nơi ta sẽ tăng cường việc dùng application logic và trừu tượng hóa logic khỏi controller.

---

## Phần 6: Triển khai thực tế — MediatR + CQRS trong .NET

### Vì sao chọn MediatR

Cách phổ biến và có thể nói là dễ dàng nhất để implement CQRS pattern trong .NET là MediatR. Thư viện nhẹ này tạo điều kiện cho in-process messaging và thường được dùng để implement CQRS pattern trong ứng dụng .NET. Nó thúc đẩy sự tách biệt sạch sẽ các mối quan tâm bằng cách decouple handler khỏi bên gọi.

Một số tính năng chính của package này bao gồm:

- Đơn giản hóa việc xử lý command và query bằng cách tận dụng mediator design pattern
- Giảm phụ thuộc giữa các application layer
- Hỗ trợ pipeline behavior cho cross-cutting concern như logging và validation

### Mediator Design Pattern — nền tảng lý thuyết

Mediator design pattern là behavioral design pattern tạo điều kiện giao tiếp giữa các thành phần hoặc object khác nhau trong hệ thống mà không cần chúng có phụ thuộc trực tiếp lẫn nhau. Thay vào đó, các thành phần này (colleague) tương tác qua một mediator object trung tâm, đóng gói logic giao tiếp. Pattern này giúp giảm coupling giữa các object, khiến hệ thống module hóa và dễ bảo trì hơn.

### Cài đặt và đăng ký

```bash
dotnet add package MediatR
```

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly())
);
```

Thêm dòng code này sẽ cho phép .NET Core nhận diện model MediatR và implementation của business logic hoặc handler trong code.

**Lưu ý thực chiến**: nếu project đã dùng MassTransit (như đã học ở Chapter 4), cần cẩn thận vì có sự trùng lặp tên interface giữa hai thư viện — phải đảm bảo dùng đúng MediatR ở các class nơi interface MediatR đang được inject.

### Tạo Command — CreateAppointmentCommand

Như bạn nhớ, một command dự kiến thực hiện các hành động sẽ sửa đổi dữ liệu trong data store, hay còn gọi là thao tác ghi. Vì ta dùng mediator pattern để điều phối các thao tác, ta cần một command model và một handler.

Model của ta tương đối đơn giản để implement. Nó thường là một class hoặc record tiêu chuẩn, nhưng với MediatR hiện diện, nó sẽ implement một type mới gọi là `IRequest`. `IRequest` sẽ liên kết model class này với một handler tương ứng.

```csharp
public record CreateAppointmentCommand(
    Guid DoctorId, Guid PatientId, Location Location,
    TimeSlot Slot, string Purpose) : IRequest<Appointment>;
```

Ta dùng kiểu record trong ví dụ này, nhưng nó cũng có thể là class nếu đó là điều bạn muốn dùng. Chú ý rằng ta kế thừa từ `IRequest<Appointment>`, điều này nói cho MediatR biết:

- Command này nên được liên kết với một handler có kiểu trả về tương ứng
- Handler liên kết với command này dự kiến trả về một giá trị object `Appointment`

Command không phải lúc nào cũng cần trả về một type. Bạn không cần chỉ định kiểu dữ liệu đi kèm với `IRequest` nếu không mong đợi kiểu trả về.

### Tạo Handler cho Command

Bây giờ ta có một command model, hãy implement handler tương ứng. Handler của ta là nơi logic để thực hiện tác vụ thường nằm ở đó. Trong một số implementation, và không có rich data model, bạn có thể thực hiện tất cả validation và tác vụ bổ sung cần thiết để đảm bảo tác vụ được xử lý đúng cách. Bằng cách này, ta có thể cô lập tốt hơn business logic mong đợi khi một command cụ thể được gửi để thực thi.

```csharp
public class CreateAppointmentHandler(
    AppointmentContext _context,
    IPublishEndpoint _publishEndpoint) :
    IRequestHandler<CreateAppointmentCommand, Appointment>
{
    public async Task<Appointment> Handle(CreateAppointmentCommand request, CancellationToken cancellationToken)
    {
        var newAppointment = new Appointment
        {
            AppointmentId = Guid.NewGuid(),
            DoctorId = request.DoctorId,
            PatientId = request.PatientId,
            Location = request.Location,
            Slot = request.Slot,
            Purpose = request.Purpose
        };

        _context.Add(newAppointment);
        await _context.SaveChangesAsync();

        // Perform post-creation hand-off to services bus.
        await _publishEndpoint.Publish<AppointmentCreated>(new
        {
            newAppointment.AppointmentId,
            newAppointment.PatientId,
            newAppointment.DoctorId,
            newAppointment.Slot.Start,
            DateTime.UtcNow,
            MessageId = newAppointment.AppointmentId
        });

        return newAppointment;
    }
}
```

Ta inject database context và publisher service từ API controller ban đầu bằng constructor mặc định. Ta cũng kế thừa từ `IRequestHandler`, implement một method gọi là `Handle`, mặc định trả về theo kiểu được chỉ ra bởi `IRequest<>` gắn với command model. Nó sẽ tự động sinh ra một tham số gọi là `request`, có kiểu dữ liệu của command model.

Ở đây, ta thực hiện:

- Lấy dữ liệu từ request object được gửi qua request command object và dùng nó để điền vào object `newAppointment`
- Tạo bản ghi appointment mới trong database
- Thêm object `AppointmentCreated` cho thao tác database và service bus
- Trả về object `newAppointment`, khớp với kiểu dữ liệu chỉ ra bởi kế thừa `IRequest<Appointment>` trên command

Lưu ý rằng nếu ta không cần giá trị trả về, ta sẽ phải trả về `Unit.Value`. Đây là biểu diễn void mặc định của MediatR.

### Refactor Controller để dùng Mediator

Bây giờ ta có một command tạo appointment mới, hãy refactor controller để dùng mediator và không còn implement toàn bộ business logic. Đầu tiên, ta cần inject `IMediator` để thêm dependency cho MediatR:

```csharp
public class AppointmentsController(
    IMediator _mediator, /* other dependencies */):
    ControllerBase
```

Sau đó, ta có thể refactor method POST để trông như thế này:

```csharp
public async Task<ActionResult<Appointment>> PostAppointment(CreateAppointmentCommand createAppointmentCommand)
{
    var appointment = await _mediator.Send(createAppointmentCommand);
    return CreatedAtAction("GetAppointment", new { id = appointment.AppointmentId }, appointment);
}
```

Bây giờ, ta có thể dựa vào handler để hoàn tất toàn bộ chức năng cần thiết để tạo một appointment, và API controller không còn cần chi tiết implementation. Bạn có thể quan sát rằng mỗi cách viết code đều dẫn tới cùng một kết quả: ta cần tạo model type phù hợp với giá trị đúng trước khi gửi nó tới mediator. Mediator sẽ đã xác định model nào khớp với handler nào và tiến hành thực hiện thao tác của nó.

---

## Phần 7: Tạo Query

Command và query nhìn chung tuân theo implementation tương tự. Một query dự kiến tìm kiếm dữ liệu và trả về kết quả. Việc tìm kiếm này có thể phức tạp hoặc đơn giản. Tuy nhiên, ta implement pattern này như một cách dễ dàng để tách biệt logic query khỏi nơi khởi tạo request (như controller) và logic command.

Loại tách biệt này tăng khả năng của team trong việc bảo trì mỗi phía của ứng dụng mà không có xung đột quá mức. Ta sẽ tương tự dùng mediator pattern để điều phối cách ta thực hiện các thao tác này, và ta sẽ cần một query model và một handler.

Model của ta đơn giản vì ta có thể để nó trống hoặc bao gồm property đóng vai trò trong tiến trình được thực hiện trong handler. Ta kế thừa `IRequest<>` và định nghĩa kiểu trả về vì ta đang viết một query và mong đợi dữ liệu được trả về.

### Hai ví dụ Query

```csharp
public record GetAppointmentsQuery(): IRequest<List<Appointment>>;
```

```csharp
public record GetAppointmentByIdQuery(string Id): IRequest<AppointmentDetails>;
```

Model `GetAppointmentsQuery` không cần property nào vì nó chỉ được dùng như một khung để handler tạo liên kết. `GetAppointmentByIdQuery` có property `Id` vì ID sẽ cần thiết để handler thực thi đúng tác vụ. Sự khác biệt trong kiểu trả về cũng là điểm quan trọng cần lưu ý, vì nó thiết lập tông cho những gì handler sẽ có thể trả về.

Ta cần đảm bảo query model của mình được tạo ra cụ thể cho loại dữ liệu ta mong đợi lấy từ handler tương ứng.

### Handler cho Query

Handler query của ta sẽ thực thi query mong đợi và trả về dữ liệu theo định nghĩa của `IRequest<>`. Như đã nêu trước đó, cách dùng lý tưởng của pattern này sẽ thấy ta dùng một data store riêng biệt nơi dữ liệu được query đã được tối ưu sẵn để trả về. Điều này sẽ làm cho thao tác query của ta hiệu quả và giảm nhu cầu biến đổi và sanitize dữ liệu. Một ORM thay thế cho Entity Framework Core (EF Core), như Dapper, đôi khi có thể cung cấp thời gian phản hồi query nhanh hơn đáng kể.

Trong ví dụ của ta, ta không implement một data store riêng biệt cũng không thay đổi ORM, vì vậy ta sẽ inject database context vào handler cho query lấy tất cả appointment. Handler này sẽ được gọi là `GetAppointmentsHandler.cs`:

```csharp
public class GetAppointmentsHandler(AppointmentContext _context) :
    IRequestHandler<GetAppointmentsQuery, List<Appointment>>
{
    public async Task<List<Appointment>> Handle(
        GetAppointmentsQuery request, CancellationToken cancellationToken)
    {
        return await _context.Appointments.ToListAsync();
    }
}
```

Định nghĩa của handler này đơn giản: ta lấy danh sách appointment. Khi ta cần một appointment theo ID, điều đó ngụ ý ta cần chi tiết của appointment. Điều này sẽ đòi hỏi một query phức tạp hơn liên quan tới join và/hoặc cuộc gọi API đồng bộ để lấy chi tiết của các bản ghi liên quan. Ta có thể "xử lý" các thao tác này trong handler mới, `GetAppointmentByIdHandler`:

```csharp
public class GetAppointmentByIdHandler(AppointmentContext _context,
    PatientsApiClient _patientsApiClient,
    DoctorsApiClient _doctorsApiClient) :
    IRequestHandler<GetAppointmentByIdQuery, AppointmentDetails>
{
    public async Task<AppointmentDetails> Handle(
        GetAppointmentByIdQuery request, CancellationToken cancellationToken)
    {
        var appointment = await _context.Appointments.FindAsync(request.Id);
        if (appointment == null)
        {
            return null;
        }
        // Use the Http clients to fetch patient and doctor details
        // Use GRPC service to retrieve document information on patient
        // Combine data and return response
        return appointmentDetails;
    }
}
```

Đây là phiên bản súc tích của implementation handler vì phần lớn code được lấy từ action `GetAppointment` trong file `AppointmentsController.cs`. Chú ý rằng các service liên quan được inject cùng database context, và toàn bộ logic phức tạp cần thiết để trả về chi tiết được trừu tượng hóa khỏi controller.

### Refactor Controller cho Query

```csharp
public async Task<ActionResult<IEnumerable<Appointment>>> GetAppointments()
{
    return await _mediator.Send(new GetAppointmentsQuery());
}

public async Task<ActionResult<Appointment>> GetAppointment(string id)
{
    var appointmentDetails = await _mediator.Send(
        new GetAppointmentByIdQuery(id));
    return appointmentDetails == null ? NotFound() : Ok(appointmentDetails);
}
```

Lưu ý rằng handler `GetAppointmentById` có thể trả về null nếu không tìm thấy bản ghi. Vì lý do này, ta kiểm tra xem giá trị trả về có null không và trả về HTTP response phù hợp.

Giờ bạn đã có nền tảng cần thiết để implement CQRS trong API của mình. Ngoài việc cô lập thao tác đọc và ghi, một lợi ích cụ thể khác là nó dẫn tới controller sạch hơn và mạch lạc hơn. Càng trừu tượng hóa business logic khỏi tầng API, code càng sạch hơn và càng dễ cô lập và test business logic.

---

## Phần 8: Best Practices và khuyến nghị

Best practice thường là vấn đề về triết lý và sở thích. Phát triển phần mềm nhắm tới việc cung cấp phần mềm hoạt động, nhưng điều đó thường phải trả giá bằng khả năng đọc và test code. Pattern này, như nhiều pattern khác, tìm cách tăng khả năng test của code, nhưng nó có thể phải trả giá bằng khả năng đọc nếu không được thực hiện đúng cách.

Điều đầu tiên ta muốn xem xét là naming convention. Tiêu chuẩn đặt tên tốt có thể giúp bạn và team khi implement pattern này.

### 1. Naming Convention — quy tắc đặt tên nhất quán

Naming convention là tập hợp quy tắc hoặc hướng dẫn chuẩn hóa cho việc đặt tên các phần tử trong code, như class, method, biến, và file. Các convention này nhằm cải thiện khả năng đọc, bảo trì, và chất lượng tổng thể của codebase.

Khi implement CQRS pattern, naming convention rõ ràng và nhất quán trở nên tối quan trọng do sự tách biệt giữa command và query. Đặt tên kém có thể dẫn tới nhầm lẫn, lỗi, và giảm năng suất. Đây là những lý do chính vì sao naming convention tốt quan trọng:

- Naming convention tốt khiến ngay lập tức rõ ràng liệu một class, method, hoặc file cụ thể thuộc phía command hay query của CQRS. Ví dụ, dùng động từ mệnh lệnh để chỉ hành động (VD: `CreateAppointmentCommand`, `UpdatePatientCommand`, và `GetPatientsQuery`).
- Developer có thể nhanh chóng hiểu vai trò và mục đích của các phần tử code mà không cần tài liệu hoặc khám phá sâu rộng.
- Các phần tử được đặt tên nhất quán dễ tìm và sửa đổi hơn khi cần thay đổi.
- Naming convention giúp căn chỉnh code với khái niệm domain, khiến code trực quan hơn và phản ánh yêu cầu nghiệp vụ.

Bằng cách tuân thủ naming convention tốt, bạn tạo nền tảng cho implementation CQRS pattern dễ bảo trì, có khả năng mở rộng, và thân thiện với developer.

### 2. Tổ chức file và class

CQRS pattern nhấn mạnh việc tách biệt trách nhiệm của command và query, và tổ chức đúng đắn file và class là điều thiết yếu để duy trì sự tách biệt này. Kết hợp với Single Responsibility Principle (SRP), tổ chức tốt đảm bảo mỗi component trong kiến trúc CQRS có mục đích rõ ràng và tập trung. Bạn có thể dùng nhiều kỹ thuật để hỗ trợ tổ chức file và folder:

**Tách folder command và query riêng biệt**: tạo folder hoặc namespace riêng biệt cho command và query để củng cố sự tách biệt trách nhiệm. Như trong implementation trước đó, ta tạo một folder cho command và một cho query, với subfolder chứa class handler và model cho thao tác.

**Group theo feature**: tổ chức file theo feature hoặc domain thay vì loại kỹ thuật (VD: command, query, và handler). Cách tiếp cận này căn chỉnh với domain-driven design và giúp dễ điều hướng codebase hơn. Nó còn được gọi là **vertical slice architecture**:

```
Features/
  Appointments/
    Commands/
      CreateAppointmentCommand.cs
      CreateAppointmentHandler.cs
    Queries/
      GetAppointmentByIdQuery.cs
      GetAppointmentByIdHandler.cs
```

**Tuân thủ SRP**: SRP là trường phái tư duy thúc đẩy sự tách biệt và độc lập rõ ràng cho các khối code. Một method nên có một tác vụ duy nhất, và một class nên implement một điều duy nhất. Điều này có nghĩa là, theo các quy tắc này, bạn nên tránh đặt nhiều class definition trong một file class.

SRP là chữ S trong SOLID, và là trụ cột của thực hành phát triển tốt. Nhưng khi implement CQRS, khi dự án lớn lên, việc điều hướng nhiều file như vậy trở nên khó khăn. Đôi khi, bạn có thể thấy rằng model, handler, và đôi khi cả class DTO đều được định nghĩa trong một file. Điều này giảm số lượng file được tạo và cho phép bạn thấy toàn bộ asset liên quan ở một nơi. Đây là ví dụ về implementation này nếu ta refactor code method `GetAppointmentsByPatientId` sang CQRS:

```csharp
public class AppointmentByPatientId(Guid appointmentId,
    string doctorName, DateTime date)
{
    // Shortened for brevity
}

public record GetAppointmentByPatientIdQuery(string PatientId):
    IRequest<List<AppointmentByPatientId>>;

public class GetAppointmentsByPatientIdhandler(
    AppointmentContext _context, DoctorsApiClient _doctorsApiClient) :
    IRequestHandler<GetAppointmentByPatientIdQuery,
    List<AppointmentByPatientId>>
{
    public async Task<List<AppointmentByPatientId>> Handle(
        GetAppointmentByPatientIdQuery request, CancellationToken cancellationToken)
    {
        // Get appointments for a patient
        // Get doctor details for each appointment in parallel
        return appointments;
    }
}
```

Lưu ý rằng DTO, request model, và handler đều được định nghĩa trong cùng một file class. Điều này giảm bớt nỗ lực điều hướng khi cần thay đổi một hoặc nhiều khối code này. Bạn và team có thể quyết định cách tiếp cận nào phù hợp nhất.

---

## Phần 9: Tăng tốc thao tác đọc

Vì CQRS tách biệt đọc và ghi, tối ưu thao tác đọc là điều then chốt để cải thiện hiệu năng của ứng dụng nặng về query. Đây là các kỹ thuật cụ thể để đạt được hiệu năng đọc nhanh hơn:

### 1. Dùng EF Core No-Tracking Queries

Mặc định, EF Core theo dõi thay đổi của entity lấy từ database, điều này gây overhead hiệu năng. Dùng `AsNoTracking()` để thực thi query chỉ đọc không cần tracking. Điều này giảm memory usage và tăng tốc thực thi query cho thao tác chỉ đọc:

```csharp
await _context.Appointments.AsNoTracking().ToListAsync();
```

### 2. Secondary (Denormalized) Data Stores

Ta đã dùng SQLite, một công nghệ database quan hệ nhẹ. Database quan hệ nổi tiếng về tổ chức và khả năng đảm bảo tính toàn vẹn dữ liệu, nhưng thường phải đánh đổi hiệu năng đọc. Trong trường hợp ta có nhiều service cần cung cấp chi tiết riêng lẻ (như lấy chi tiết appointment), ta phải cân nhắc một phiên bản hợp nhất của dữ liệu trong một store riêng biệt. Sự hợp nhất này được gọi là denormalization.

Cân nhắc tạo data model được tối ưu cho đọc, denormalized, được thiết kế riêng cho các use case query cụ thể. Cân nhắc dùng database, table, hoặc materialized view riêng biệt để pre-aggregate và lưu trữ dữ liệu ở định dạng được tối ưu cho việc query. Điều này có thể giảm đáng kể chi phí tính toán của join, cuộc gọi API, và aggregation lúc runtime. Cách tiếp cận này cũng đưa vào nhu cầu code đồng bộ và job cần chạy nền để đảm bảo read-only data store phản ánh trạng thái hiện tại của dữ liệu. Đây là một trong những yếu tố chính mà implementation này phải đối mặt, và nó thêm một tầng phức tạp nữa cho dự án.

### 3. Dùng ADO.NET

Như đã đề cập trước đó, đây là ORM thay thế thường được dùng cho EF Core flagship. EF Core cực kỳ performant và cải thiện với mỗi phiên bản mới. Nó vẫn là một abstraction layer với thao tác materialization và một số chi phí hiệu năng. ADO.NET vẫn là một phần cơ bản của hệ sinh thái .NET, cung cấp API mức thấp cho tương tác database. Nó cung cấp tính năng truy cập dữ liệu mạnh mẽ, linh hoạt, và hiệu năng cao, vẫn được hỗ trợ cho các kịch bản cần kiểm soát chi tiết đối với thao tác database. Nó cho phép bạn thực thi raw SQL query trực tiếp, mang lại kiểm soát hoàn toàn đối với SQL được gửi tới database:

```csharp
var appointments = new List<Appointment>();
using (var connection = new SqlConnection(
    _context.Database.GetDbConnection().ConnectionString))
{
    await connection.OpenAsync(cancellationToken);
    using (var command = new SqlCommand("SELECT * FROM Appointments", connection))
    {
        using (var reader = await command.ExecuteReaderAsync(cancellationToken))
        {
            while (await reader.ReadAsync(cancellationToken))
            {
                var appointment = new Appointment
                {
                    Id = reader.GetInt32(reader.GetOrdinal("Id")),
                    // Map other properties as needed
                };
                appointments.Add(appointment);
            }
        }
    }
}
```

Dù ADO.NET mạnh mẽ, nó có thể liên quan tới nhiều boilerplate code hơn EF Core. Dùng ORM cho phần lớn thao tác và ADO.NET cho các tối ưu hiệu năng quan trọng có thể là cách tiếp cận cân bằng cho dự án lớn. Nếu bạn muốn trải nghiệm hiện đại hóa hoàn toàn hoặc abstraction bổ sung, cân nhắc kết hợp ADO.NET với thư viện như Dapper, thêm khả năng micro-ORM nhẹ mà không hy sinh sự kiểm soát.

### 4. Dùng Dapper

Dapper là micro-ORM cho .NET cung cấp truy cập dữ liệu hiệu năng cao. Nó đặc biệt phù hợp cho phía query của CQRS, nơi có thể cần join hoặc projection phức tạp. Nó nhẹ và nhanh, lý tưởng cho các kịch bản có query nhạy cảm về hiệu năng. Nó cho phép thực thi trực tiếp SQL query và cung cấp kiểm soát chi tiết đối với tương tác database. Nó cũng đòi hỏi phải viết SQL query nhưng cung cấp một số khả năng mapping và projection dữ liệu, khiến nó dễ dùng hơn một chút so với ADO.NET:

```csharp
using (var connection = new SqlConnection(
    _context.Database.GetDbConnection().ConnectionString))
{
    await connection.OpenAsync(cancellationToken);
    var query = "SELECT * FROM Appointments";
    var appointments = await connection.QueryAsync<Appointment>(query);
    return appointments.ToList();
}
```

Dapper thường được xem là điểm cân bằng giữa ADO.NET và EF Core, mang lại sự kết hợp giữa hiệu năng và dễ dùng. Nó trừu tượng hóa phần lớn code lặp lại cần thiết trong ADO.NET và tự động map row database sang object .NET. Nó cũng có ít abstraction layer hơn EF Core, khiến nó performant hơn.

Việc chọn giữa Dapper, ADO.NET, và EF Core phụ thuộc vào yêu cầu cụ thể của dự án, như hiệu năng, khả năng bảo trì, và mức độ quen thuộc của team với SQL hoặc công cụ ORM.

CQRS là pattern không nên được xem nhẹ. Nó đưa vào nhiều lợi thế nhưng phù hợp hơn cho các dự án phức tạp hơn và team phát triển giàu kinh nghiệm.

---

## Tổng kết chương

Chương này thảo luận về CQRS pattern và implementation, lợi ích, và nhược điểm của nó, đặc biệt trong kiến trúc microservices. Về cốt lõi, CQRS chia trách nhiệm hệ thống thành các đường riêng biệt cho đọc (query) và ghi (command). Sự tách biệt này tăng cường khả năng mở rộng, bảo trì, và hiệu năng trong hệ thống có business logic phức tạp.

Trong microservices, CQRS căn chỉnh với các nguyên tắc về tính tự trị và khả năng mở rộng. Nó cho phép service scale độc lập thao tác đọc và ghi, và tối ưu cơ chế lưu trữ và truy xuất dữ liệu cho các use case cụ thể. Sự linh hoạt này giải quyết sự kém hiệu quả vốn có trong hệ thống monolithic truyền thống, nơi một model duy nhất được dùng cho cả query và command.

Ta đã triển khai CQRS thực tế với MediatR trong .NET, xây dựng command (`CreateAppointmentCommand`) và query (`GetAppointmentsQuery`, `GetAppointmentByIdQuery`) cho appointment booking service, giúp controller trở nên mỏng và tập trung, business logic được cô lập trong handler dễ test. Ta cũng học được các best practice quan trọng: naming convention rõ ràng, tổ chức file/class theo folder kỹ thuật hoặc vertical slice architecture, và bốn kỹ thuật tăng tốc thao tác đọc: no-tracking query, denormalized data store, ADO.NET, và Dapper.

Bài học cốt lõi: CQRS mang lại giá trị lớn khi áp dụng đúng ngữ cảnh — cho service có business logic phức tạp và nhu cầu tách biệt rõ ràng giữa đọc/ghi — nhưng cần tránh áp dụng bừa bãi cho các service chỉ cần CRUD đơn giản.
