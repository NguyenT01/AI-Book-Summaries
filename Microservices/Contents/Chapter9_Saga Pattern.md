# Bài học: Saga Pattern — Khi transaction không thể "tất cả hoặc không gì" nữa

> Chương 9 — Implementing Transactions across Microservices Using the Saga Pattern
> Ví dụ xuyên suốt: Luồng đăng ký bệnh nhân + đặt lịch hẹn — AppointmentsApi và PaymentsApi phối hợp qua MassTransit + RabbitMQ, dùng cả choreography lẫn orchestration (Saga State Machine)

---

## Phần 1: Vấn đề thực tế — một thao tác nghiệp vụ, bốn service, bốn database

Chương trước khám phá chiến lược thiết kế database và tầm quan trọng của chúng trong việc đảm bảo tính nhất quán dữ liệu, khả năng mở rộng, và khả năng bảo trì trong kiến trúc microservices. Ta đã xem xét nhiều kỹ thuật: shared database, database per service, CQRS, và event sourcing — đều thiết yếu để quản lý dữ liệu trong hệ phân tán. Dù các pattern này giúp thiết lập nền tảng vững chắc để xử lý dữ liệu microservices, chúng cũng đưa vào một thách thức then chốt: đảm bảo tính nhất quán dữ liệu xuyên suốt nhiều service khi thực thi một business transaction.

Ứng dụng monolith truyền thống dựa vào ACID transaction, nơi một database duy nhất đảm bảo transaction hoặc hoàn tất toàn bộ hoặc rollback toàn bộ. Tuy nhiên, việc implement transaction atomic như vậy trở nên phức tạp trong hệ thống dựa trên microservices, nơi mỗi service có database riêng. Một thao tác nghiệp vụ đơn lẻ có thể liên quan tới nhiều microservice, và nếu bất kỳ bước nào thất bại, hệ thống phải bù trừ (compensate) cho lỗi đó hoặc rollback thay đổi để duy trì tính nhất quán.

Đây chính là lúc **Saga pattern** xuất hiện. Pattern này quản lý distributed transaction mà không dựa vào một database transaction duy nhất. Thay vì đảm bảo ACID, nó đảm bảo tính nhất quán thông qua các compensating transaction được điều phối trên nhiều service.

### Analogy: Saga giống một chuỗi domino có khả năng "tự sửa chữa"

Hãy hình dung một chuỗi domino: đẩy đổ quân đầu tiên sẽ kích hoạt quân tiếp theo, cứ thế tiếp diễn. Nhưng khác với domino thật (một khi đổ là đổ luôn), saga giống một chuỗi domino thông minh: nếu quân domino thứ 3 không đổ đúng cách (thất bại), toàn bộ các quân domino đã đổ trước đó (quân 1, 2) sẽ tự động "dựng lại" theo đúng thứ tự ngược — đây chính là cơ chế compensating transaction (giao dịch bù trừ).

### Định nghĩa Saga

Như đã thảo luận trước đó, ứng dụng microservices tốt nhất nên implement theo pattern Database-Per-Service. Nhược điểm lớn của lựa chọn pattern này là ta không thể đảm bảo database của mình luôn đồng bộ nếu một trong các thao tác thất bại.

Ta cần một cơ chế trải dài qua nhiều service và có thể implement transaction xuyên suốt các data store khác nhau, như two-phase commit — đòi hỏi mọi data store phải commit hoặc rollback đồng thời. Điều này sẽ hoàn hảo, ngoại trừ việc một số NoSQL database và message broker không tương thích với mô hình này.

Hãy hình dung một bệnh nhân mới được đăng ký với trung tâm chăm sóc sức khỏe của ta. Quy trình này đòi hỏi bệnh nhân cung cấp thông tin của họ, một số tài liệu thiết yếu, và đặt lịch hẹn đầu tiên — điều này cần thanh toán. Các hành động này đòi hỏi bốn microservice khác nhau tham gia; do đó, bốn data store khác nhau sẽ bị ảnh hưởng.

Ta có thể gọi một thao tác trải dài qua nhiều service là một **saga**. Về bản chất, saga là một chuỗi các local transaction. Mỗi transaction cập nhật database mục tiêu và sinh ra một message hoặc event kích hoạt thao tác tiếp theo trong saga. Nếu một trong các local transaction trong chuỗi thất bại, saga sẽ thực thi rollback trên các database bị ảnh hưởng bởi các transaction trước đó.

### Ba loại transaction trong một saga

- **Reversible (Có thể đảo ngược)**: đây là các transaction có thể bị đảo ngược bởi một transaction khác có hiệu ứng ngược lại.
- **Retriable (Có thể thử lại)**: các transaction này được đảm bảo sẽ thành công và được implement sau pivot transaction.
- **Pivot (Bản lề)**: đúng như tên gọi, thành công hay thất bại của các transaction này mang tính quyết định cho việc saga có tiếp tục hay không. Nếu transaction commit, saga chạy tới khi hoàn tất. Các transaction này có thể được đặt như compensating transaction cuối cùng hoặc retriable transaction đầu tiên của saga. Chúng cũng có thể được implement như không thuộc loại nào cả.

---

## Phần 2: Issues and Considerations — Cái giá phải trả

Vì cho tới chương này, ta đã loại bỏ khả năng implement ACID transaction xuyên suốt data store trong kiến trúc microservices, pattern này không dễ implement. Nó đòi hỏi sự phối hợp tuyệt đối và hiểu biết về các bộ phận chuyển động của ứng dụng.

Pattern này cũng khó debug. Vì ta đang implement một function duy nhất trải dài qua các service tự trị, ta đã giới thiệu một điểm lỗi mới và một điểm chạm mới, và cần nỗ lực đặc biệt để track và trace nơi lỗi có thể xảy ra. Độ phức tạp này tăng theo từng bước có service tham gia vào saga.

Ta cần đảm bảo saga xử lý được lỗi tạm thời trong kiến trúc. Đây là các lỗi xảy ra trong lúc thao tác và có thể không phải vĩnh viễn. Nhắc lại từ các chương trước về nhu cầu đảm bảo idempotency. Mọi thao tác được bảo vệ bởi retry logic nên đảm bảo thay đổi nó kích hoạt chỉ xảy ra đúng một lần. Đây là phần thiết yếu của việc implement transaction và retry. Bất kể thế nào, ta phải bao gồm retry logic để đảm bảo một lỗi đơn lẻ trong một lần thử không kết thúc saga sớm. Khi làm vậy, ta cũng phải đảm bảo dữ liệu của mình nhất quán với mỗi lần retry.

Pattern này chắc chắn không thiếu thách thức. Dù Saga pattern mang lại cách thực dụng để điều phối business workflow trong hệ phân tán, chính điểm mạnh của nó cũng đưa vào một tập hợp nhược điểm cấu trúc và vận hành mà mọi team cần cân nhắc trong trade-off thiết kế:

- Pattern này từ bỏ sự thoải mái của ACID guarantee để đổi lấy eventual consistency. Điều đó có nghĩa mọi service downstream phải được thiết kế để chịu đựng sự phân kỳ tạm thời và trình bày trạng thái mạch lạc cho người dùng có thể đọc dữ liệu trước khi saga hội tụ hoàn toàn.
- Nó đòi hỏi logic bổ sung, như versioning, read-model isolation, hoặc compensating flag, để ngăn người dùng hành động dựa trên thông tin lỗi thời, và đòi hỏi giao tiếp rõ ràng về trạng thái "đang xử lý" ở tầng API.
- Nó tăng độ phức tạp tổng thể. Mỗi local transaction phải idempotent và có khả năng tự hoàn tác, hoặc thực thi một compensation riêng biệt mà không làm hỏng dữ liệu lịch sử. Saga càng nhiều bước, càng nhiều tổ hợp thành công, retry, và đường lỗi cần được bao phủ trong test.
- Saga đưa vào độ trễ vì mỗi bước phải chờ network call, queueing, và persistence trước khi bước tiếp theo bắt đầu. Nếu transaction chạy dài giữ resource nghiệp vụ, hệ thống phải giữ các lock đó nhẹ nhàng hoặc chịu rủi ro resource starvation nhân tạo. Kịch bản throughput cao làm trầm trọng thêm vấn đề back-pressure khi nhiều saga đồng thời retry cùng lúc, có khả năng dẫn tới "retry storm" làm bất ổn message broker hoặc downstream API.

Team hiểu và chuẩn bị cho các hạn chế và thách thức của saga vẫn có thể thu hoạch được lợi ích của pattern — khả năng phục hồi, loose coupling, và sự tiến hóa độc lập của service — trong khi tránh những bất ngờ khó chịu ở production. Điều này chắc chắn giúp đảm bảo dữ liệu nhất quán hơn giữa các service loosely coupled. Saga thường được điều phối bằng orchestration hoặc choreography. Cả hai phương pháp đều có ưu/nhược điểm. Ta bắt đầu khám phá choreography.

---

## Phần 3: Understanding and Implementing Choreography

Choreography là phương pháp điều phối saga nơi các service tham gia dùng message hoặc event để thông báo cho nhau về việc hoàn tất hay thất bại. Trong mô hình này, event broker nằm giữa các service nhưng không kiểm soát luồng message hay luồng saga. Điều này nghĩa là không có điểm tham chiếu hay kiểm soát trung tâm, và mỗi service giám sát một message đóng vai trò là confirmation trigger để nó khởi động thao tác của mình.

Trong saga choreography, mỗi participant tự trị, chỉ phản ứng với domain event mà nó đăng ký, và khi local transaction thành công hoặc thất bại, phát ra event mới sẽ được quan sát bởi bất kỳ service nào cần kết quả đó. Để relay đó đáng tin cậy, mỗi event phải phục vụ hai mục đích riêng biệt:

- Nó phải cung cấp vừa đủ dữ liệu nghiệp vụ để service tiếp theo làm việc.
- Nó phải mang một correlation identifier cho phép mọi message trong chuỗi được ghép lại về đúng một business request đã sinh ra saga.

Làm đúng hai mối quan tâm này là ranh giới giữa một workflow thanh lịch, dễ quan sát và một mớ message coupling chặt, dễ vỡ. Nếu một event chứa nhiều dữ liệu hơn mức recipient tức thời cần, một số vấn đề phát sinh:

- Payload trở nên lớn hơn cần thiết, tăng băng thông mạng và chi phí deserialize.
- Nó rò rỉ chi tiết implementation, dụ dỗ team khác phụ thuộc vào field không dành cho họ.
- Khi một service khác sau này quyết định thay đổi model nội bộ (ví dụ: đổi tên field hoặc thêm giá trị tùy chọn mới), hàng chục consumer downstream có thể vỡ đột ngột.

Bằng cách giới hạn payload chỉ ở tập con thông tin mà hop tiếp theo thực sự dùng, bạn duy trì ranh giới giữa các microservice và làm chậm hiệu ứng lan tỏa của thay đổi.

Ngược lại, giả sử event cung cấp ít hơn mức cần thiết. Trong trường hợp đó, recipient sẽ cần gọi service khác hoặc query data store khác để lấy ngữ cảnh còn thiếu, biến một luồng bất đồng bộ vốn dĩ thành luồng đồng bộ, "lắm chuyện".

### Correlation ID — ba mục đích thực tiễn

Một saga choreography cũng cần biết message nào thuộc về nhau. Một hành động người dùng đơn lẻ có thể sinh ra hàng chục event, retry, và compensation trải dài qua nhiều phút hoặc giờ. Bằng cách gán một correlation ID (thường gọi SagaId, CorrelationId, hoặc TraceId) khi saga bắt đầu và đưa token đó vào mọi event tiếp theo, bạn tạo ra một dấu vết breadcrumb mà hệ thống tracing, log aggregator, và người debug có thể theo dõi từ đầu tới cuối.

Correlation ID phục vụ ba mục đích thực tiễn:

- **Tracing and observability**: middleware distributed-tracing (VD: OpenTelemetry) có thể trực quan hóa toàn bộ saga như một span duy nhất chứa child span cho từng service, làm rõ độ trễ hoặc lỗi bắt nguồn từ đâu.
- **Idempotency and deduplication**: consumer có thể giữ một bản ghi nhẹ các message đã xử lý theo một ID cho trước. Nếu cùng event được gửi hai lần (do producer retry), consumer loại bỏ bản trùng mà không làm hỏng trạng thái local.
- **Business auditability**: product owner và compliance auditor thường cần trả lời câu hỏi chuyện gì đã xảy ra với payment #123. Lưu correlation ID cùng mỗi thay đổi database và message log cho phép tái dựng toàn bộ câu chuyện.

Điểm rút ra chính từ mô hình choreography là không có điểm kiểm soát trung tâm. Mỗi service sẽ giám sát event và quyết định có hành động hay không. Nội dung của message sẽ thông báo cho service biết nó nên làm gì, và nếu nó hành động, nó sẽ phản hồi bằng một message thông báo thành công hay thất bại của hành động đó. Nếu service cuối cùng của saga thành công, không có message nào được sinh ra và saga kết thúc.

Một saga được thiết kế tốt biến một workflow nhiều bước (như đăng ký bệnh nhân, tính phí, đặt lịch hẹn, và lưu tài liệu) thành một chuỗi các local transaction tự trị nhưng được điều phối.

### Ví dụ luồng choreography — đăng ký bệnh nhân + đặt lịch

1. Người dùng gửi yêu cầu đăng ký và đặt lịch hẹn (client request).
2. Registration service lưu dữ liệu người dùng mới, sau đó publish một event kèm theo chi tiết lịch hẹn và thanh toán liên quan. Event này có thể được gọi là, ví dụ, `USER_CREATED`.
3. Payment service lắng nghe event `USER_CREATED` và cố gắng xử lý thanh toán khi cần thiết. Khi thành công, nó sinh ra event `PAYMENT_SUCCESS`.
4. Appointment booking service xử lý event `PAYMENT_SUCCESS` và tiến hành thêm thông tin lịch hẹn như mong đợi. Service này thực hiện sắp xếp đặt lịch và sinh ra event `BOOKING_SUCCESS` cho service tiếp theo.
5. Document upload service nhận event `BOOKING_SUCCESS` và tiến hành upload tài liệu, thêm bản ghi vào data store của document service.

Ví dụ này cho thấy ta có thể track các tiến trình dọc theo chuỗi. Nếu muốn biết chi tiết của từng chặng và kết quả, ta có thể để registration service lắng nghe toàn bộ event và tạo cập nhật trạng thái hoặc log tiến trình dọc theo saga. Nó cũng có thể giao tiếp thành công hoặc thất bại của saga trở lại client.

### Rollback on Failure

Saga là cần thiết vì chúng cho phép ta rollback các thay đổi đã xảy ra khi có sự cố. Service sẽ publish một event nêu rõ một local transaction đã thất bại. Ta cần code bổ sung trong service trước đó để phản ứng với thủ tục rollback tương ứng. Ví dụ, nếu thao tác payment service thất bại, luồng sẽ như sau:

1. Appointment booking service thất bại khi xác nhận đặt lịch và publish event `BOOKING_FAILED`.
2. Payment service nhận event `BOOKING_FAILED` và tiến hành phát hành hoàn tiền cho khách hàng. Đây sẽ là bước remediation.
3. Registration service trước đó sẽ thấy event `BOOKING_FAILED` và thông báo cho client rằng việc đặt lịch không thành công.

Trong tình huống này, ta không hoàn toàn đảo ngược mọi bước vì ta giữ lại thông tin đăng ký của người dùng để tham khảo sau này. Điều thiết yếu, tuy nhiên, là service tiếp theo trong saga, upload tài liệu, không được cấu hình để lắng nghe event `BOOKING_FAILED`. Do đó, nó sẽ không có tác động gì trừ khi nhận event `BOOKING_SUCCESS`.

Một saga choreography chỉ thành công khi mọi participant publish event thành công hoặc thất bại của nó kịp thời. Lỗi nguy hiểm nhất, do đó, không phải là một event `BOOKING_FAILED` tường minh mà là sự vắng mặt hoàn toàn của message `BOOKING_SUCCESS` hoặc `BOOKING_FAILED`. Điều này có thể xảy ra nếu appointment booking service commit local database transaction rồi crash trước khi phát ra event, hoặc nếu kết nối message broker của nó bị treo trong khi tiến trình vẫn active. Vì không thành phần đơn lẻ nào quản lý toàn bộ tiến trình end-to-end, các service khác chờ đợi, không chắc chắn liệu booking có đang tiến hành, đã thất bại âm thầm, hay đã thành công nhưng không giao tiếp được. Giảm thiểu điều này đòi hỏi logic timeout tường minh bên ngoài chính các business service.

Một kỹ thuật phổ biến là giới thiệu một watchdog nhẹ hoặc process monitor service đăng ký toàn bộ saga event và giữ một deadline ledger theo correlation ID. Bất cứ khi nào nó quan sát một event `USER_CREATED`, ví dụ, nó ghi lại rằng message theo dõi `PAYMENT_SUCCESS` hoặc `BOOKING_FAILED` phải xuất hiện trong, ví dụ, năm phút. Nếu deadline trôi qua, watchdog phát ra một synthetic timeout event, như `BOOKING_TIMEOUT`, kích hoạt compensation giống như một lỗi thật.

Một biện pháp bảo vệ khác là transactional outbox pattern. Booking service viết ý định publish `BOOKING_SUCCESS` vào cùng database transaction lưu bản ghi booking, và một outbox dispatcher chạy nền đảm bảo event được phát ra ngay cả khi service crash. Trong thực tế, team có thể kết hợp cả outbox pattern và watchdog cho at-least-once delivery và tính liveness, để đóng khoảng trống mà choreography thuần túy để lại.

Ta cũng có thể lưu ý rằng các bước remediation của ta liên quan tới thao tác cụ thể. Payment service của ta có khả năng là một wrapper quanh một payment engine bên thứ ba, engine này cũng sẽ ghi một bản ghi database local của thao tác thanh toán. Do saga thiếu sự hoàn tất, nó sẽ không xóa bản ghi thanh toán mà đánh dấu nó là thanh toán đã hoàn tiền hoặc hủy nó trong các bước remediation.

Dù đây không phải sự phản ánh tương đương với việc một local transaction sẽ hoàn tác một database write, rollback có thể trông khác nhau cho mỗi service dựa trên business rule hoặc bản chất của thao tác. Ta cũng lưu ý rằng rollback của ta không bao gồm mọi service, vì business rule của ta khuyến nghị giữ lại thông tin đăng ký người dùng để tham khảo sau này.

### Compensation cũng có thể thất bại

Pattern choreography giả định rằng mọi service có thể hoàn tác công việc của nó; tuy nhiên, không gì đảm bảo bước compensation sẽ thành công ở lần thử đầu tiên. Ví dụ, một payment refund có thể thất bại vì payment provider offline, mạng thẻ của người dùng bị down, hoặc concurrency conflict xảy ra trong database thanh toán. Nếu compensation không thể hoàn tất, saga giờ ở trạng thái nguy hiểm nhất, nơi forward work đã bị rollback ở một số service (lịch hẹn bị hủy) nhưng chưa ở service khác (tiền vẫn bị giữ).

Hệ thống vững chắc giải quyết điều này theo nhiều lớp. Ở tầng hạ tầng, refund command được đặt trên một retry policy với exponential backoff để các gián đoạn tạm thời tự động biến mất. Sau một số lần retry nhất định, message được chuyển tới dead-letter queue, kích hoạt cảnh báo. Cuối cùng, watchdog đã mô tả trước đó duy trì một compensation outstanding flag cho mỗi saga. Giả sử flag vẫn được đặt vượt quá một service-level agreement (ví dụ, ba mươi phút cho financial settlement). Trong trường hợp này, monitor escalate tới một hàng đợi human-resolution chuyên biệt, nơi support staff có thể kiểm tra correlation ID, xác minh partial rollback, và thực hiện refund thủ công nếu cần thiết.

Bằng cách thừa nhận rằng bản thân compensation cũng có thể lỗi, và bằng cách cung cấp retry tự động, dead-letter handling, và human fallback, một choreography saga có thể phục hồi một cách graceful ngay cả khi lưới an toàn của nó bị rách.

Trong một message bus đáng tin cậy, việc gửi trùng không phải là bug mà là tiền đề thiết kế, vì vậy mọi handler, kể cả handler đảo ngược công việc, phải hành xử đúng nếu được gọi nhiều hơn một lần. Khi payment service nhận event `BOOKING_FAILED`, nó phát hành refund. Tuy nhiên, nếu network jitter khiến booking service republish cùng event failure, logic payment sẽ kích hoạt lại trừ khi được bảo vệ. Giải pháp là lưu một deduplication key, gồm correlation ID và loại event (hoặc monotonic event sequence), trong một idempotency table trong database payment. Trước khi xử lý, handler kiểm tra xem key đó đã tồn tại chưa; nếu có, nó acknowledge message và return mà không khởi tạo refund lần hai. Vì refund thường tương tác với gateway bên ngoài, một mức idempotency bổ sung, thường là idempotency key trong API của gateway, là thận trọng để ngăn cuộc gọi credit khách hàng nhiều lần nếu payment service tự retry sau crash. Xử lý compensation như thao tác first-class, idempotent ngăn một con đường phục hồi trở thành nguồn không nhất quán mới.

---

## Phần 4: Building a Choreography Saga Pattern with MassTransit and RabbitMQ

Saga này sẽ tập trung vào giao tiếp giữa appointment booking và payment service. Vì ta đã tạo appointment booking service, bạn có thể tiếp tục với công việc đó. Setup liên quan cho payment service sẽ được chia sẻ khi cần.

Dùng choreography saga pattern, ta sẽ xây dựng tương tác giữa appointment booking và payment processing microservice. Hệ thống sẽ thực hiện những điều sau:

1. Đặt lịch hẹn.
2. Kích hoạt yêu cầu thanh toán.
3. Nếu thanh toán thành công, saga hoàn tất.
4. Nếu thanh toán thất bại, việc đặt lịch bị rollback.

Ta phải bao gồm một shared class library để chứa các event model chuẩn được dùng xuyên suốt mỗi service:

```bash
dotnet new classlib -n Shared
dotnet sln add Shared/Shared.csproj
```

Sau đó, ta cần tạo payment service, thêm reference tới shared project, và appointment booking service:

```bash
dotnet new webapi -n PaymentsApi --use-controllers
dotnet sln add PaymentsApi/PaymentsApi.csproj
dotnet add PaymentsApi/PaymentsApi.csproj reference Shared/Shared.csproj
dotnet add AppointmentsApi/AppointmentsApi.csproj reference Shared/Shared.csproj
```

Mỗi microservice cần MassTransit, RabbitMQ, và EF Core. Chạy các lệnh này bên trong mỗi thư mục microservice:

```bash
dotnet add package MassTransit.AspNetCore
dotnet add package MassTransit.RabbitMQ
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
```

Cả hai (mọi) service sẽ tham chiếu shared project này, và các event type phải hiện diện ở đây để mỗi service có thể nhận diện chúng. Ta cần class `AppointmentCreated` và `PaymentProcessed` đại diện cho event mà service tương ứng sẽ gửi tới message queue. Các model này có thể được định nghĩa trong một file class gọi là `Events.cs`:

```csharp
public record AppointmentCreated(Guid CorrelationId, Guid AppointmentId, Guid PatientId, DateTime AppointmentDate);
public record PaymentProcessed(Guid CorrelationId, Guid AppointmentId, Guid PatientId, decimal Amount, bool Success);
```

Các model này được định nghĩa và dùng cho:

- `AppointmentCreated`: được publish khi một lịch hẹn được tạo
- `PaymentProcessed`: được publish bởi payment service, chỉ ra liệu thanh toán thành công hay thất bại

Ta cần thêm một cột mới vào entity class `Appointment` để chỉ ra liệu một lịch hẹn có bị hủy hay không:

```csharp
public class Appointment
{
    // Các thuộc tính hiện có
    public bool IsCanceled { get; set; } = false;
    // Code
}
```

Sau khi thêm cột này, chạy migration mới và cập nhật database để đảm bảo cột hiện diện:

```bash
dotnet ef migrations add AddIsCanceledToAppointments --project AppointmentsApi/AppointmentsApi.csproj --context AppointmentContext
dotnet ef database update --project AppointmentsApi/AppointmentsApi.csproj --context AppointmentContext
```

Vì ta đang xây dựng trên appointment booking microservice từ các chương trước, ta biết rằng một khi lịch hẹn được tạo, chi tiết của nó sẽ được đặt trên instance RabbitMQ ta đã thiết lập. Ta sẽ thêm một consumer khác để theo dõi message mà payment processor service có thể sinh ra trong trường hợp thất bại. Tạo một class mới trong folder Consumers gọi là `PaymentProcessedConsumer.cs`:

```csharp
public class PaymentProcessedConsumer(AppointmentContext _dbContext) : IConsumer<PaymentProcessed>
{
    public async Task Consume(ConsumeContext<PaymentProcessed> context)
    {
        var message = context.Message;
        Console.WriteLine($"[Appointment] CorrelationId={message.CorrelationId}: Payment " +
            $"{(message.Success ? "succeeded" : "failed")} for Appointment {message.AppointmentId}");

        if (!message.Success)
        {
            var appointment = await _dbContext.Appointments.FindAsync(message.AppointmentId);
            if (appointment != null)
            {
                appointment.IsCanceled = true;
                await _dbContext.SaveChangesAsync();
            }
        }
    }
}
```

Trong payment processor service, ta cần tạo một consumer sẽ tiêu thụ event `AppointmentCreated` được publish lên queue. Tạo một folder gọi là Consumers và, trong đó, một file gọi là `AppointmentCreatedConsumer`:

```csharp
public class AppointmentCreatedConsumer : IConsumer<AppointmentCreated>
{
    public async Task Consume(ConsumeContext<AppointmentCreated> context)
    {
        var message = context.Message;
        Console.WriteLine($"CorrelationId={message.CorrelationId}: Payment | Processing payment for Appointment ID:{message.AppointmentId}");
        // Giả lập thanh toán thất bại 50% thời gian
        var paymentSuccess = new Random().Next(0, 2) == 1;
        await context.Publish(new PaymentProcessed(message.CorrelationId,
            message.AppointmentId, message.PatientId, 100.00m, paymentSuccess));
    }
}
```

Trong kịch bản thực tế, tiến trình này sẽ kết nối tới payment gateway và xử lý thanh toán theo nhu cầu hệ thống. Trong ví dụ này, ta mô phỏng một lỗi tiềm ẩn và chèn lại message xác nhận vào queue.

Tương tự cấu hình ta đã tạo cho appointments service giao tiếp với RabbitMQ, ta cần thêm vào file `Program.cs` của payment service:

```csharp
builder.Services.AddMassTransit(mt =>
{
    mt.AddConsumer<AppointmentCreatedConsumer>();
    mt.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ReceiveEndpoint("payment-service-appointment-created", e =>
        {
            e.ConfigureConsumer<AppointmentCreatedConsumer>(context);
        });
    });
});
```

Giờ đây, sau khi cố gắng xử lý một thanh toán, một object `PaymentProcessed` được đặt lên queue và sẽ được tiêu thụ bởi `PaymentProcessedConsumer` trong appointment booking service. Khi property `Success` của object `PaymentProcessed` là false, appointment booking service sẽ đánh dấu lịch hẹn là đã hủy. Đây là ví dụ đơn giản hóa về cách xử lý thất bại, và bạn có thể cần thực hiện dọn dẹp dữ liệu phức tạp hơn, như xóa bản ghi tham chiếu hồi tố.

Bất kể độ phức tạp của nỗ lực remediation, khái niệm nền tảng vẫn là choreography dựa vào giao tiếp bất đồng bộ giữa mọi service tham gia saga.

### Pros and Cons

Trong một saga dựa trên choreography, điểm mạnh rõ ràng nằm ở tính đơn giản và sự tự trị của service. Mỗi microservice chỉ chịu trách nhiệm cho local transaction của nó và cho việc phát event mà service downstream tiêu thụ, giữ coupling thấp và cho phép các team riêng lẻ tiến hóa service của họ độc lập. Vì mỗi bước có thể publish thành công hoặc thất bại bất đồng bộ, bạn thậm chí có thể fan out một event khởi tạo duy nhất, như `BOOKING_CONFIRMED`, tới nhiều consumer song song. Tính song song này có thể giảm đáng kể độ trễ end-to-end khi không có phụ thuộc thứ tự giữa các thao tác đó.

Tuy nhiên, chính tính linh hoạt đó trở thành gánh nặng một khi phụ thuộc service ngầm len lỏi vào. Nếu hai hoặc nhiều consumer phụ thuộc cùng một event, điều này có thể dẫn tới race condition, trạng thái không nhất quán, hoặc lỗi tinh vi. Trong những trường hợp như vậy, bạn cần chia nhỏ event thành các loại chi tiết hơn hoặc áp đặt logic bổ sung lên consumer để trì hoãn công việc của chúng cho tới khi event tiên quyết đã đến. Cả hai cách tiếp cận đều đưa lại độ phức tạp vào một luồng vốn dĩ được dự định đơn giản, tự trị.

Observability là một lĩnh vực khác mà choreography thuần túy có thể gây căng thẳng cho công cụ của tổ chức. Vì saga diễn ra như một chuỗi message qua nhiều queue và ranh giới process, việc trace một business transaction đơn lẻ đòi hỏi một hệ thống distributed-tracing vững chắc (như OpenTelemetry), log tập trung được annotate với correlation ID, và lý tưởng nhất là một số hình thức trực quan hóa event-stream hoặc workflow dashboard. Nếu không có những điều đó, ngay cả một saga năm bước cũng có thể biến thành một mớ handoff vô hình, khiến việc phân tích nguyên nhân gốc rễ và các bài tập phục hồi trở nên chậm chạp đau đớn. Càng nhiều participant và compensating path bạn thêm, càng quan trọng để đầu tư vào các khả năng monitoring và tracing cross-cutting này.

Nhìn chung, choreography tỏa sáng trong các luồng đơn giản, loosely coupled nơi độ trễ và sự tự trị là tối quan trọng; tuy nhiên, nó có thể trở nên dễ vỡ và mờ đục khi số lượng service, phụ thuộc ngầm, và failure mode tăng lên. Tại điểm đó, giới thiệu một orchestrator trung tâm có thể giúp bằng cách làm cho toàn bộ workflow tường minh và dễ quan sát/kiểm soát hơn. Vì lý do này, ta sẽ xem xét một saga pattern khác, orchestration, implement một điểm kiểm soát trung tâm.

---

## Phần 5: Understanding and Implementing Orchestration

Khi nghĩ về orchestration, ta nghĩ về coordination. Một dàn nhạc là một nhóm nhạc công phối hợp làm việc cùng nhau để tạo ra một âm thanh thống nhất. Mỗi nhạc công chơi phần của mình, nhưng họ được dẫn dắt bởi một nhạc trưởng hướng dẫn họ theo cùng một con đường.

Phương pháp orchestration để implement một saga không khác biệt đáng kể về việc đòi hỏi một điểm kiểm soát trung tâm, giống một nhạc trưởng. Toàn bộ service được giám sát bởi một điểm kiểm soát trung tâm để đảm bảo chúng thực hiện vai trò của mình hiệu quả và báo cáo bất kỳ thất bại nào tương ứng. Kiểm soát trung tâm được gọi là orchestrator, một microservice đứng giữa client và mọi microservice khác. Nó xử lý mọi transaction, chỉ định các service tham gia khi nào hoàn tất một thao tác dựa trên phản hồi nhận được trong suốt saga. Orchestrator thực thi yêu cầu, track và diễn giải trạng thái của yêu cầu sau mỗi task, và xử lý các thao tác remediation khi cần thiết.

Dù bạn có thể tự tay xây dựng một saga orchestrator như một microservice đơn giản lắng nghe event và gọi service downstream, nhiều team chọn thay thế bằng các workflow engine được xây dựng chuyên biệt cung cấp durability, quản lý trạng thái, retry policy, và monitoring có sẵn. Đây là nơi các nền tảng được thiết kế để model các tiến trình phức tạp, chạy dài dưới dạng code tỏa sáng. Một số ví dụ như sau:

- **Temporal**: một workflow engine mã nguồn mở, code-centric, được thiết kế để viết ứng dụng đáng tin cậy, chạy dài. Temporal expose observability phong phú qua metric built-in và một web UI cho phép bạn inspect workflow đang in-flight, xem event history của chúng, và trigger retry hoặc compensation ad hoc. Vì workflow là code thông thường, team có thể tận dụng framework unit-testing hiện có của họ để viết automated test cho logic branching phức tạp và kịch bản thất bại, khiến Temporal trở thành lựa chọn phổ biến cho microservices orchestration trong môi trường polyglot.
- **Cadence**: phiên bản upstream, sớm hơn của Temporal server, ban đầu phát triển tại Uber. Nó bao gồm các tính năng như hierarchical namespace cho multi-tenant isolation, task routing linh hoạt, và pluggable persistence backend (VD: Cassandra hoặc MySQL). Cadence cũng hỗ trợ workflow versioning, cron scheduling của workflow, và external signal handling, cho phép human-in-the-loop approval hoặc can thiệp thủ công trong một saga vốn tự động hóa.
- **AWS Step Functions**: một dịch vụ orchestration được quản lý toàn diện bởi Amazon Web Services (AWS). Nó cho phép bạn định nghĩa serverless workflow như state machine dùng Amazon States Language (một DSL dựa trên JSON hoặc YAML) hoặc thông qua SDK cấp cao hơn như AWS CDK. Với tier Express Workflows, Step Functions cũng hỗ trợ workflow throughput cao, ngắn hạn với chi phí một phần nhỏ, dễ dàng kết nối microservice lại với nhau theo mô hình pay-as-you-go mà không cần dựng hạ tầng orchestration của riêng bạn.
- **Azure Durable Functions**: một extension của Azure Functions cung cấp hỗ trợ first-class để viết stateful serverless workflow trong C# hoặc JavaScript. Bạn viết orchestrator function gọi ra activity function; Durable Functions minh bạch checkpoint tiến trình của chúng vào Azure Storage để bạn có thể viết code trông có vẻ đồng bộ nhưng thực thi bất đồng bộ dưới nắp capo. Các pattern built-in bao gồm function chaining, fan-out/fan-in parallelism, human interaction qua external event, và durable timer dựa trên thời gian.
- **Camunda**: một nền tảng workflow và decision automation mã nguồn mở được xây dựng quanh chuẩn BPMN 2.0 và DMN. Khác với engine code-first, Camunda nhấn mạnh cách tiếp cận model-driven: business analyst có thể vẽ sơ đồ tiến trình trong graphical editor và deploy chúng tới một lightweight process engine. Với team microservices, Camunda tích hợp qua worker-pattern SDK hoặc cơ chế external task, cho phép service claim work item và hoàn tất chúng bất đồng bộ.

### Luồng orchestration cho ví dụ đăng ký + đặt lịch

1. Người dùng gửi yêu cầu đăng ký và đặt lịch hẹn (client request).
2. Orchestrator lưu trữ tập trung dữ liệu cần thiết cho các service tham gia saga.
3. Orchestrator gọi registration service, service này lưu dữ liệu người dùng mới và phản hồi với message thành công.
4. Orchestrator sau đó gửi một yêu cầu tới payment service và lắng nghe kết quả của thao tác thanh toán.
5. Orchestrator tiến hành kích hoạt appointment booking service, service này xử lý việc đặt lịch tương ứng và phản hồi cho orchestrator.
6. Orchestrator cuối cùng kích hoạt document upload service, upload các tài liệu, và thêm bản ghi vào document service data store.
7. Orchestrator sau đó phản hồi cho client, cập nhật trạng thái của thao tác và phản hồi client với kết quả tổng thể.

Về cơ bản, orchestrator chiếm giữ một vai trò trung tâm không chỉ là observer thụ động của event, mà là nhạc trưởng chủ động phát ra lệnh cho từng service tham gia và sau đó chờ phản hồi của chúng dưới dạng event. Về mặt kỹ thuật, các lệnh đó có thể được implement theo một trong hai cách nền tảng: REST đồng bộ hoặc messaging bất đồng bộ.

Dù có vẻ đơn giản để orchestrator thực hiện một HTTP POST tới appointment service và block ngay lập tức tới khi nó trả về 200 OK, tight coupling này hy sinh nhiều lợi ích về resilience vốn thúc đẩy việc dùng saga ngay từ đầu. Một cách tiếp cận vững chắc hơn là để orchestrator gửi một command message vào một queue hoặc topic mà appointment service đăng ký.

Vì vậy, thay vì service giao tiếp qua HTTP, chúng phát ra event mô tả kết quả của chúng. Luồng một chiều đó, từ orchestrator tới service, và từ service tới orchestrator, tách biệt ý định khỏi kết quả. Nó cũng cho phép orchestrator quản lý các tiến trình chạy dài. Thay vì giữ một kết nối HTTP mở trong nhiều phút trong khi một người phê duyệt yêu cầu bảo hiểm, orchestrator có thể tạm dừng trong state machine của nó tới khi nhận event liên quan tiếp theo. Đằng sau hậu trường, hầu hết workflow engine hoặc orchestrator tùy chỉnh xem mỗi event đến như một trigger để tiếp tục state machine của saga, đánh giá compensation branch, rồi gửi lệnh tiếp theo hoặc kết luận tiến trình.

Bằng cách phân biệt tường minh command khỏi event, một orchestrated saga xây dựng trên messaging đạt được loose coupling, khả năng phục hồi trước lỗi từng phần, và observability rõ ràng thông qua correlation ID. Nhờ đó nó bảo tồn tính tự trị của mỗi microservice trong khi vẫn cho phép một workflow đơn nhất, nhất quán trải dài qua nhiều bounded context.

### Rolling Back on Failure

Rolling back là phần thiết yếu nhất của việc implement một saga, và giống pattern choreography, ta bị chi phối bởi business rule của thao tác và các thao tác service riêng lẻ.

Khi một bước downstream trong saga orchestrated thất bại, nhiệm vụ đầu tiên của orchestrator là nhận diện event lỗi đó và đánh dấu workflow tổng thể đã gặp lỗi. Sau đó nó tra cứu bước hoặc các bước trước đó phải được hoàn tác để khôi phục trạng thái nhất quán. Với mỗi bước đó, orchestrator phát ra một compensating command lên message bus, và service liên quan tiêu thụ command đó, thực thi logic rollback nội bộ của nó, sau đó publish một event khi hoàn tất.

Orchestrator, khi thấy event, biết compensation đã thành công và có thể chuyển đổi saga instance của nó sang một trạng thái cuối cùng, thành công. Tại điểm đó, nó có thể phát ra một terminal notification hoặc cập nhật status store để client hoặc operator hiểu rằng yêu cầu booking ban đầu không được xử lý và bất kỳ tiền nào đã bị lấy đã được hoàn trả. Xuyên suốt tiến trình này, mỗi command và event mang cùng correlation identifier, đảm bảo khả năng truy vết end-to-end đầy đủ của cả đường forward lẫn đường reverse của business transaction.

Điểm rút ra chính là các service sẽ phản hồi thất bại tới một điểm trung tâm, điểm này sau đó sẽ điều phối các thao tác rollback trên các service khác nhau. Tái sử dụng kịch bản thất bại đã thảo luận trước đó, orchestration của ta sẽ trông như thế này:

1. Appointment booking service báo cáo thất bại cho orchestrator.
2. Orchestrator nhận thông báo thất bại và gọi payment service để phát hành hoàn tiền.
3. Orchestrator sẽ kích hoạt các thao tác dọn dẹp bổ sung cần thiết cho dữ liệu và trạng thái có thể đã được lưu trữ khi bắt đầu thao tác.
4. Orchestrator sẽ thông báo cho client về thất bại của thao tác.

Rollback ở đây có thể lập luận là dễ implement hơn. Điều này không phải vì ta đang thay đổi cách hoặc điều gì service làm, mà vì ta có thể đảm bảo thứ tự các remediation sẽ xảy ra nếu thứ tự đó là thiết yếu. Ta có thể hoàn thành điều đó mà không đưa quá nhiều độ phức tạp vào luồng.

Saga orchestrator này là bộ điều khiển trung tâm của tiến trình saga. Nó nhận một command, lắng nghe event, và xác định hành động tiếp theo. MassTransit tạo một saga state machine để quản lý tiến trình này. Một Saga State Machine lắng nghe command đến và track trạng thái của mỗi saga instance. Các bước cụ thể kiểm soát từng bước của saga.

---

## Phần 6: Implementing Orchestration

Ví dụ này minh họa implementation của appointment booking và payment processing dùng orchestrator saga pattern với MassTransit và RabbitMQ. Nó tuân theo cùng tiến trình như ví dụ choreography trước đó, nhưng service tương tác thông qua một saga orchestrator tập trung thay vì giao tiếp trực tiếp qua event.

Orchestrator gửi command tới mỗi service để chỉ thị chúng làm gì, sau đó lắng nghe event để học kết quả và quyết định bước tiếp theo. Sự phân tách rõ ràng giữa ý định (command) và kết quả (event) này là đặc trưng của giao tiếp saga orchestrated.

Trong ví dụ này, ta có:

1. `AppointmentRequested`: người dùng yêu cầu một lịch hẹn.
2. `SagaStateMachine`: điều này tạo một saga instance và kích hoạt thanh toán.
3. `PaymentRequested`: payment microservice xử lý thanh toán.
4. `PaymentSucceeded` hoặc `PaymentFailed`: payment microservice publish một event.

Saga State Machine sẽ phản ứng theo hai cách này:

- Nếu thành công, saga chuyển tiếp sang `Completed` (lịch hẹn được xác nhận)
- Nếu thất bại, saga chuyển tiếp sang `Failed` (lịch hẹn bị rollback/hủy)

Implementation orchestration saga của MassTransit đòi hỏi một số lưu trữ database. Mỗi saga instance được đại diện bởi một state machine tiến triển để phản hồi message đến. Các workflow này thường trải dài phút hoặc giờ, và bản thân tiến trình orchestrator có thể bị restart, scale out, hoặc failover. Do đó, bạn không thể dựa vào in-memory object để theo dõi tiến độ của saga. Vì vậy, MassTransit tích hợp với durable storage, đảm bảo mọi transition và bất kỳ dữ liệu tạm thời nào được ghi vào một backing store. Điều này cho phép orchestrator tiếp tục đúng chỗ đã dừng khi event tiếp theo tới, đảm bảo message retried hoặc trùng lặp được xử lý idempotently, và cung cấp một audit trail hoàn chỉnh của vòng đời saga.

Ta có thể thêm nó vào Shared project, để nó accessible cho mọi project tham chiếu:

```bash
dotnet add package MassTransit.EntityFrameworkCore
```

Ta bắt đầu bằng cách thêm các event model sau vào thư mục shared:

```csharp
// Command yêu cầu đặt lịch hẹn
public record AppointmentCreated(Guid AppointmentId, Guid PatientId, DateTime AppointmentDate);
// Command từ saga orchestrator tới payment service
public record PaymentRequested(Guid AppointmentId, decimal Amount);
// Event được publish bởi payment service
public record PaymentSucceeded(Guid AppointmentId);
public record PaymentFailed(Guid AppointmentId);
// Event kết luận saga
public record AppointmentConfirmed(Guid AppointmentId);
public record AppointmentCanceled(Guid AppointmentId);
```

Các model này phục vụ các mục đích sau:

1. `AppointmentCreated`: appointment service gửi cái này tới saga orchestrator để bắt đầu booking saga.
2. `PaymentRequested`: saga orchestrator chỉ thị payment service tính phí người dùng.
3. `PaymentSucceeded` | `PaymentFailed`: payment service publish kết quả của nó.
4. `AppointmentConfirmed` | `AppointmentCanceled`: saga thông báo kết quả.

Cho ví dụ này, ta sẽ tập trung vào event `AppointmentCanceled` và định nghĩa một consumer trong folder Consumers của appointments service:

```csharp
public class AppointmentCanceledConsumer : IConsumer<AppointmentCanceled>
{
    private readonly AppointmentContext _dbContext;
    public AppointmentCanceledConsumer(AppointmentContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task Consume(ConsumeContext<AppointmentCanceled> context)
    {
        var appointment = await _dbContext.Appointments.FindAsync(context.Message.AppointmentId);
        if (appointment != null)
        {
            appointment.IsCanceled = true;
            await _dbContext.SaveChangesAsync();
            Console.WriteLine($"[AppointmentService] Appointment {appointment.AppointmentId} canceled.");
        }
    }
}
```

Lưu ý rằng, dù ta tập trung vào kết quả của một appointment booking, khái niệm vẫn là mỗi service sẽ chịu trách nhiệm xử lý event lỗi liên quan và rollback khi cần thiết.

Với loại saga này, ta cần track kết quả của các bước khác nhau, điều này đưa vào nhu cầu về một database mới. Ta đã khám phá kỹ chi tiết thiết lập một database dùng EF Core, vì vậy ta sẽ bỏ qua chi tiết các bước này và tiến hành thêm một database mới. Ta sẽ tạo một entity model và database context mới trong một folder mới trong Shared project gọi là Data:

```csharp
public class BookingSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; } // Để correlate message
    public Guid AppointmentId { get; set; }
    public string CurrentState { get; set; } = "None";
}
```

Dưới đây là EF Core DbContext đại diện database sẽ được dùng cho saga storage:

```csharp
public class BookingSagaDbContext : DbContext
{
    public BookingSagaDbContext(DbContextOptions<BookingSagaDbContext> options) : base(options) { }
    public DbSet<BookingSagaState> BookingStates { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<BookingSagaState>(cfg =>
        {
            cfg.HasKey(x => x.CorrelationId);
        });
        base.OnModelCreating(modelBuilder);
    }
}
```

EF repository của MassTransit mong đợi một `CorrelationId` primary key cho saga state.

Giờ, ta tạo saga state machine. Ta có thể đặt nó trong một folder gọi là SagaOrchestrator và gọi class `BookingStateMachine.cs`:

```csharp
public class BookingStateMachine : MassTransitStateMachine<BookingSagaState>
{
    public BookingStateMachine()
    {
        // Định nghĩa các trạng thái
        InstanceState(x => x.CurrentState);

        Event(() => AppointmentRequestedEvent, x =>
        {
            x.CorrelateById(ctx => ctx.Message.AppointmentId);
            x.SelectId(ctx => ctx.Message.AppointmentId); // Saga Key
        });
        Event(() => PaymentSucceededEvent, x => x.CorrelateById(ctx => ctx.Message.AppointmentId));
        Event(() => PaymentFailedEvent, x => x.CorrelateById(ctx => ctx.Message.AppointmentId));

        Initially(
            When(AppointmentRequestedEvent)
                .Then(ctx =>
                {
                    ctx.Instance.AppointmentId = ctx.Data.AppointmentId;
                    Console.WriteLine($"[Saga] Received AppointmentRequested for {ctx.Data.AppointmentId}");
                })
                .TransitionTo(BookingInitiated)
                .Publish(ctx => new PaymentRequested(ctx.Data.AppointmentId, 99.99m)) // Yêu cầu thanh toán
        );

        During(BookingInitiated,
            When(PaymentSucceededEvent)
                .Then(ctx =>
                {
                    Console.WriteLine($"[Saga] Payment succeeded for {ctx.Instance.AppointmentId}");
                })
                .Publish(ctx => new AppointmentConfirmed(ctx.Instance.AppointmentId))
                .TransitionTo(BookingCompleted)
                .Finalize(),
            When(PaymentFailedEvent)
                .Then(ctx =>
                {
                    Console.WriteLine($"[Saga] Payment failed for {ctx.Instance.AppointmentId}");
                })
                .Publish(ctx => new AppointmentCanceled(ctx.Instance.AppointmentId))
                .TransitionTo(BookingFailed)
                .Finalize()
        );

        // Đặt trạng thái là final để cleanup
        SetCompletedWhenFinalized();
    }

    // Trạng thái
    public State BookingInitiated { get; private set; }
    public State BookingCompleted { get; private set; }
    public State BookingFailed { get; private set; }

    // Event
    public Event<AppointmentRequested> AppointmentRequestedEvent { get; private set; }
    public Event<PaymentSucceeded> PaymentSucceededEvent { get; private set; }
    public Event<PaymentFailed> PaymentFailedEvent { get; private set; }
}
```

Khi `AppointmentRequested`, ta lưu `AppointmentId` và publish `PaymentRequested`. Khi một thanh toán thành công được thực hiện, một event `AppointmentConfirmed` được publish. Trong trường hợp một event `PaymentFailed`, một event `AppointmentCanceled` được publish.

Giờ ta đã định nghĩa các event, state machine, trạng thái, và database context cho saga, hãy đăng ký các asset này trong file `Program.cs`. Đầu tiên, ta đăng ký database context mới:

```csharp
builder.Services.AddDbContext<BookingSagaDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("SagaDatabase")));
```

Để đơn giản, ta sẽ dùng database PostgreSQL và đây là connection string, ta sẽ thêm vào file `appsettings.json`:

```json
"ConnectionStrings": {
  "SagaDatabase": "Host=localhost;Database=sagadb;Username=postgres;Password=password"
}
```

Tiếp theo, ta cần đăng ký state machine và database context trong cấu hình MassTransit. Ta sẽ sửa cấu hình để trông như thế này. Trong lúc đó, ta cũng sẽ đăng ký `AppointmentCanceledConsumer`:

```csharp
builder.Services.AddMassTransit(x =>
{
    // Consumer hiện có
    x.AddConsumer<AppointmentCanceledConsumer>();
    x.AddSagaStateMachine<BookingStateMachine, BookingSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Pessimistic;
            r.AddDbContext<DbContext, BookingSagaDbContext>();
            r.UsePostgres();
        });
    x.UsingRabbitMq((context, cfg) =>
    {
        // Cấu hình receive endpoint
        // Cấu hình saga endpoint
        cfg.ConfigureEndpoints(context);
    });
});
```

`AddSagaStateMachine<BookingStateMachine, BookingSagaState>()` chỉ thị MassTransit host saga machine với một EF-based repository. Cuộc gọi `cfg.ConfigureEndpoints(context)` tự động wire up một queue cho endpoint của saga.

Để mô phỏng, ta cũng có thể implement MassTransit và `PaymentRequestedConsumer` trong payment service:

```csharp
public class PaymentRequestedConsumer : IConsumer<PaymentRequested>
{
    public async Task Consume(ConsumeContext<PaymentRequested> context)
    {
        var success = new Random().Next(0, 2) == 1;
        Console.WriteLine($"[PaymentService] Payment requested for Appointment {context.Message.AppointmentId}. Success? {success}");
        if (success)
        {
            await context.Publish(new PaymentSucceeded(context.Message.AppointmentId));
        }
        else
        {
            await context.Publish(new PaymentFailed(context.Message.AppointmentId));
        }
    }
}
```

Tương tự, ta đăng ký MassTransit và consumer này trong file `Program.cs` của payment service:

```csharp
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddConsumer<PaymentRequestedConsumer>();
    cfg.UsingRabbitMq((context, config) =>
    {
        config.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        config.ConfigureEndpoints(context);
    });
});
```

Dùng saga với MassTransit mang lại nhiều lợi thế, cho phép ta track và retry mỗi bước độc lập, từ đó bù trừ cho các thao tác thất bại. State machine sẽ cho phép ta track tiến độ của mỗi thao tác, và vì service giao tiếp qua message, chúng có thể tiến hóa độc lập trong khi vẫn duy trì tính toàn vẹn tiến trình.

### Pros and Cons

Một trong những điểm mạnh hấp dẫn nhất của saga pattern dựa trên orchestrator là mức độ kiểm soát nó mang lại đối với toàn bộ vòng đời của một business process. Vì orchestrator phát tường minh mỗi command theo lượt, bạn có kiểm soát chính xác thứ tự thao tác, hình dạng của dữ liệu chảy giữa các bước, và logic xử lý lỗi chính xác ở mỗi điểm nối. Choreography tuần tự, message-driven này nghĩa là orchestrator có thể áp dụng conditional branching và enforce global timeout hoặc retry policy tập trung, mà không rải logic đó khắp mọi microservice.

Mức độ giám sát tập trung đó, tuy nhiên, có cái giá. Dù orchestrator giải phóng mỗi participant khỏi việc phải theo dõi event của mọi thành phần khác, bạn vẫn phải thiết kế và test kỹ lưỡng compensating transaction cho mọi failure tiềm ẩn. Rollback một thanh toán thành công, ví dụ, nghĩa là orchestrator phải vừa publish một command `RefundPayment` vừa chờ một event `PaymentRefunded` trước khi nó có thể đánh dấu toàn bộ saga là thất bại. Dù orchestrator tập trung hóa workflow, nó không tự động làm compensation trở nên đơn giản. Bạn vẫn cần logic refund idempotent, reconciliation với payment gateway, và cập nhật database trong payment service, cũng như các accommodation cho partial failure trong quá trình refund.

Về mặt kỹ thuật, luồng tuần tự này được enforce bởi state machine của orchestrator, nơi mỗi event đến kích hoạt một transition sang trạng thái mới, và chỉ trong trạng thái đó orchestrator phát ra command tiếp theo. MassTransit wire up dedicated endpoint cho cả command lẫn event, và saga repository của orchestrator lưu trạng thái hiện tại của nó trong một database, cho phép nó an toàn tạm dừng giữa các bước, sống sót qua restart, và xử lý message trùng lặp idempotently.

Điều thiết yếu cần nhớ là dù orchestration giảm coupling trực tiếp service-to-service, mọi service vẫn phải tuân theo các command và event được định nghĩa bởi saga. Appointment booking service, ví dụ, phải implement một handler cho `BookAppointmentCommand` tôn trọng input contract mong đợi và phát ra `BookingSucceeded` hoặc `BookingFailed`. Theo nghĩa này, service vẫn tự trị trong logic nội bộ của chúng nhưng phải tuân thủ một shared choreography contract được thiết lập bởi orchestrator.

Vì tích hợp chặt với `SagaStateMachine<T>` và `SagaStateMachineInstance` của MassTransit có thể khiến logic saga cảm giác coupling với API của MassTransit, một chiến lược giảm thiểu là cô lập định nghĩa business workflow cốt lõi của bạn vào một library hoặc layer riêng biệt. Ở đó, bạn định nghĩa interface và data model thuần túy, framework-agnostic. State machine và EF repository đặc thù của MassTransit sau đó trở thành một infrastructure concern chỉ đơn thuần wire các định nghĩa đó vào message bus và database. Sự tách biệt đó giảm bớt khó khăn khi hoán đổi messaging framework hoặc tiến hóa logic saga của bạn độc lập với đường ống của orchestrator.

Hãy đảm bảo scope đúng nhu cầu dùng saga của bạn. Nếu saga của bạn có rất ít bước, cả một state machine có thể là overkill. Đôi khi, một cách tiếp cận đơn giản hơn, như xây dựng một orchestrator service và track trạng thái trong một bảng, có thể là đủ. Đôi khi, có một kịch bản request → response → done hoặc fail súc tích có thể được implement, nơi library có nhiều tính năng nâng cao như timer, chiến lược concurrency, và transition nâng cao có thể không được dùng nhưng vẫn đòi hỏi học tập.

Dù hỗ trợ EF concurrency built-in có lợi, nó có thể thêm độ phức tạp nếu bạn chưa quen thuộc với concurrency token. Bạn có thể phải cấu hình repository của mình cẩn thận cho tải ở quy mô production.

---

## Tổng kết chương

Ta bắt đầu bằng cách nhấn mạnh tầm quan trọng của tính nhất quán dữ liệu trong kiến trúc microservices, nơi mỗi service thường có database chuyên biệt riêng. ACID transaction truyền thống trở nên khó khăn trong bối cảnh này vì một thao tác nghiệp vụ đơn lẻ có thể trải dài qua nhiều service. Saga pattern, được giới thiệu như một giải pháp cho thách thức này, chia một transaction lớn hơn thành nhiều local transaction. Mỗi microservice tham gia xử lý bước của nó độc lập và giao tiếp thành công hoặc thất bại thông qua event, từ đó cho phép các hành động bù trừ khi một trong các bước thất bại.

Một phần chính của chương bao trùm khái niệm coordination, có thể được implement thông qua choreography hoặc orchestration. Trong cách tiếp cận choreography, mỗi service đăng ký các event liên quan và publish event mới sau khi hoàn tất local transaction của nó. Nếu có gì sai, các service cần rollback lắng nghe event thất bại và thực hiện compensation local. Cách tiếp cận này giữ cho service tự trị nhưng có thể trở nên ngày càng phức tạp khi số lượng participant và kịch bản thất bại có thể tăng lên, khiến việc monitor và debug trở nên khó khăn hơn.

Ta đã xem xét một ví dụ thực tế dùng MassTransit và RabbitMQ, giải thích khó khăn của việc gia tăng độ phức tạp trong một saga được choreography. Khi thêm nhiều service và bước hơn, người ta dễ dàng rơi vào một mạng lưới phức tạp của event và hành động remediation. Đây là một lý do đáng kể để cân nhắc orchestration như một lựa chọn thay thế.

Trong một saga được orchestrated, một microservice chuyên biệt được biết đến như orchestrator ngồi ở trung tâm của hệ thống và đưa ra quyết định về workflow. Thay vì để service phát ra event theo lịch trình riêng của chúng, orchestrator kiểm soát mỗi bước, gọi các service riêng lẻ theo trình tự, lắng nghe thành công hoặc thất bại, rồi hành động tương ứng. Nếu một thất bại xảy ra trong bất kỳ bước nào, orchestrator có thể xác định hướng tốt nhất cho việc rollback hoặc bù trừ. Dù mô hình này thường dễ debug và bảo trì hơn, nó đưa vào một điểm kiểm soát duy nhất có thể trở thành bottleneck nếu không được thiết kế cẩn thận. Nó cũng đòi hỏi orchestrator biết chính xác cách mỗi service tham gia và điều gì cần xảy ra tiếp theo, có thể thêm độ phức tạp cho logic trung tâm.

**Bài học cốt lõi**: implement pattern này làm tăng độ phức tạp code tổng thể và đòi hỏi lập kế hoạch kỹ lưỡng về error handling, retry logic, và compensating operation. Tuy nhiên, lợi ích về tính nhất quán và độ tin cậy có thể rất đáng kể, vì ứng dụng sẽ được trang bị tốt hơn để xử lý lỗi từng phần và duy trì dữ liệu nhất quán phân tán.

Giờ ta đã khám phá saga và cách chúng hoạt động cũng như được implement, chương tiếp theo sẽ chuyển sang khám phá các kỹ thuật xây dựng microservices có khả năng phục hồi (resilient) hơn.
