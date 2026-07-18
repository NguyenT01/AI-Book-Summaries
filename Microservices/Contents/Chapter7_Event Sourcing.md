# Bài học: Event Sourcing — Khi "trạng thái hiện tại" không đủ, ta cần biết "chuyện gì đã xảy ra"

> Chương 7 — Applying Event-Sourcing Patterns
> Ví dụ xuyên suốt: Event Store cho AppointmentsApi dùng PostgreSQL + EF Core + MediatR (`AppointmentCreatedEvent`, `AppointmentCancelledEvent`)

---

## Phần 1: Vấn đề thực tế — dữ liệu phân tán dễ "lệch pha"

Chương trước khám phá CQRS pattern, khuyến khích ta tạo sự tách biệt rõ ràng giữa code và data source chi phối thao tác đọc và ghi. Với kiểu tách biệt này, ta có rủi ro gặp sự không nhất quán trong dữ liệu giữa các thao tác, dẫn tới nhu cầu về kỹ thuật bổ sung để đảm bảo tính nhất quán dữ liệu.

Ta phải đối mặt với pattern microservices điển hình, nơi mỗi service được kỳ vọng có data store riêng. Hãy nhớ rằng sẽ có tình huống dữ liệu cần được chia sẻ giữa các service. Cần có cơ chế nào đó vận chuyển dữ liệu đúng cách giữa các service để duy trì đồng bộ.

Event sourcing được ca ngợi như một giải pháp cho vấn đề này, nơi một data store mới được giới thiệu để theo dõi toàn bộ thao tác command khi chúng xảy ra. Các bản ghi trong data store này được coi là event và chứa đủ thông tin để hệ thống theo dõi điều gì xảy ra với mỗi thao tác command. Các bản ghi này được gọi là event và đóng vai trò là kho lưu trữ trung gian cho kiến trúc dịch vụ hướng sự kiện hoặc bất đồng bộ. Chúng cũng có thể đóng vai trò như audit log vì chúng sẽ lưu trữ toàn bộ chi tiết cần thiết để replay các thay đổi được thực hiện đối với domain.

Sau khi đọc chương này, ta sẽ có thể thực hiện những điều sau:

- Hiểu event là gì
- Hiểu pattern event-sourcing trong microservices
- Hiểu cách tạo một event store
- Hiểu cách dùng event store cho thao tác đọc trong CQRS

### Analogy: Sổ cái kế toán thay vì chỉ ghi số dư

Hãy hình dung sự khác biệt giữa hai cách quản lý tài khoản ngân hàng. Cách 1: chỉ lưu một con số duy nhất — "số dư hiện tại là 5 triệu". Cách 2: lưu toàn bộ sổ cái giao dịch — "nạp 10 triệu, rút 3 triệu, nạp 2 triệu, rút 4 triệu..." Với cách 2, bạn không chỉ biết số dư hiện tại (bằng cách cộng dồn tất cả giao dịch), mà còn có thể trả lời câu hỏi "tại sao" — tại sao số dư lại là 5 triệu, những gì đã xảy ra để dẫn tới con số đó. Event Sourcing chính là cách 2 áp dụng cho toàn bộ hệ thống phần mềm.

---

## Phần 2: Hiểu về Events và Event Sourcing

Event sourcing là một pattern kiến trúc mạnh mẽ lưu trữ trạng thái của một hệ thống dưới dạng một chuỗi event bất biến. Các event này đại diện cho các thay đổi đối với hệ thống theo thời gian thay vì lưu trực tiếp trạng thái hiện tại. Bằng cách tạo và lưu trữ event, developer có thể đạt được mức độ hiểu biết chi tiết hơn về sự tiến hóa của trạng thái hệ thống, khiến nó đặc biệt có lợi cho các ứng dụng đòi hỏi audit trail vững chắc, tái tạo trạng thái, hoặc xử lý real-time.

### Events là gì?

Một event, trong bối cảnh phát triển phần mềm, đề cập tới một bản ghi bất biến của một sự việc xảy ra trong hệ thống, thường là kết quả của việc xử lý thành công một command. Chúng thường được lưu trữ ở định dạng JSON hoặc định dạng phổ quát thay thế (VD: Avro, Protobuf, v.v.) để đảm bảo tương thích không phụ thuộc hệ thống và hỗ trợ versioning. Sau đó chúng được dùng để thực hiện các hành động bổ sung ở background. Ví dụ, trong hệ thống quản lý phòng khám của ta, một event có thể gắn với các thao tác như:

- **AppointmentCreated**: một bệnh nhân đặt lịch hẹn
- **DoctorAdded**: một bác sĩ mới được thêm vào hệ thống
- **PaymentProcessed**: một khoản thanh toán được hoàn tất
- **DiagnosisAdded**: một bản ghi mới được thêm vào bệnh sử của bệnh nhân

Event là bất biến, nghĩa là chúng không thể bị thay đổi sau khi được ghi lại. Tính bất biến này là nền tảng để đảm bảo tính toàn vẹn và độ tin cậy của event. Event mang lại nhiều lợi thế cho phát triển phần mềm hiện đại:

- **Auditability**: bằng cách ghi lại mọi thay đổi dưới dạng event, bạn có thể tái tạo lại lịch sử hoàn chỉnh của hoạt động hệ thống, vô giá cho compliance và troubleshooting.
- **Decoupling**: event thúc đẩy sự tách rời giữa các thành phần. Event producer không cần biết về consumer, cho phép sự tiến hóa độc lập của các phần hệ thống. Điều này thường được thực hiện bằng message broker, như đã thấy ở Chapter 4, Asynchronous Communication between Microservices.
- **Scalability**: hệ thống hướng sự kiện có thể scale hiệu quả hơn bằng cách phân phối việc xử lý event qua nhiều consumer hoặc microservice. Điều này có nghĩa là số lượng consumer có thể được tăng theo nhu cầu khi có "quá nhiều" message cần xử lý. Tăng số lượng consumer sẽ đảm bảo hiệu quả theo nhu cầu.
- **Reproducibility**: vì event bất biến, bạn có thể replay chúng để tái tạo trạng thái của hệ thống tại bất kỳ thời điểm nào, đặc biệt hữu ích cho debug và testing. Về bối cảnh, event bất biến nghĩa là chúng không thể được sửa đổi sau khi tạo. Điều này thường được thực hiện qua versioning, cryptographic hashing, hoặc dùng data store append-only.
- **Real-time processing**: event cho phép cập nhật và hành động real-time. Ví dụ, một payment event có thể kích hoạt cập nhật inventory hoặc thông báo khách hàng.
- **Flexibility in data modeling**: event lưu trữ "lý do" (why) đằng sau thay đổi trạng thái, cung cấp ngữ cảnh phong phú hơn và cho phép insight mới khi ứng dụng của bạn tiến hóa.

Trong một số trường hợp, event được dùng cho mục đích phân tích, thông báo, và thậm chí thiết lập audit trail của toàn bộ thao tác dữ liệu trong hệ thống. Với sự đa dạng của cách event có thể được dùng, cấu trúc của chúng phải được thiết lập và nhất quán để đảm bảo chúng có thể đáp ứng nhu cầu của hệ thống.

### Key Attributes of Events — thuộc tính chính của event

Event có thể được dùng để xây dựng nền tảng cho chức năng cốt lõi của bất kỳ ứng dụng nào. Dù khái niệm này có thể phù hợp cho nhiều tình huống, ta cần hiểu một số thuộc tính chính của event, scoping đúng nhu cầu giới thiệu chúng, và duy trì các tiêu chuẩn cụ thể trong implementation của ta.

Một event thường chứa các thuộc tính sau:

- **Event name**: tên mô tả chỉ loại event
- **Timestamp**: thời điểm chính xác event xảy ra
- **Event ID**: định danh duy nhất cho event
- **Aggregate ID**: định danh gắn event với một entity hoặc process cụ thể
- **Payload**: dữ liệu bổ sung nắm bắt chi tiết của event (VD: appointment ID, chi tiết bệnh nhân, v.v.)
- **Metadata**: dữ liệu tùy chọn cung cấp ngữ cảnh, như nguồn của event (như tên model hoặc kiểu dữ liệu) hoặc phiên bản của schema

Aggregate ID không nhất thiết đặc thù cho các aggregate ta đã đề cập ở Chapter 4 (Applying Domain-Driven Design Principles). Nó nhìn chung đề cập tới ID của entity đang được theo dõi khi event được tạo. Do đó, event store có thể được dùng trong hệ thống không implement DDD, và mở rộng ra, trong các ứng dụng không dựa trên microservices.

Hãy nhớ rằng event đại diện cho các sự thật hoặc những gì đã xảy ra trong hệ thống, và do đó, business logic không nên thay đổi các sự thật này.

---

## Phần 3: Event-Sourcing Patterns trong Microservices

Ứng dụng được xây dựng với kiến trúc microservices được cấu trúc để có các service tách rời và độc lập. Mỗi microservice có thể có một data store độc lập, điều này đặt ra thách thức riêng để giữ dữ liệu đồng bộ giữa các service.

Thách thức này tăng lên khi ta xét rằng ta phải thỏa hiệp với các nguyên tắc ACID của mình, cụ thể là chữ A, nghĩa là atomic. Ta không thể đảm bảo rằng toàn bộ thao tác ghi của ta sẽ được hoàn tất như một khối. Nguyên tắc atomic quy định rằng toàn bộ thao tác dữ liệu nên hoàn tất hoặc thất bại như một khối. Với việc cho phép các công nghệ khác nhau được dùng cho data store, ta không thể đảm bảo điều đó.

Với tất cả các yếu tố này, ta có thể chuyển sang event sourcing pattern, cho phép ta lưu trữ event theo dõi toàn bộ hoạt động xảy ra đối với dữ liệu trong mỗi service. Pattern này có lợi cho giao tiếp bất đồng bộ giữa các service, nơi ta có thể theo dõi mọi thay đổi dưới dạng event. Các event này có thể đóng vai trò:

- **Persistent events**: event chứa đủ chi tiết để thông báo và tái tạo lại domain object
- **Audit logs**: event được tạo với mỗi thay đổi để chúng có thể đóng vai trò kép làm audit
- **Entity state identification**: ta có thể dùng event để xem trạng thái tại một thời điểm cụ thể của một entity theo yêu cầu

Ý tưởng theo dõi thay đổi đối với entity được gọi là **replaying**. Ta có thể replay event theo 2 bước:

1. Lấy toàn bộ hoặc một phần event đã lưu cho một aggregate cho trước.
2. Lặp qua toàn bộ event và trích xuất thông tin liên quan để tái tạo lại instance của aggregate.

Event sourcing liên quan tới việc query bản ghi bằng `AggregateId` và `Timestamp` để sắp xếp và định danh bản ghi event. Event replay đòi hỏi lặp qua toàn bộ event, lấy thông tin, sau đó thay đổi trạng thái của aggregate mục tiêu.

### Pros and Cons of Event Sourcing — ưu và nhược điểm

Event replay và cách ta thực hiện update phụ thuộc vào việc aggregate có phải là domain aggregate hay không. Nếu aggregate dựa vào domain service để thao tác, ta cần làm rõ rằng replay không phải về việc lặp lại hoặc thực hiện lại (redo) thao tác command. Command là một thao tác thay đổi dữ liệu trong hệ thống, có thể tạo và lưu trữ một event. Thao tác này cũng có khả năng là một thao tác chạy dài với side effect sinh dữ liệu event, mà ta có thể không muốn.

Event replay xem xét dữ liệu, thực hiện logic để trích xuất thông tin, và áp dụng event để tái tạo lại aggregate. Nhìn chung, event đã lưu có thể được xử lý khác với ứng dụng đang dùng kỹ thuật này.

Ngoài ra, event là dữ liệu được lưu trữ bên dưới trạng thái hiện tại. Điều này nghĩa là ta có thể tái sử dụng chúng để dựng bất kỳ projection nào của dữ liệu ta cần. Projection ad-hoc của dữ liệu có thể được dùng để đọc dữ liệu lưu trong cấu trúc dự án CQRS, phân tích dữ liệu, business intelligence, và thậm chí trí tuệ nhân tạo và mô phỏng. Event là hằng số và sẽ luôn giống nhau bây giờ và sau này. Đây là một lợi ích bổ sung ở chỗ ta luôn có thể chắc chắn về sự nhất quán của dữ liệu.

Dù event sourcing mang lại nhiều lợi ích, nó cũng đưa vào một số phức tạp nhất định. Developer cần hiểu sự phức tạp của event sourcing, có thể liên quan tới đường cong học tập dốc hơn so với cách tiếp cận truyền thống. Ta phải điều hướng các thách thức như:

- Quản lý thay đổi đối với cấu trúc của event. Điều này sẽ đòi hỏi cân nhắc cẩn thận về versioning và backward compatibility.
- Query phức tạp vì trạng thái không được lưu trực tiếp. Do đó, query trạng thái hiện tại có thể đòi hỏi pattern chuyên biệt hoặc tính toán bổ sung.
- Xử lý lỗi và đảm bảo consistency khi có lỗi xảy ra. Trong trường hợp này, bạn có thể cần cơ chế retry và replay để đảm bảo event được lưu trữ đúng cách.
- Tối ưu hiệu năng vì event replaying có thể nhanh chóng trở thành thao tác intensive dựa trên số lượng event đã log. Để giải quyết điều này, ta có thể chụp snapshot của trạng thái aggregate và business entity đã được sửa đổi gần đây, dùng các snapshot này như một phiên bản đã lưu của bản ghi, tránh nhu cầu lặp qua nhiều event.

---

## Phần 4: Creating an Event Store — Tạo một Event Store

Ta phải hiểu event store trước khi tới giai đoạn scoping tạo một cái. Cho cuốn sách này, ta sẽ kết luận rằng event store là nguồn bản ghi dài hạn có thứ tự, dễ truy vấn, và bền vững đại diện cho event đối với entity trong data store.

Hãy nhớ rằng một bản ghi event sẽ có `AggregateId`, `Timestamp`, `EventType`, và `Payload` đại diện cho trạng thái của dữ liệu tại thời điểm đó. Một ứng dụng sẽ lưu trữ bản ghi event trong một data store. Data store này có một API hoặc interface nào đó cho phép thêm và lấy event cho một entity hoặc aggregate. Event store cũng có thể hoạt động như một message broker, cho phép các service khác đăng ký nhận event khi source publish chúng.

Với tổng quan về event là gì và nên được dùng cho gì, ta có thể liệt kê một số điểm quan trọng cần cân nhắc khi scoping thiết kế phù hợp cho event store:

- Vì event nên bất biến, data store nên là append-only.
- Mỗi thay đổi xảy ra trong domain cần được ghi lại trong event store.
- Vì event cần chứa chi tiết cụ thể liên quan tới event được ghi lại, ta phải duy trì một số tính linh hoạt với cấu trúc dữ liệu.
- Một event có thể chứa dữ liệu từ nhiều nguồn trong domain. Ta phải tận dụng các quan hệ có thể có giữa các bảng để lấy toàn bộ chi tiết cần thiết để log event.

Với các yêu cầu này, ta có thể quyết định dùng data store quan hệ hoặc phi quan hệ. Nếu ta dùng data store quan hệ, ta nên có các bảng denormalized được model từ dữ liệu event ta dự định lưu trữ. Bảng dữ liệu denormalized này sẽ đại diện cho một view tổng hợp của các điểm dữ liệu và dễ đọc hơn thay vì join nhiều bảng để lấy dữ liệu. Một cách khác và hiệu quả hơn để xử lý event storage là dùng database phi quan hệ hoặc NoSQL. Điều này sẽ cho phép ta lưu trữ dữ liệu event liên quan dưới dạng document theo cách linh động hơn nhiều.

### Scoping Event Storage using a Relational Database

Database quan hệ phổ biến vì chúng nghiêm ngặt và dễ enforce tiêu chuẩn chất lượng dữ liệu. Nhược điểm là chúng để lại ít không gian cho tính linh hoạt, có thể lập luận rằng đây là nhu cầu quan trọng nhất khi lưu trữ event.

Một cách giải quyết là implement bảng denormalized, nơi dữ liệu liên quan được lưu trữ dưới dạng đầy đủ chứ không chỉ là ID tham chiếu. Điều này giúp nắm bắt toàn bộ chi tiết của một event và tạo read model chuyên biệt của event. Tuy nhiên, cách tiếp cận này có thể dẫn ta xuống một cái hố thỏ (rabbit hole) của việc model nhiều bảng dựa trên nhiều event. Điều này có thể không bền vững về lâu dài vì ta sẽ cần giới thiệu bảng mới với mỗi event mới được scoping và liên tục thay đổi thiết kế khi event tiến hóa.

Một cách khác là tạo một bảng log duy nhất với các cột khớp các điểm dữ liệu đã nêu trước đó cho một event. Ví dụ, bảng này và các kiểu dữ liệu tương ứng sẽ như sau:

| Column name | Data type |
|---|---|
| Id | int |
| AggregateId | Guid |
| Timestamp | DateTime |
| EventType | Varchar |
| Payload | String/JSON |
| VersionNumber | Int |

Ta có thể thêm ràng buộc cho bản ghi bằng cách giới thiệu một UNIQUE index trên các cột `AggregateId` và `VersionNumber`. `VersionNumber` đại diện cho phiên bản trạng thái của một aggregate. Nó tăng lên mỗi khi một event mới được áp dụng cho aggregate đó. Về mặt khái niệm, con số này phản ánh số lượng thay đổi đã xảy ra. Khi lưu một event mới vào event store, phiên bản aggregate mong đợi được so sánh với phiên bản hiện tại trong lưu trữ.

Điều này sẽ giúp ta có query nhanh hơn và đảm bảo ta không lặp lại bất kỳ tổ hợp nào của các giá trị này.

`Payload` ở đây sẽ là một biểu diễn serialized của dữ liệu liên quan tới bản ghi event. Serialization này tốt nhất nên được thực hiện ở định dạng dễ đọc như JSON. Một số công nghệ database quan hệ cho phép ta query và thao tác JSON, như:

- **PostgreSQL** cung cấp kiểu cột JSON (text-based) và JSONB (binary-optimized JSON). Điều này lý tưởng khi bạn cần đọc/ghi thường xuyên trên field JSON với query phức tạp như filter hoặc partial update.
- **MySQL** cung cấp kiểu cột JSON native. Nó phù hợp cho hệ thống cần query vừa phải và thao tác JSON đơn giản hơn.
- **Microsoft SQL Server** hỗ trợ lưu trữ JSON trong cột NVARCHAR chuẩn trong khi cung cấp function mạnh mẽ để query và thao tác. Nó hỗ trợ thao tác query và update JSON với `JSON_QUERY`, `JSON_VALUE`, và mệnh đề `FOR JSON`.
- **Oracle Database** hỗ trợ kiểu JSON native, phiên bản 21c, hiệu quả cho hệ thống enterprise cần indexing mạnh mẽ.

Loại công nghệ database được dùng đóng vai trò trong việc ta có thể lưu trữ và lấy dữ liệu linh hoạt và hiệu quả tới đâu.

### Scoping Event Storage using a Non-Relational Database

Database phi quan hệ, cụ thể là document database, được đặc trưng bởi khả năng lưu trữ dữ liệu không cấu trúc hiệu quả. Hãy nói về ý nghĩa của điều này:

- Schema linh hoạt và có thể tiến hóa. Cột không bắt buộc trong giai đoạn thiết kế, vì vậy việc mở rộng và thu hẹp dữ liệu dựa trên nhu cầu tức thời khá dễ dàng.
- Kiểu dữ liệu không được implement nghiêm ngặt, vì vậy cấu trúc dữ liệu có thể tiến hóa tại bất kỳ điểm nào mà không ảnh hưởng tiêu cực tới bản ghi đã lưu trước đó.
- Dữ liệu có thể lồng nhau và chứa chuỗi. Điều này quan trọng vì ta không cần trải dữ liệu liên quan trên nhiều document. Một document có thể đại diện cho một biểu diễn denormalized của dữ liệu từ nhiều nguồn.

Ví dụ phổ biến của document store là MongoDB, Microsoft Azure Cosmos DB, và Amazon DynamoDB. Ngoài các yêu cầu query và tích hợp cụ thể cho mỗi tùy chọn database, khái niệm về cách document được hình thành và lưu trữ là giống nhau.

Trong một document data store, dữ liệu được lưu trữ ở định dạng JSON (trừ khi được yêu cầu hoặc implement cụ thể khác). Một mục event sẽ trông như thế này:

```json
{
  "type": "AppointmentCreatedEvent",
  "aggregateId": "aggregateId-guid-value",
  "data": {
    "doctorId": "doctorId-guid-value",
    "customerId": "customerId-guid-value",
    "dateTime": "recorded-date-time"
  },
  "timestamp": "2022-01-01T21:00:46Z"
}
```

Một lợi thế khác khi dùng document data store là ta có thể đại diện hiệu quả hơn cho một bản ghi với lịch sử event của nó trong một materialized view. Điều đó có thể trông như thế này:

```json
{
  "type": "AppointmentCreatedEvent",
  "aggregateId": "aggregateId-guid-value",
  "doctorId": "doctorId-guid-value",
  "customerId": "customerId-guid-value",
  "dateTime": "recorded-date-time",
  "history": [
    {
      "type": "AppointmentCreatedEvent",
      "data": {
        "doctorId": "doctorId-guid-value",
        "customerId": "customerId-guid-value",
        "dateTime": "recorded-date-time"
      },
      "timestamp": "2022-01-01T21:00:46Z"
    },
    {
      "type": "AppointmentUpdatedEvent",
      "data": {
        "doctorId": "different-doctorId-guid-value",
        "customerId": "customerId-guid-value",
        "comment": "Update comment here"
      },
      "timestamp": "2022-01-01T21:00:46Z"
    }
  ],
  "createdDate": "2022-01-01T21:00:46Z"
}
```

Biểu diễn dữ liệu này có thể có lợi cho việc lấy một bản ghi với toàn bộ event của nó, đặc biệt nếu ta dự định hiển thị dữ liệu này trên UI. Ta đã tạo một view đóng vai trò vừa là snapshot của trạng thái hiện tại của dữ liệu aggregate vừa là luồng event đã ảnh hưởng tới nó.

Ta muốn event store của mình performant nhất có thể, vì vậy bạn có thể cân nhắc thêm compound và text index vào schema để tối ưu hiệu năng query và dùng partition để scale horizontal khi có tải quá mức. Bạn cũng có thể dùng chiến lược event versioning và schema migration để quản lý thay đổi đối với cấu trúc event trong database NoSQL.

Các tối ưu này rất quan trọng để xử lý luồng event lớn. Để quản lý điều này hiệu quả, cân nhắc lưu trữ event trong collection hoặc bảng đã partition dùng Aggregate ID làm partition key. Cách tiếp cận này đảm bảo query cho một aggregate duy nhất được scope tới một partition cụ thể, giảm read footprint. Bạn cũng có thể implement cơ chế đọc event theo trang (paginated) để stream event theo từng khối, cải thiện memory efficiency.

---

## Phần 5: Implementing an Event Store with PostgreSQL

Ta sẽ dùng PostgreSQL để tạo một event store cho event liên quan tới việc tạo lịch hẹn. Ta sẽ implement database mới này dùng code-first và EF Core. Để hỗ trợ PostgreSQL, ta cần package Npgsql. Chạy lệnh sau ở terminal tại thư mục gốc của project:

```bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

Bây giờ, ta có thể định nghĩa một data model mới trong folder Data gọi là `EventEntity`. Nó sẽ trông như thế này:

```csharp
public class EventEntity
{
    [Key]
    public Guid EventId { get; set; }
    public Guid AggregateId { get; set; }
    public string EventType { get; set; }
    public DateTime EventTimestamp { get; set; }
    [Column(TypeName = "jsonb")]
    public string Payload { get; set; }
}
```

Ta chú thích property `EventId` để chỉ ra nó đóng vai trò khóa chính của bảng và bao gồm chú thích EF Core trên cột `Payload` để map với kiểu cột PostgreSQL JSONB. Bây giờ, ta có thể định nghĩa một data context mới đại diện cho database event. Ta gọi nó là `EventStoreDbContext` và định nghĩa như sau:

```csharp
public class EventStoreDbContext(DbContextOptions<EventStoreDbContext> options)
    : DbContext(options)
{
    public DbSet<EventEntity> Events { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<EventEntity>().ToTable("appointment_events");
    }
}
```

Bây giờ, bạn có thể cập nhật file `appsettings.json` với dòng sau, điền chi tiết server, username, và password cụ thể của bạn. Trong workload production, lưu secret trong biến môi trường hoặc secret vault:

```json
{
  "ConnectionStrings": {
    "EventStoreDatabase": "Host=localhost;Database=event_store;Username=yourusername;Password=yourpassword"
  }
}
```

Ta cũng sẽ thêm cấu hình vào file `Program.cs`:

```csharp
builder.Services.AddDbContext<EventStoreDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("EventStoreDatabase")));
```

Ta sẽ tiếp tục với một data store chỉ phục vụ microservice đặt lịch hẹn để đơn giản hóa. Để tạo database, ta cần phát triển và thực thi một migration. Bạn có thể làm điều này với các lệnh sau:

```bash
dotnet ef migrations add InitializeEventStoreDatabase --context EventStoreDbContext
dotnet ef database update --context EventStoreDbContext
```

---

## Phần 6: Implementing Event-Sourcing Patterns

Event sourcing được implement để cung cấp một điểm tham chiếu duy nhất cho lịch sử những gì đã xảy ra trong một aggregate, và mở rộng ra, một bounded context. Event có thể được dùng trong việc implement pattern này để giúp ta model kết quả mong đợi trong bounded context của ta. Event được scope dựa trên ubiquitous language đã thiết lập trong bounded context và được thông tin bởi quyết định trong domain.

Trong domain, aggregate chịu trách nhiệm tạo domain event, và domain event của ta thường được raise dựa trên kết quả của một hành động hoặc command của người dùng. Quan trọng cần lưu ý rằng domain event không được raise dựa trên các hành động như:

- Button click, di chuyển chuột, sự kiện cuộn trang, hoặc exception ứng dụng đơn giản. Event nên dựa trên ubiquitous language đã thiết lập của bounded context.
- Event từ hệ thống khác hoặc bên ngoài context hiện tại. Điều thiết yếu là thiết lập đúng ranh giới giữa mỗi domain context.
- Yêu cầu người dùng đơn giản tới hệ thống. Tại điểm này, một yêu cầu người dùng là một command. Event được raise dựa trên kết quả của command.

### Domain Events and Event Sourcing

Event sourcing dùng domain event để lưu trữ các trạng thái mà một aggregate đã trải qua. Implement domain event trong code có thể tương đối đơn giản dùng thư viện MediatR, thư viện đã đóng vai trò không thể thiếu trong implementation CQRS pattern của ta.

### Implementing Domain Events trong ứng dụng

Domain event trong microservice đặt lịch hẹn của ta đại diện cho các hành động hoặc quyết định quan trọng trong domain, như lên lịch, đổi lịch, hoặc hủy lịch hẹn. Ta phải lưu trữ các event này theo trình tự để implement event sourcing, cho phép tái tạo trạng thái và một audit trail hoàn chỉnh. Ta có thể mở rộng implementation MediatR của mình để publish và xử lý các event này.

Bạn có thể định nghĩa các class sau trong một folder Events mới trong folder Models hiện có:

```csharp
public class AppointmentCreatedEvent : INotification
{
    public Guid AppointmentId { get; set; }
    public DateTime AppointmentDate { get; set; }
    public Guid PatientId { get; set; }
    public Guid DoctorId { get; set; }
    public TimeSlot Slot { get; set; }
    public Location Location { get; set; }
}

public class AppointmentCancelledEvent : INotification
{
    public Guid AppointmentId { get; set; }
    public string Reason { get; set; }
    public DateTime CancelledAt { get; set; }
}
```

Cả hai model đều kế thừa từ `INotification`, một interface từ MediatR, đại diện cho một notification message trong pub-sub pattern. Nó chủ yếu được dùng để định nghĩa domain event hoặc message không cần phản hồi nhưng được broadcast tới nhiều handler có thể hành động dựa trên chúng. Mỗi notification có thể có một hoặc nhiều handler phản hồi khi nó được publish. Handler implement interface `INotificationHandler<T>`, với T là kiểu notification.

Bây giờ, ta định nghĩa handler sau để dispatch cảnh báo email tới bệnh nhân:

```csharp
public class NotifyEmailAppointmentCreatedHandler(IEmailService _emailService) :
    INotificationHandler<AppointmentCreatedEvent>
{
    public async Task Handle(AppointmentCreatedEvent notification,
        CancellationToken cancellationToken)
    {
        // Send Email notifications
        return Task.CompletedTask;
    }
}
```

Kiểu event của ta, `AppointmentCreatedEvent`, được dùng làm kiểu target cho `INotificationHandler`. Nó cũng giúp ta tách biệt mối quan tâm và cô lập tốt hơn các đoạn code liên quan tới event đã raise. Tương tự, nếu ta muốn định nghĩa một event handler cho thao tác SignalR, ta có thể định nghĩa handler thứ hai cho cùng kiểu event:

```csharp
public class NotifySignalRHubsAppointmentCreatedHandler :
    INotificationHandler<AppointmentCreatedEvent>
{
    public Task Handle(AppointmentCreated notification,
        CancellationToken cancellationToken)
    {
        // SignalR awesomeness here
        return Task.CompletedTask;
    }
}
```

Với điều chỉnh này, ta có thể publish một event từ action POST trong API controller như sau:

```csharp
public async Task<ActionResult<Appointment>> PostAppointment(
    CreateAppointmentCommand createAppointmentCommand)
{
    var appointment = await _mediator.Send(createAppointmentCommand);
    await _mediator.Publish(new AppointmentCreatedEvent
    {
        AppointmentId = appointment.AppointmentId,
        AppointmentDate = appointment.Slot.Start,
        PatientId = appointment.PatientId,
        Location = appointment.Location,
        Slot = appointment.Slot
    });
    return CreatedAtAction("GetAppointment", new { id = appointment.AppointmentId }, appointment);
}
```

Bây giờ, ta dùng method `Publish` từ thư viện MediatR để raise một event, và toàn bộ handler được định nghĩa để theo dõi kiểu event đã chỉ định được gọi thực hiện.

Ta vẫn cần implement một handler lưu trữ dữ liệu liên quan tới event. Ta có thể tạo một handler mới gọi là `SaveAppointmentCreatedEventDetailsHandler`, sẽ tìm thêm chi tiết về appointment và lưu event trong bảng database Postgres của ta:

```csharp
public class SaveAppointmentCreatedEventDetailsHandler(
    EventStoreDbContext _eventStoreDbContext /* Additional dependencies */)
    : INotificationHandler<AppointmentCreatedEvent>
{
    public async Task Handle(AppointmentCreatedEvent notification,
        CancellationToken cancellationToken)
    {
        // Use the clients to fetch document, patient and doctor details
        // shortened for brevity
        AppointmentDetails appointmentDetails = new(
            /* Detail properties as needed */
        );
        // Save the details to the event store
        EventEntity @event = new()
        {
            AggregateId = notification.AppointmentId,
            EventId = Guid.NewGuid(),
            EventType = nameof(AppointmentCreatedEvent),
            Payload = JsonSerializer.Serialize(appointmentDetails), // serialize data as JSON
            EventTimestamp = DateTime.UtcNow
        };
        _eventStoreDbContext.Events.Add(@event);
        await _eventStoreDbContext.SaveChangesAsync();
    }
}
```

Bảng event của ta sẽ được cập nhật với thông tin liên quan ở background mỗi khi ta tạo một appointment mới.

Các kịch bản khác bạn có thể cân nhắc bao gồm:

- **AppointmentCancelledEvent**: cân nhắc một flag trên entity Appointment chỉ ra trạng thái của nó (pending, completed, canceled, v.v.) và log mỗi thay đổi.
- **AppointmentUpdatedEvent**: điều gì nếu bệnh nhân yêu cầu giờ mới hoặc địa điểm/bác sĩ chỉ định thay đổi?

Mỗi kiểu event phải bao gồm một phiên bản payload của bản ghi đang được thao tác. Bằng cách này, ta luôn có thể biết trạng thái của bản ghi tại thời điểm đó.

Hãy nhớ rằng toàn bộ điều này xảy ra trong cùng một transaction. Trong khi event trong CQRS có thể được xem như một tùy chọn thay thế tiềm năng cho giao tiếp bất đồng bộ (ví dụ, gửi email), thao tác diễn ra real-time. Chúng có thể gây gánh nặng cho API với tính toán bổ sung ở background. Hãy chủ động trong việc implement event và quyết định điều gì có thể giao cho service bên ngoài và message queue so với điều gì cần được xử lý nội bộ.

### Using Events for Read-Only Operations

Bản ghi lưu trong event store có thể cung cấp năng lượng cho các query chỉ đọc trong hệ thống. Bằng cách tận dụng bản chất linh hoạt của event, ta có thể định hình payload để lưu đủ dữ liệu đáp ứng một số nhu cầu query của ta.

Payload ta đang lưu trữ được model từ thông tin chi tiết trả về khi endpoint `GetAppointment` được gọi. Do đó, ta có thể cân nhắc refactor `GetAppointmentByIdHandler` để dùng event store trả về chi tiết và loại bỏ nhu cầu cho nhiều cuộc gọi API. Nó sẽ trông như thế này:

```csharp
public class GetAppointmentByIdHandler(EventStoreDbContext _eventStoreDbContext)
    : IRequestHandler<GetAppointmentByIdQuery, AppointmentDetails>
{
    public async Task<AppointmentDetails> Handle(
        GetAppointmentByIdQuery request, CancellationToken cancellationToken)
    {
        var appointmentEvent = await _eventStoreDbContext.Events
            .Where(e => e.AggregateId.ToString() == request.Id)
            .OrderByDescending(e => e.EventTimestamp)
            .FirstOrDefaultAsync();
        return appointmentEvent == null ? null
            : JsonSerializer.Deserialize<AppointmentDetails>(appointmentEvent.Payload);
    }
}
```

Handler đã refactor của ta giờ dựa vào event store, giúp ta implement khái niệm cơ bản của việc tách biệt data store đọc và ghi. Ta cũng có thể dựa vào kiểu event để giúp lọc event ta muốn trả về:

```csharp
public class GetAppointmentsByPatientIdHandler(EventStoreDbContext _eventStoreDbContext)
    : IRequestHandler<GetAppointmentByPatientIdQuery, List<AppointmentByPatientId>>
{
    public async Task<List<AppointmentByPatientId>> Handle(
        GetAppointmentByPatientIdQuery request, CancellationToken cancellationToken)
    {
        var patientJson = JsonSerializer.Serialize(new { Patient = new { request.PatientId } });
        var events = await _eventStoreDbContext.Events
            .Where(e => e.EventType == "AppointmentCreatedEvent" &&
                        EF.Functions.JsonContains(e.Payload, patientJson))
            .AsNoTracking()
            .ToListAsync();

        List<AppointmentByPatientId> appointments = [];
        foreach (var @event in events)
        {
            var appointment = JsonSerializer.Deserialize<AppointmentDetails>(@event.Payload);
            if (appointment != null && appointment.StartTime < DateTime.UtcNow)
                continue;
            appointments.Add(new AppointmentByPatientId(
                appointment.AppointmentId,
                appointment.Doctor.FirstName + " " + appointment.Doctor.LastName,
                appointment.StartTime
            ));
        }
        return appointments.OrderBy(q => q.Date).ToList();
    }
}
```

Ta bắt đầu bằng cách hình thành một biểu diễn JSON của cặp key-value Patient-PatientId, như nó sẽ xuất hiện trong payload ta đã lưu trữ. Sau đó, ta query event store cho toàn bộ bản ghi có payload chứa chuỗi cặp key-value đó. `EF.Functions.JsonContains` là một method EF Core độc đáo cho phép ta truy vấn hiệu quả một cột JSONB trong PostgreSQL cho các khớp cụ thể.

Nhược điểm rõ ràng nhất trong ví dụ này là ta phải load toàn bộ event để tìm những cái có ngày trong tương lai. Đây không phải kỹ thuật query hiệu quả nhất, nhưng ta bị giới hạn bởi thực tế rằng cột JSONB có hỗ trợ query và filter hạn chế. Ngoài ra, event store được tối ưu cho thao tác ghi. Dùng nó cho query đọc phức tạp có thể dẫn tới nghẽn hiệu năng. Read model phù hợp hơn cho thao tác đọc.

---

## Phần 7: Best Practices and Recommendations

### Designing Events with Care

Event là nền tảng của event sourcing, vì vậy thiết kế chúng cẩn thận là điều thiết yếu. Mỗi event nên đại diện cho một thay đổi có ý nghĩa, bất biến trong domain. Ví dụ, thay vì lưu trữ thay đổi generic như `AppointmentUpdated`, dùng event cụ thể như `AppointmentScheduled`, `AppointmentRescheduled`, hoặc `AppointmentCancelled`. Để đảm bảo forward compatibility, bao gồm số phiên bản trong mỗi event và cân nhắc cách xử lý sự tiến hóa schema.

Backward compatibility đảm bảo phiên bản mới của ứng dụng bạn vẫn có thể hiểu và xử lý event được tạo bởi phiên bản hệ thống cũ hơn. Forward compatibility cho phép phiên bản cũ hơn của hệ thống bỏ qua hoặc xử lý an toàn event được sinh bởi phiên bản mới hơn. Khi giới thiệu field mới, đảm bảo thay đổi có tính cộng thêm, không xóa hoặc đổi tên field hiện có, và giá trị mặc định được cung cấp cho consumer cũ hơn. Event processor nên chấp nhận dữ liệu thiếu và bỏ qua field lạ mà chúng không nhận diện.

Một cách tiếp cận phổ biến để hỗ trợ compatibility là version schema event của bạn một cách tường minh, bằng cách bao gồm một field Version trong event object, cho phép event processor chọn logic đúng dựa trên số phiên bản.

Event nên được lưu trữ atomic cùng với thay đổi đối với database chính, đảm bảo hệ thống vẫn nhất quán bất chấp lỗi. Event handler cũng phải idempotent để tránh side effect ngoài ý muốn khi event được replay hoặc xử lý nhiều lần.

Đảm bảo event được viết với kiểm tra version bên trong một database transaction. Một pattern điển hình là load phiên bản aggregate hiện tại, so sánh với phiên bản mong đợi, và chỉ insert event mới nếu các phiên bản khớp nhau, tất cả trong một transaction duy nhất — đây là kỹ thuật **optimistic concurrency**.

### Using Snapshots for Performance Optimization

Đọc hàng ngàn event có thể tốn kém. Snapshot cung cấp một cơ chế để lưu trữ trạng thái của một aggregate tại một thời điểm cụ thể. Khi tái dựng trạng thái, bắt đầu từ snapshot gần nhất và chỉ replay các event sau đó. Điều này giảm đáng kể chi phí tính toán.

Model EF Core cho snapshot:

```csharp
public class AggregateSnapshot
{
    public Guid AggregateId { get; set; }
    public int Version { get; set; }
    public string PayloadSnapshot { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

Thiết kế từng bước của cơ chế snapshotting:

1. **Định nghĩa cấu trúc snapshot**: định nghĩa bảng database lưu snapshot trạng thái aggregate.
2. **Xác định tần suất snapshot**: quyết định khi nào tạo snapshot (VD: sau một số lượng event nhất định).
3. **Implement việc tạo snapshot**: theo dõi số lượng event kể từ snapshot cuối cùng, sinh snapshot khi đạt ngưỡng.
4. **Restore từ snapshot**: khi load một aggregate, lấy snapshot gần nhất trước, rồi áp dụng event xảy ra sau đó.

Snapshot thường xuyên hơn tăng tốc đáng kể tái dựng trạng thái, đặc biệt hữu ích cho thao tác nhạy cảm về thời gian như hiển thị trạng thái lịch hẹn hiện tại hoặc sinh báo cáo real-time. Tuy nhiên, tạo snapshot quá thường xuyên gây tiêu thụ lưu trữ quá mức, làm phình to kích thước database, ảnh hưởng thời gian backup/recovery, và cần chiến lược cleanup snapshot lỗi thời.

### Choosing an Appropriate Data Store and ORM

PostgreSQL với JSONB cung cấp phương pháp lưu trữ toàn diện. Kết hợp với EF Core, có giới hạn về cách query có thể interrogate nội dung payload. Cách khác, cân nhắc dùng hệ thống chuyên biệt như EventStoreDB, được tinh chỉnh riêng cho lưu trữ event, có sẵn indexing tốt trên các field thường được query. Tuy nhiên, cần thời gian học API riêng và có cộng đồng nhỏ hơn.

Bằng cách giữ read-check và insert trong cùng một transaction, không writer nào khác có thể can thiệp giữa việc đọc phiên bản hiện tại và viết event mới. Cần cột version được index trong bảng event, bao gồm trong primary key hoặc uniqueness constraint.

Ngoài ra, một retention policy nên được cân nhắc, nơi độ dài của event trong event store chính bị giới hạn. Không phải toàn bộ event lịch sử cần được giữ vô thời hạn — sau một khoảng thời gian, event cũ hơn có thể được archive vào giải pháp lưu trữ tiết kiệm chi phí hơn (blob storage, data lake, cold-tier database), giữ event store chính gọn nhẹ và nhanh trong khi bảo tồn dữ liệu lịch sử cho audit/phân tích tương lai.

Cuối cùng, đừng bỏ qua testing. Unit test xác thực hành vi của domain aggregate và khả năng áp dụng và sinh event đúng cách. Integration test đảm bảo event có thể được lưu trữ và lấy chính xác từ event store bên dưới, verify tương tác với database bao gồm kiểm soát version, serialize/deserialize, và tính toàn vẹn transaction.

---

## Tổng kết chương

Chương này khám phá kỹ lưỡng event sourcing pattern, nhấn mạnh vai trò của nó trong kiến trúc phần mềm hiện đại, đặc biệt trong microservices. Event sourcing ghi lại toàn bộ thay đổi trạng thái dưới dạng một chuỗi event bất biến, cung cấp insight về sự tiến hóa của dữ liệu và cho phép các tính năng vững chắc như audit trail, tái tạo trạng thái, và xử lý real-time.

Một event ghi lại một thay đổi trong hệ thống, sở hữu các thuộc tính chính như định danh duy nhất, timestamp, và payload, khiến chúng bất biến và đáng tin cậy. Tính bất biến này đảm bảo tính toàn vẹn dữ liệu và hỗ trợ reproducibility, auditability, và decoupling giữa các thành phần.

Ta đã triển khai thực tế một event store dùng PostgreSQL + EF Core, kết hợp với domain event qua MediatR (`INotification`/`Publish` — khác biệt quan trọng so với `IRequest`/`Send` của CQRS đã học ở Chapter 6), và dùng chính event store đó để phục vụ thao tác đọc trong CQRS — hoàn tất vòng lặp khép kín giữa 2 chương liên tiếp.

Cuối cùng, ta học được các best practice quan trọng: thiết kế event cụ thể và có ý nghĩa nghiệp vụ, đảm bảo forward/backward compatibility qua versioning, giữ consistency + idempotency + optimistic concurrency control trong transaction, dùng snapshot để tối ưu hiệu năng khi event stream lớn, chọn đúng data store/ORM, và áp dụng retention policy/archiving cho event cũ.

**Bài học cốt lõi**: Event Sourcing là giải pháp mạnh mẽ cho vấn đề data consistency trong hệ thống phân tán, nhưng đi kèm độ phức tạp đáng kể — cần cân nhắc kỹ lưỡng giống mọi pattern khác đã học xuyên suốt cuốn sách này, và chỉ nên áp dụng khi lợi ích (audit trail, reproducibility, real-time processing) thực sự vượt trội hơn chi phí vận hành và độ phức tạp phát sinh.
