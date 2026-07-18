# Bài học: Giao tiếp Bất đồng bộ — Khi service không cần chờ nhau nữa

> Chương 4 — Asynchronous Communication Between Microservices
> Ví dụ xuyên suốt: AppointmentsApi (publisher) phát sự kiện `AppointmentCreated` qua RabbitMQ, một consumer (notification) nhận và gửi email xác nhận

---

## Phần 1: Vấn đề thực tế — luồng đặt lịch khám đang "chờ" quá nhiều

Ở chương trước, ta đã khám phá giao tiếp đồng bộ giữa các microservice và các use case cụ thể. Giờ ta chuyển sang giao tiếp bất đồng bộ — yếu tố then chốt giúp đạt được khả năng mở rộng, khả năng phục hồi, và tính linh hoạt trong hệ thống phân tán.

Khác với giao tiếp đồng bộ dựa vào tương tác trực tiếp, thời gian thực giữa các service, giao tiếp bất đồng bộ cho phép các service tương tác mà không đòi hỏi phản hồi ngay lập tức. Sự tách rời (decoupling) này đảm bảo mỗi service có thể hoạt động độc lập, tăng cường độ vững chắc tổng thể của hệ thống.

Có những lúc một microservice cần kích hoạt một tác vụ ở microservice khác để hoàn tất một thao tác, nhưng kết quả của tác vụ đó không cần được biết ngay. Ví dụ điển hình: gửi email xác nhận sau khi hệ thống quản lý phòng khám đã đặt lịch hẹn thành công. Hãy hình dung người dùng phải chờ UI hoàn tất thao tác loading để thấy xác nhận. Giữa lúc bấm Submit và thấy xác nhận, booking service cần hoàn tất các thao tác:

- Tạo bản ghi lịch hẹn
- Gửi email cho bệnh nhân đã đặt lịch
- Tạo mục lịch (calendar entry) cho hệ thống

Dù đã cố gắng hết sức, việc booking service cố hoàn tất các thao tác này sẽ tốn thời gian và có thể dẫn tới trải nghiệm người dùng kém hấp dẫn hơn. Ta cũng có thể lập luận rằng trách nhiệm gửi email không nên thuộc về bản chất của booking service. Tương tự, quản lý lịch nên đứng độc lập. Vì vậy, ta có thể refactor các tác vụ của booking service thành:

- Tạo bản ghi lịch hẹn
- Gọi đồng bộ tới doctors service để lấy chi tiết bác sĩ
- Gọi đồng bộ tới patients service để lấy chi tiết bệnh nhân
- Gọi đồng bộ tới Email service để gửi email cho bệnh nhân đã đặt lịch
- Gọi đồng bộ tới calendar management service để tạo mục lịch cho hệ thống

Ta đã refactor booking service để thực hiện ít thao tác hơn, đẩy các thao tác không liên quan trực tiếp tới việc đặt lịch sang các service khác. Đây là một refactor tốt, nhưng ta vẫn giữ lại — thậm chí có thể khuếch đại — lỗ hổng chính trong thiết kế này. Ta vẫn sẽ chờ hoàn tất một thao tác có khả năng tốn thời gian trước khi chuyển sang thao tác tiếp theo, và thao tác đó cũng mang cùng rủi ro. Tại đây, ta cần cân nhắc mức độ quan trọng của việc chờ phản hồi cho các hành động này so với việc chỉ cần nhập bản ghi đặt lịch vào database — đây là thao tác quan trọng nhất để người dùng biết được kết quả của quá trình đặt lịch.

Trong bối cảnh này, ta có thể dùng pattern giao tiếp bất đồng bộ để đảm bảo thao tác chính được hoàn tất, còn các thao tác khác như gửi email và ghi mục lịch có thể xảy ra dần dần mà không ảnh hưởng tới trải nghiệm người dùng.

### Analogy: Gọi điện thoại vs Gửi thư (tiếp nối Chapter 3)

Nếu đồng bộ giống gọi điện thoại — phải chờ trên đường dây — thì bất đồng bộ giống **gửi thư qua bưu điện**: bạn bỏ thư vào hòm thư (message queue), rồi đi làm việc khác ngay. Người nhận sẽ đọc và xử lý thư khi họ rảnh. Bạn không cần biết chính xác khi nào họ đọc — chỉ cần tin tưởng rằng thư sẽ tới.

---

## Phần 2: Chiến lược cho các service bất đồng bộ

Triển khai giao tiếp bất đồng bộ trong ứng dụng microservices là yếu tố then chốt để đạt được khả năng mở rộng, phục hồi, và linh hoạt. Bằng cách tách rời các service và cho phép tương tác không chặn (non-blocking), hệ thống có thể xử lý tải thay đổi và duy trì khả năng phản hồi.

Giao tiếp bất đồng bộ trong microservices có thể đạt được qua nhiều pattern và giao thức:

### 1. Message Queues (Hàng đợi tin nhắn)

Message broker tạo điều kiện giao tiếp giữa các service. Service gửi message vào queue, sau đó được xử lý bởi service tiêu thụ (consuming service). Cách tiếp cận này tách rời các service và cho phép xử lý message linh hoạt. Ví dụ, Amazon Simple Queue Service (SQS) là dịch vụ hàng đợi message được quản lý toàn diện, cho phép giao tiếp tách rời giữa các thành phần phân tán. Một lợi ích của các dịch vụ này là chúng là hàng đợi FIFO (First-in-First-out) — bảo toàn đúng thứ tự message và đảm bảo mỗi message được xử lý đúng một lần. Hàng đợi FIFO đặc biệt hữu ích trong ứng dụng tài chính, hệ thống xử lý đơn hàng, hoặc bất kỳ kịch bản nào cần bảo toàn trình tự sự kiện.

Các implementation message queue khác bao gồm RabbitMQ, Azure Queue Storage, và ActiveMQ — mỗi cái cung cấp các tính năng khác nhau liên quan tới đảm bảo gửi (delivery guarantee), thứ tự (ordering), throughput, và khả năng tích hợp.

**Chi tiết về Message Queue**: một message queue thường được implement như cầu nối giữa hai hệ thống — publisher và consumer. Khi publisher đặt message vào queue, consumer xử lý thông tin trong message ngay khi có sẵn. Queue enforce phương thức phân phối FIFO, đảm bảo thứ tự xử lý luôn được đảm bảo. Điều này thường được implement theo mô hình giao tiếp một-đối-một, mỗi ứng dụng đăng ký (subscribing application) sẽ có một queue riêng được cung cấp.

Hệ thống thanh toán thường sử dụng pattern này nhiều, nơi trình tự thực tế của các khoản thanh toán được gửi có ý nghĩa quan trọng, và cần đảm bảo tính bền vững của các chỉ thị. Nếu suy nghĩ kỹ, hệ thống thanh toán thường có tỷ lệ lỗi rất thấp. Phần lớn thời gian, khi ta gửi một yêu cầu thanh toán, ta có thể yên tâm rằng nó sẽ được hoàn tất thành công tại một thời điểm nào đó trong tương lai.

Với message queue, consumer thường hoạt động theo mô hình polling — được thiết kế để kiểm tra message theo khoảng thời gian định trước, và khi có message, sẽ cố xử lý nó. Là thực hành tốt để xóa message khỏi queue khi có nhiều consumer cho cùng một queue, vì điều này giúp ngăn cùng một message bị nhiều consumer khác nhau lấy, dẫn tới khả năng xử lý trùng lặp.

Dù message queue có công dụng riêng và mang lại độ tin cậy nhất định cho thiết kế hệ thống, chúng có thể kém hiệu quả và đưa vào một số nhược điểm ta cần chuẩn bị sống chung trong một hệ thống phân tán. Cũng có những tình huống nhiều consumer cần một bản sao của cùng một message. Trong trường hợp đó, ta tập trung vào một thiết lập messaging phân tán hơn, như message bus.

### 2. Publish-Subscribe (Pub-Sub) Model

Service publish message tới các topic, và nhiều subscriber nhận các message này. Mô hình này phù hợp cho kiến trúc hướng sự kiện (event-driven architecture), nơi nhiều service cần phản ứng với các sự kiện cụ thể.

### 3. Event Streaming

Triển khai các luồng sự kiện liên tục mà service có thể tiêu thụ theo thời gian thực. Phương pháp này phù hợp cho ứng dụng cần xử lý ngay lập tức các luồng dữ liệu.

Message queue cho phép giao tiếp tách rời giữa các service bằng cách lưu trữ message cho tới khi consumer xử lý chúng. Việc chọn công nghệ phù hợp phụ thuộc vào yêu cầu ứng dụng cụ thể, như throughput message, độ trễ, khả năng mở rộng, và khả năng tích hợp. Đánh giá các yếu tố này đảm bảo giải pháp được chọn phù hợp với mục tiêu kiến trúc và ràng buộc vận hành của hệ thống.

### Analogy: Queue giống hàng chờ khám bệnh, Pub-Sub giống loa phát thanh bệnh viện

Message Queue giống hàng người xếp hàng chờ khám — mỗi bệnh nhân (message) được khám (xử lý) đúng một lần, đúng thứ tự, bởi đúng một bác sĩ (consumer). Pub-Sub giống hệ thống loa phát thanh trong bệnh viện: khi có thông báo "phòng mổ 3 sẵn sàng", **tất cả** các bộ phận liên quan (điều dưỡng, hộ lý, bác sĩ gây mê) đều nghe được cùng lúc và tự hành động theo vai trò của mình.

---

## Phần 3: Hiểu về Message Bus Systems

Một message, event, hoặc service bus cung cấp giao diện nơi nhiều subscriber độc lập hoặc cạnh tranh có thể xử lý một message đã publish. Điều này có lợi khi publish cùng một dữ liệu tới nhiều ứng dụng hoặc service. Ta không cần kết nối tới nhiều queue để gửi một message; ta có thể có một kết nối, hoàn tất một hành động gửi, và không cần lo lắng về phần còn lại.

Quay lại kịch bản đặt lịch hẹn có nhiều mối quan tâm vận hành, ta có thể dùng message bus để phân phối dữ liệu tới các service liên quan và cho phép chúng hoàn tất thao tác theo thời gian riêng của mình. Thay vì gọi trực tiếp tới các API khác, appointment booking API sẽ tạo một message và đặt nó lên message bus. Email service và calendar service đăng ký message bus, xử lý message tương ứng.

Pattern này có nhiều lợi ích, bao gồm tách rời (decoupling) và khả năng mở rộng ứng dụng. Điều này giúp các microservice trở nên độc lập hơn với nhau và giảm giới hạn liên quan tới việc thêm nhiều service trong tương lai. Nó cũng tăng tính ổn định cho tương tác tổng thể giữa các service, vì message bus đóng vai trò trung gian lưu trữ dữ liệu cần thiết để hoàn tất một thao tác. Nếu một consuming service không khả dụng, message sẽ được giữ lại, và các message đang chờ sẽ được xử lý. Nếu tất cả service đang chạy và message đang bị dồn ứ (backing up), ta có thể scale số lượng instance của các service để giảm backlog message nhanh hơn.

### Command Message vs Event Message

Có nhiều loại message, nhưng chương này tập trung vào hai loại: **command message** và **event message**.

- **Command message** yêu cầu một hành động cụ thể được thực hiện. Vì vậy, message tới calendar services gửi chỉ thị để tạo một mục lịch. Với bản chất của các command này, ta có thể tận dụng pattern bất đồng bộ và để message được xử lý dần theo thời gian. Cách này giúp ngay cả một lượng lớn message cũng được xử lý.

- **Event message** đơn giản hơn là thông báo rằng một hành động đã xảy ra. Vì các message này được tạo sau khi hành động xảy ra, chúng dùng thì quá khứ và có thể gửi tới nhiều microservice. Trong trường hợp này, message tới email service có thể xem như một event, và email service sẽ chuyển tiếp thông tin đó tương ứng. Loại message này thường chỉ chứa đủ thông tin để cho các service tiêu thụ biết hành động nào đã hoàn tất.

Command message thường nhắm tới các microservice cần post hoặc sửa đổi dữ liệu. Message này phải được tiêu thụ và thực hiện trước khi dữ liệu mong đợi khả dụng trong một khoảng thời gian. Đây là một trong những nhược điểm chính của mô hình messaging bất đồng bộ, được gọi là eventual consistency.

---

## Phần 4: Eventual Consistency — đánh đổi bắt buộc phải chấp nhận

Một trong những thách thức lớn nhất trong thiết kế microservices là quản lý dữ liệu và phát triển chiến lược để giữ dữ liệu đồng bộ. Đôi khi điều này có nghĩa là có nhiều bản sao của cùng dữ liệu trong nhiều database microservice khác nhau. Eventual consistency là khái niệm trong hệ thống điện toán phân tán chấp nhận rằng dữ liệu sẽ không đồng bộ trong một khoảng thời gian. Loại ràng buộc này chỉ chấp nhận được trong hệ thống phân tán và ứng dụng có khả năng chịu lỗi (fault-tolerant).

Việc quản lý một tập dữ liệu trong một database khá dễ dàng, như trường hợp ứng dụng monolithic, vì dữ liệu luôn cập nhật cho bất kỳ phần nào khác của ứng dụng truy cập. Cách tiếp cận một database đảm bảo transaction ACID, nhưng ta vẫn đối mặt với thách thức quản lý concurrency. Concurrency đề cập tới việc có thể có nhiều phiên bản của cùng dữ liệu khả dụng tại các thời điểm khác nhau. Thách thức này dễ quản lý hơn trong ứng dụng một database duy nhất, nhưng đặt ra thách thức riêng trong hệ thống phân tán.

Có thể hợp lý khi giả định rằng khi dữ liệu thay đổi trong một microservice, dữ liệu sẽ thay đổi ở microservice khác, dẫn tới một số không nhất quán giữa các data store trong một khoảng thời gian.

### CAP Theorem — nền tảng lý thuyết đứng sau quyết định này

Định lý CAP giới thiệu khái niệm rằng ta không thể đảm bảo cả 3 thuộc tính chính của một hệ thống phân tán, đó là consistency, availability, và partition tolerance:

- **Consistency**: mỗi thao tác đọc từ data store sẽ trả về phiên bản dữ liệu hiện tại và mới nhất, hoặc lỗi sẽ được nêu ra nếu hệ thống không thể đảm bảo đó là phiên bản mới nhất.
- **Availability**: dữ liệu sẽ luôn được trả về cho thao tác đọc, ngay cả khi đây không phải phiên bản mới nhất được đảm bảo.
- **Partition tolerance**: nghĩa là hệ thống sẽ vận hành ngay cả khi có lỗi tạm thời mà bình thường có thể dừng hệ thống. Hãy tưởng tượng nếu có vấn đề kết nối nhỏ giữa các service và hệ thống messaging. Cập nhật dữ liệu sẽ bị trì hoãn, nhưng nguyên tắc này gợi ý ta phải chọn giữa availability và consistency.

Chọn consistency hay availability là quyết định quan trọng khi tiến về phía trước. Vì microservices thường luôn cần khả dụng, ta phải cẩn trọng với các lựa chọn và mức độ áp đặt nghiêm ngặt các ràng buộc xung quanh chính sách consistency của hệ thống: eventual hay strong.

Có những kịch bản nơi strong consistency là tùy chọn, vì toàn bộ công việc thực hiện bởi một thao tác được hoàn tất hoặc rollback. Các cập nhật này hoặc bị mất (nếu rollback) hoặc được lan truyền tới các microservice khác theo thời gian riêng mà không ảnh hưởng tiêu cực tới hoạt động tổng thể của ứng dụng. Nếu chọn mô hình này, ta có thể đánh giá trải nghiệm người dùng thông qua việc thông báo cho họ biết rằng cập nhật không phải lúc nào cũng tức thời trên các màn hình/module khác nhau.

Dùng mô hình pub-sub là cách số một để implement kiểu giao tiếp hướng sự kiện này giữa các service, nơi tất cả giao tiếp qua một messaging bus. Sau mỗi thao tác hoàn tất, mỗi microservice publish một event message lên message bus, mà các service khác cuối cùng sẽ lấy và xử lý.

Ta thường dùng giao tiếp hướng sự kiện qua mô hình pub-sub để implement eventual consistency. Khi dữ liệu được cập nhật ở một microservice, ta có thể publish một message tới một messaging bus trung tâm, và các microservice khác có bản sao dữ liệu có thể nhận thông báo bằng cách đăng ký bus. Vì các cuộc gọi là bất đồng bộ, từng microservice riêng lẻ có thể tiếp tục phục vụ request bằng bản sao dữ liệu hiện có, và hệ thống cần chấp nhận rằng dù có thể chưa nhất quán ngay lập tức — nghĩa là dữ liệu có thể chưa đồng bộ ngay — cuối cùng nó sẽ nhất quán trên các microservice.

### Analogy: CAP Theorem giống việc chọn giữa "tin tức nóng hổi" và "tin tức luôn có sẵn"

Hãy nghĩ CAP giống một tòa soạn báo: bạn không thể vừa đảm bảo mọi độc giả luôn đọc được tin mới nhất tuyệt đối (Consistency), vừa đảm bảo báo luôn có sẵn để đọc bất cứ lúc nào (Availability), khi đường truyền tin từ phóng viên hiện trường về tòa soạn có thể bị gián đoạn (Partition). Microservices trong thực tế gần như luôn chọn ưu tiên Availability.

---

## Phần 5: Hiểu về Pub-Sub Pattern

Pattern pub-sub đã trở nên phổ biến rộng rãi và được đánh giá cao, sử dụng nhiều trong hệ thống phân tán. Về cơ bản, pattern này xoay quanh việc publish dữ liệu, gọi theo ngữ cảnh là message, tới một hệ thống messaging trung gian (thường gọi là event bus hoặc message broker) — có thể mô tả là bền vững và đáng tin cậy — và sau đó các ứng dụng đăng ký (subscribing application) theo dõi hệ thống trung gian này. Subscriber thể hiện quan tâm tới các loại sự kiện cụ thể và nhận các message liên quan từ broker. Sự tách rời này tăng cường tính module hóa và khả năng mở rộng của hệ thống, vì các service có thể tiến hóa độc lập mà không có phụ thuộc trực tiếp.

Implement pattern pub-sub bao gồm:

- **Event definition**: tạo các event class đóng gói dữ liệu cần truyền tải
- **Publishing events**: service tạo và gửi event tới message broker
- **Subscribing to events**: service đăng ký nhận các event cụ thể từ broker
- **Event handling**: khi nhận event, subscriber thực thi business logic phù hợp

### Event Definition — 4 thành phần chính

- **Event schema**: biểu diễn có cấu trúc của dữ liệu event, chi tiết hóa các thuộc tính và kiểu dữ liệu tương ứng. Ví dụ, event `AppointmentCreated` có thể bao gồm các thuộc tính như `AppointmentId`, `PatientId`, và `DoctorId`. Định nghĩa schema tường minh đảm bảo mọi service diễn giải event đều hiểu nhất quán cấu trúc và nội dung của nó.
- **Event type**: phân loại bản chất của event. Đặt tên rõ ràng cho loại event cho phép service đăng ký và xử lý chỉ những event liên quan tới hoạt động của mình.
- **Event versioning**: khi hệ thống tiến hóa, cấu trúc event có thể cần thay đổi. Implement chiến lược versioning đảm bảo tính tương thích ngược, cho phép service xử lý event theo phiên bản mà chúng hỗ trợ, tránh gián đoạn.
- **Event metadata**: thông tin bổ sung như timestamp, source identifier, và correlation ID cung cấp ngữ cảnh về nguồn gốc của event và mối quan hệ với các event/tiến trình khác. Metadata này rất quan trọng để trace luồng event và debug.

### Publishing và Subscribing Events

Publish event liên quan tới việc một service phát ra thông báo về thay đổi trạng thái hoặc sự kiện đáng chú ý để thông báo cho các service khác trong hệ thống. Khi một hành động đáng chú ý xảy ra, service chịu trách nhiệm tạo một event đóng gói dữ liệu liên quan. Event này thường bao gồm chi tiết như loại event, timestamp, và metadata liên quan. Event được tạo ra sẽ được gửi tới event bus hoặc message broker, và các service đăng ký (subscriber) đã thể hiện quan tâm tới loại event cụ thể sẽ nhận event đã publish từ event bus. Một service đăng ký quan tâm tới các loại event cụ thể với event broker hoặc bus. Việc đăng ký này đảm bảo service được thông báo khi event liên quan xảy ra. Khi nhận được event đã đăng ký, service thực thi business logic đã định sẵn để xử lý dữ liệu của event. Việc xử lý này có thể liên quan tới cập nhật trạng thái nội bộ, kích hoạt workflow bổ sung, hoặc phát ra event mới.

### Message Delivery Guarantee — ba mức độ cần nhớ kỹ

Quan trọng để hiểu các đánh đổi liên quan tới đảm bảo gửi message. Các đảm bảo này định nghĩa tần suất một message có thể được gửi tới consumer và ảnh hưởng trực tiếp tới độ tin cậy, hiệu năng, và độ phức tạp của hệ thống.

| Mức độ | Đặc điểm | Khi nào dùng |
|---|---|---|
| **At-most-once** | Message được gửi tối đa một lần. Có thể mất, nhưng không bao giờ trùng lặp. Hữu ích trong hệ thống chấp nhận mất message thỉnh thoảng hoặc downstream service idempotent và việc xử lý lại không khả thi. Mô hình này ưu tiên hiệu năng hơn độ tin cậy. Hệ thống messaging không bao gồm logic retry. | Chấp nhận mất message thỉnh thoảng, ưu tiên hiệu năng |
| **At-least-once** | Message được gửi một hoặc nhiều lần. Không bị mất, nhưng có thể xảy ra trùng lặp. Dùng khi mọi message phải được xử lý, như trong hệ thống billing hoặc order processing. Consumer phải implement idempotency để xử lý message trùng lặp một cách hợp lý. Đây là đảm bảo được dùng nhiều nhất trong hệ thống phân tán. Nhiều dịch vụ messaging cloud-native như Azure Service Bus, Amazon SNS/SQS, và Apache Kafka cung cấp mặc định at-least-once. | Phổ biến nhất — billing, order processing (đòi hỏi consumer phải idempotent) |
| **Exactly-once** | Message được gửi đúng một lần, không trùng lặp và không mất. Ví dụ sử dụng là các giao dịch tài chính rủi ro cao, gửi command nhiệm vụ quan trọng, hoặc các lĩnh vực khác nơi trùng lặp hoặc mất mát sẽ gây ra vấn đề hệ thống. Đây là đảm bảo khó implement nhất, thường đòi hỏi hỗ trợ distributed transaction hoặc cơ chế phối hợp nâng cao giữa broker và consumer. Ít hệ thống thực sự cung cấp ngữ nghĩa exactly-once thật sự. Kafka cung cấp xử lý exactly-once qua idempotent producer, transactional write, và quản lý consumer offset. | Giao dịch tài chính quan trọng — khó implement nhất |

---

## Phần 6: Triển khai thực tế — RabbitMQ + MassTransit trong .NET

Giờ ta hiểu rõ hơn tại sao service có thể giao tiếp bất đồng bộ và các pattern có thể dùng, ta có thể sửa đổi appointment service hiện có để giao tiếp với một notification service, dispatch email tạo lịch hẹn.

Để triển khai, ta sẽ thực hiện các thay đổi:

- Cài đặt và cấu hình một service bus dùng RabbitMQ
- Sửa đổi appointments service để tạo message
- Implement một notifications service để tiêu thụ và xử lý message

### Chọn Message Bus

Khi chọn messaging broker cho microservices .NET, cần cân nhắc yêu cầu khả năng mở rộng, khả năng tích hợp, độ phức tạp vận hành, và chi phí để xác định tùy chọn phù hợp nhất cho nhu cầu ứng dụng. Một số lựa chọn thay thế và ưu/nhược điểm:

| Broker | Ưu điểm | Nhược điểm |
|---|---|---|
| **Apache Kafka** | Xử lý khối lượng dữ liệu lớn hiệu quả; phù hợp real-time analytics và event streaming | Phức tạp hơn khi setup và quản lý; có thể quá mức cần thiết cho nhu cầu messaging đơn giản |
| **Azure Service Bus** | Tích hợp liền mạch với dịch vụ Azure; hỗ trợ kịch bản messaging nâng cao | Ràng buộc vào hệ sinh thái Azure; cân nhắc về chi phí |
| **Amazon SQS** | Có khả năng mở rộng và dễ dùng; tích hợp tốt với dịch vụ AWS khác | Giới hạn ở messaging dựa trên queue; thiếu một số tính năng nâng cao của broker khác |
| **NATS** | Thiết kế đơn giản; độ trễ thấp; phù hợp ứng dụng cloud-native | Tùy chọn persistence hạn chế; ít tính năng hơn các broker toàn diện hơn |
| **ActiveMQ** | Hỗ trợ nhiều giao thức; linh hoạt và trưởng thành | Có thể tốn tài nguyên; có thể cần bảo trì nhiều hơn |
| **RabbitMQ** | Hỗ trợ nhiều giao thức, khả năng liên thông cao | Kém hiệu năng hơn với workload cao và không hỗ trợ event streaming |

Cho hoạt động này, ta tiếp tục với RabbitMQ, được biết đến nhờ cài đặt và cấu hình đơn giản. Nó cung cấp giao diện quản lý trực quan, dễ tiếp cận cho developer và administrator. Nó cũng xuất sắc trong các kịch bản routing phức tạp và hỗ trợ nhiều pattern messaging, bao gồm one-to-one, one-to-many, và pub-sub. Cuối cùng, nó sở hữu cộng đồng lớn, tích cực và tài liệu phong phú, hỗ trợ xử lý sự cố và chia sẻ kiến thức.

Các thành phần chính tóm tắt cách RabbitMQ hoạt động:

- **Producers và consumers**: ứng dụng gửi message gọi là producer, ứng dụng nhận message gọi là consumer
- **Exchanges và queues**: producer gửi message tới exchange, exchange route chúng tới queue dựa trên quy tắc định sẵn. Consumer sau đó lấy message từ các queue này
- **Bindings**: liên kết giữa exchange và queue xác định message nên chảy như thế nào
- **Message routing**: RabbitMQ hỗ trợ nhiều loại exchange (direct, topic, fanout, headers) để kiểm soát cơ chế routing message

### Cài đặt RabbitMQ qua Docker

```bash
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:4.0-management
```

Sau khi setup xong, truy cập `http://localhost:15672/`, xác thực bằng `guest`/`guest` để mở dashboard quản lý — cung cấp metrics thời gian thực về message rate, queued message, trạng thái node, thông tin chi tiết mỗi queue (message count, consumer, binding), và insight về các connection/channel đang hoạt động.

### Implementing a Publisher — Appointments Service

MassTransit là package mã nguồn mở đơn giản hóa việc phát triển ứng dụng dựa trên message, cung cấp abstraction nhất quán trên nhiều message transport bao gồm RabbitMQ, Azure Service Bus, và Amazon SQS.

```bash
dotnet add package MassTransit
dotnet add package MassTransit.AspNetCore
dotnet add package MassTransit.RabbitMQ
```

Định nghĩa message (MassTransit yêu cầu message type là class, record, hoặc interface):

```csharp
public class AppointmentCreated
{
    public Guid AppointmentId { get; set; }
    public Guid PatientId { get; set; }
    public Guid DoctorId { get; set; }
    public DateTime AppointmentDate { get; set; }
}
```

Cấu hình MassTransit dùng RabbitMQ trong `Program.cs`:

```csharp
using MassTransit;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq();
});
```

Lợi ích bổ sung của cấu hình này: giờ ta có thể inject `IPublishEndpoint` service vào bất kỳ class nào cần publish message tới RabbitMQ. Sửa controller và POST action:

```csharp
public class AppointmentsController : ControllerBase
{
    private readonly IPublishEndpoint _publishEndpoint;

    public AppointmentsController(/* Other services */, IPublishEndpoint publishEndpoint)
    {
        _publishEndpoint = publishEndpoint;
    }

    [HttpPost]
    public async Task<ActionResult<Appointment>> PostAppointment(Appointment appointment)
    {
        /* Xử lý appointment và insert vào database */

        // Publish AppointmentCreated tới RabbitMQ
        await _publishEndpoint.Publish<AppointmentCreated>(new
        {
            appointment.AppointmentId,
            appointment.PatientId,
            appointment.DoctorId,
            appointment.Slot.Start
        });

        return CreatedAtAction("GetAppointment", new { id = appointment.AppointmentId }, appointment);
    }
}
```

Ta đã sửa POST method để lưu appointment đến vào database và gọi generic method `Publish` trên interface `IPublishEndpoint`. Ta dùng class `AppointmentCreated` để định nghĩa loại event sẽ publish.

### Implementing a Consumer — Notification Service

Consumer service cho hoạt động này sẽ thông báo cho bệnh nhân liên quan mỗi khi một lịch hẹn được tạo. Service này phải đăng ký RabbitMQ và phản hồi các message đến tương ứng. Nó cũng phải giao tiếp đồng bộ với Doctor và Patient API để lấy chi tiết liên hệ.

Bằng cách thêm MassTransit vào `AppointmentsApi`, ta có thể tự động monitor một message queue. Nó có thể xử lý message đến từ queue mà không cần một service riêng biệt. Điều này tuyệt vời vì ta có thể retrofit service hiện có để dispatch notification ở background.

**Lưu ý thiết kế quan trọng**: dùng service này để vừa quản lý appointment vừa dispatch notification có thể xem là vi phạm nguyên tắc chi phối microservices, vì service này giờ có hai chức năng. Đây cũng là vùng xám, vì các notification liên quan tới appointment, và ta muốn đảm bảo chúng được dispatch bất đồng bộ. Nếu tất cả service cần dispatch notification, thì một `NotificationsService` trung tâm sẽ là cách tiếp cận phù hợp hơn. Sách chọn sửa `AppointmentsApi` mà không phức tạp hóa tình huống, để monitor RabbitMQ exchange và dispatch email khi cần.

Nếu consumer và publisher nằm ở project/solution khác nhau, một shared library phải là điểm tham chiếu trung tâm cho các service phụ thuộc. Cùng một namespace phải được dùng cho cả publisher và consumer.

Định nghĩa consumer class implement `IConsumer<T>`, với T là loại message mong đợi nhận (`AppointmentCreated`):

```csharp
public class AppointmentCreatedConsumer(
    IEmailService _emailService,
    PatientsApiClient patientsApiClient,
    DoctorsApiClient doctorsApiClient) : IConsumer<AppointmentCreated>
{
    public async Task Consume(ConsumeContext<AppointmentCreated> context)
    {
        var message = context.Message;

        var doctor = await doctorsApiClient.GetDoctorAsync(message.DoctorId);
        var patient = await patientsApiClient.GetPatientAsync(message.PatientId);

        var emailContent = $"Dear {patient.FirstName} {patient.LastName},\n\n" +
            $"Your appointment with Dr. {doctor.FirstName} {doctor.LastName} " +
            $"is confirmed for {message.AppointmentDate}.\n\n" +
            $"Best regards,\nHealth Care Management System";

        await _emailService.SendEmailAsync(patient.Email, "Appointment Confirmation", emailContent);
    }
}
```

Ta đang implement interface `IConsume<T>`, yêu cầu định nghĩa method `Consume`. Khi một message được nhận, nó sẽ thực thi code chứa trong đó, nơi ta lấy chi tiết bác sĩ và bệnh nhân, soạn email, và dispatch bằng email service. Email service này dùng MailKit để xử lý thao tác gửi email:

```csharp
using MailKit.Net.Smtp;
using MimeKit;

public interface IEmailService
{
    Task SendEmailAsync(string to, string subject, string body);
}

public class EmailService : IEmailService
{
    public async Task SendEmailAsync(string to, string subject, string body)
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress("HealthCare System", "no-reply@healthcare.com"));
        message.To.Add(new MailboxAddress("Patient", to));
        message.Subject = subject;
        message.Body = new TextPart("plain") { Text = body };

        using var client = new SmtpClient();
        await client.ConnectAsync("smtp.mailserver.com", 587, false);
        await client.AuthenticateAsync("smtp_username", "smtp_password");
        await client.SendAsync(message);
        await client.DisconnectAsync(true);
    }
}
```

Chi tiết nhạy cảm như `smtp_username` và `smtp_password` nên được trừu tượng hóa vào configuration hoặc file secrets — được tham chiếu trực tiếp ở đây chỉ vì mục đích đơn giản hóa ví dụ.

Đăng ký email service và cấu hình MassTransit trong `Program.cs`:

```csharp
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<AppointmentCreatedConsumer>();
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ReceiveEndpoint("appointment_created_queue", e =>
        {
            e.ConfigureConsumer<AppointmentCreatedConsumer>(context);
        });
    });
});
```

Với các sửa đổi này, ta có thể xác nhận rằng gọi POST method của API sẽ ảnh hưởng tới database và đặt một message lên queue trong RabbitMQ. Consumer sẽ cố xử lý message ngay khi nó khả dụng. Đây hoàn tất một implementation pub-sub tiêu chuẩn. Ghi nhớ, quyết định thiết kế của bạn rất quan trọng và phụ thuộc nhiều vào nhu cầu dự án. Đây là thực hành chuẩn để tạo một microservice phân phối alert và notification thay mặt cho các microservice khác. Phương pháp này giảm trách nhiệm của các service riêng lẻ và cho phép xử lý dữ liệu bất đồng bộ thực sự.

---

## Phần 7: Ba cạm bẫy thực chiến của giao tiếp bất đồng bộ

Implement kiến trúc pub-sub mang lại lợi ích đáng kể trong việc tách rời thành phần hệ thống và tăng cường khả năng mở rộng. Tuy nhiên, nó cũng đặt ra nhiều thách thức ảnh hưởng tới hiệu năng, độ tin cậy, và khả năng bảo trì hệ thống.

### 1. Message Ordering (Thứ tự message)

Một thách thức chính là đảm bảo thứ tự message. Trong hệ thống pub-sub, message có thể không đến theo đúng trình tự được gửi. Sự mất trật tự này có thể dẫn tới không nhất quán nếu message cần được xử lý theo một trình tự cụ thể. Để giải quyết, developer có thể bao gồm số thứ tự hoặc timestamp trong payload message, cho phép subscriber sắp xếp lại message khi nhận.

Implement sequencing như vậy đòi hỏi thiết kế cẩn thận để đảm bảo mọi message được gắn tag phù hợp và subscriber có logic để sắp xếp lại chúng đúng cách. Đầu tiên, đảm bảo event `AppointmentCreated` bao gồm thuộc tính cho số thứ tự — ta sẽ giới thiệu timestamp cho ví dụ này:

```csharp
public class AppointmentCreated
{
    // Shortened for brevity.
    public DateTime Timestamp { get; set; }
}
```

Đảm bảo set giá trị thời gian hợp lệ khi tạo message `AppointmentCreated`:

```csharp
await _publishEndpoint.Publish<AppointmentCreated>(new
{
    // Other assignments
    DateTime.UtcNow
});
```

Khi tạo message `AppointmentCreated` mới, set `Timestamp` thành UTC hiện tại để đảm bảo nhất quán trên các hệ thống khác nhau. Trong consumer, lưu lại timestamp đã xử lý gần nhất cho mỗi appointment. Trước khi xử lý message mới, kiểm tra xem timestamp của nó có mới hơn timestamp đã xử lý gần nhất không, để đảm bảo message được xử lý đúng thứ tự. Nên lưu giá trị này bằng non-volatile storage (file hoặc database) để đảm bảo không bao giờ mất khi service restart. Để đơn giản, ta dùng local variable:

```csharp
private static readonly ConcurrentDictionary<Guid, DateTime> LastProcessedTimestamps = new();

public async Task Consume(ConsumeContext<AppointmentCreated> context)
{
    var message = context.Message;
    var lastTimestamp = LastProcessedTimestamps.GetOrAdd(message.AppointmentId, DateTime.MinValue);

    if (message.Timestamp > lastTimestamp)
    {
        // Process message and send email.
        // Shortened for brevity.
        LastProcessedTimestamps[message.AppointmentId] = message.Timestamp;
    }
    else
    {
        // implement logic to handle out-of-order messages, such as
        // logging or storing for later processing
    }
}
```

Quyết định chiến lược quản lý message không đúng thứ tự. Bạn có thể implement cơ chế retry hoặc lưu trữ tạm thời để sắp xếp lại. Giới hạn concurrency và đặt prefetch count thấp có thể giúp duy trì thứ tự nhưng có thể ảnh hưởng throughput. Đánh giá đánh đổi giữa thứ tự message và hiệu năng dựa trên yêu cầu ứng dụng. MassTransit có thể được sửa để hỗ trợ điều này:

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<AppointmentCreatedConsumer>();
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ReceiveEndpoint("appointment_created_queue", e =>
        {
            e.PrefetchCount = 1; // Fetch one message at a time
            e.UseConcurrencyLimit(1); // Process one message at a time
            e.ConfigureConsumer<AppointmentCreatedConsumer>(context);
        });
    });
});
```

### 2. Duplicate Messages (Message trùng lặp)

Một vấn đề đáng kể khác là trùng lặp message. Cùng một message có thể được gửi nhiều lần hơn một, dẫn tới xử lý dư thừa và các bất thường dữ liệu tiềm tàng.

Điều kiện mạng không ổn định có thể dẫn tới trùng lặp message. Ví dụ, nếu acknowledgment từ consumer tới broker bị mất do vấn đề mạng, broker có thể giả định message chưa được xử lý và gửi lại. Tương tự, nếu consumer không acknowledge trong khoảng thời gian timeout quy định, broker có thể gửi lại message.

Nhiều hệ thống messaging ưu tiên đảm bảo message được gửi ít nhất một lần, ngay cả khi điều đó có nghĩa là thỉnh thoảng bị trùng lặp. Cách tiếp cận này ưu tiên độ tin cậy hơn là gửi đơn lẻ nghiêm ngặt.

Để giảm thiểu điều này, cần thiết kế **idempotent message handler** — các message handler có thể xử lý cùng một message nhiều lần mà không gây tác dụng phụ. Cách khác, kết hợp unique message identifier cho phép subscriber theo dõi và loại bỏ trùng lặp hiệu quả. Implement idempotency hoặc cơ chế phát hiện trùng lặp thêm phức tạp cho hệ thống nhưng quan trọng để duy trì tính toàn vẹn dữ liệu.

Một cách khác để quản lý vấn đề này là giới thiệu message ID được theo dõi trong database hoặc cache. Điều này được kiểm tra mỗi lần để nhận diện và xử lý trùng lặp tiềm ẩn phù hợp:

```csharp
public class AppointmentCreated
{
    // Shortened for brevity
    public Guid MessageId { get; set; }
}
```

Khi tạo message `AppointmentCreated` mới, gán giá trị Guid từ bản ghi database vào thuộc tính `MessageId` để đảm bảo tính duy nhất:

```csharp
await _publishEndpoint.Publish<AppointmentCreated>(new
{
    // Other assignments
    MessageId = appointment.AppointmentId
});
```

Trong consumer, lưu lại record các message ID đã xử lý để phát hiện và bỏ qua trùng lặp. Một lần nữa, lưu ID bằng database hoặc cache là tốt nhất, nhưng ví dụ này dùng `ConcurrentDictionary`. `Consume` method sẽ kiểm tra xem ID của message hiện tại có trong danh sách không và dừng thực thi nếu đúng:

```csharp
private static readonly ConcurrentDictionary<Guid, bool> ProcessedMessageIds = new();

public async Task Consume(ConsumeContext<AppointmentCreated> context)
{
    var message = context.Message;

    if (ProcessedMessageIds.ContainsKey(message.MessageId))
    {
        Console.WriteLine($"Duplicate message detected: {message.MessageId}. Ignoring.");
        return; // Exit without processing
    }

    // Proceed with processing the message

    ProcessedMessageIds[message.MessageId] = true;
}
```

Xử lý message trùng lặp có thể giảm đáng kể thời gian xử lý giữa tiêu thụ message và tăng hiệu quả.

### 3. Additional Considerations — Bốn mối lo hạ tầng cấp cao hơn

Hệ thống càng phát triển, nó càng trở nên phân tán hơn. Tại điểm này, các vấn đề ta đối mặt và phải xử lý không thể giải quyết hết bằng code. Dù là developer, ta phải luôn cân nhắc ảnh hưởng hạ tầng của nỗ lực phát triển:

- **Scalability**: khi số lượng publisher và subscriber tăng, hệ thống có thể gặp nghẽn hiệu năng, dẫn tới tăng độ trễ hoặc mất message. Nên dùng distributed message broker hỗ trợ load balancing và horizontal scaling để tăng cường khả năng mở rộng. Partition topic và implement consumer group cũng có thể phân phối tải đồng đều hơn. Tuy nhiên, các giải pháp này đòi hỏi hạ tầng vững chắc và cấu hình cẩn thận để hoạt động hiệu quả dưới tải thay đổi.

- **Fault tolerance**: lỗi ở bất kỳ phần nào của hệ thống pub-sub có thể dẫn tới mất message hoặc downtime hệ thống. Implement redundancy và cơ chế failover, như có nhiều instance publisher/subscriber và dùng persistent storage cho message, có thể giảm thiểu rủi ro này. Dù các biện pháp này cải thiện độ tin cậy, chúng cũng đưa vào độ phức tạp và yêu cầu tài nguyên bổ sung.

- **Security**: truy cập trái phép vào message có thể dẫn tới rò rỉ dữ liệu, làm lộ thông tin nhạy cảm. Implement cơ chế authentication và authorization vững chắc là thiết yếu để kiểm soát truy cập topic. Mã hóa message khi truyền tải và khi lưu trữ bảo vệ thêm dữ liệu khỏi truy cập trái phép. Tuy nhiên, các biện pháp bảo mật này có thể đưa vào overhead hiệu năng và cần quản lý liên tục cho key và access control.

- **Logging and tracing**: do bản chất bất đồng bộ và phân tán, monitoring và debugging trong hệ thống pub-sub có thể phức tạp. Theo dõi luồng message và chẩn đoán vấn đề đòi hỏi công cụ logging và monitoring toàn diện. Implement correlation ID có thể giúp trace đường đi message xuyên suốt các thành phần, hỗ trợ debug dễ dàng hơn. Thiết lập hạ tầng monitoring như vậy đòi hỏi nỗ lực phát triển bổ sung và có thể tăng độ phức tạp hệ thống.

---

## Tổng kết chương

Chương này đi sâu vào giao tiếp bất đồng bộ giữa các microservice — yếu tố then chốt cho khả năng mở rộng, phục hồi, và linh hoạt trong hệ thống phân tán. Ta khám phá ba pattern chính: message queue (một-đối-một, đảm bảo thứ tự FIFO), pub-sub (một-đối-nhiều, tách rời triệt để), và event streaming.

Eventual consistency — hệ quả tất yếu của việc tách dữ liệu ra nhiều microservice — được lý giải qua CAP theorem, buộc ta phải đánh đổi giữa Consistency và Availability trong một hệ thống luôn cần chấp nhận Partition tolerance.

Ta đã triển khai thực tế một hệ thống pub-sub hoàn chỉnh với RabbitMQ và MassTransit trong .NET: appointment service publish event `AppointmentCreated`, notification service tiêu thụ và gửi email xác nhận. Cuối cùng, ta giải quyết hai thách thức thực chiến quan trọng nhất của messaging bất đồng bộ — message ordering (dùng timestamp) và duplicate message (dùng idempotent handler với message ID tracking) — cùng bốn mối lo hạ tầng cấp cao hơn: scalability, fault tolerance, security, và logging/tracing.

Chương tiếp theo sẽ tiếp tục khám phá các pattern thiết kế nâng cao hơn cho hệ thống microservices.
