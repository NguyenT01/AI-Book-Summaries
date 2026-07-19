# Bài học: Khi "không cần lo máy chủ" trở thành lợi thế cạnh tranh — Serverless Microservices

> Chương 15 — Serverless Microservices Development
> Ví dụ xuyên suốt: dịch vụ đặt lịch hẹn (Appointment Booking) trong hệ thống HealthCare, xây bằng Azure Functions + Durable Functions, dùng Azure Table Storage và Queue Storage.

---

## Phần 1: Vấn đề thực tế — khi orchestration của Kubernetes cũng là một gánh nặng

Hãy nhớ lại chương trước: bạn vừa dựng xong cụm Kubernetes cho hệ thống HealthCare, viết Deployment, Service, quản lý replica, và tự hào vì hệ thống giờ đã tự phục hồi. Nhưng rồi một buổi sáng, bạn nhận ra: dịch vụ đăng ký bệnh nhân (patient registration) gần như không có traffic vào ban đêm, chỉ bận rộn trong giờ hành chính — vậy mà cụm Kubernetes vẫn giữ nguyên số Pod đó chạy 24/7, vẫn tốn tài nguyên, vẫn tốn chi phí y hệt lúc cao điểm. Bạn bắt đầu tự hỏi: liệu có cách nào để hạ tầng chỉ "sống" đúng lúc có việc cần làm, và hoàn toàn "biến mất" (và ngừng tính phí) khi không ai dùng tới?

Đây chính là câu hỏi mà serverless computing trả lời. Serverless không có nghĩa là không còn server nào cả — server vẫn tồn tại, chỉ là trách nhiệm quản lý nó được chuyển hẳn sang cho nhà cung cấp cloud. Bạn không còn phải cấp phát VM, scale container, hay vá lỗi hệ điều hành. Bạn chỉ deploy function, định nghĩa trigger cho nó, và để nhà cung cấp lo phần còn lại.

Nhìn lại hành trình hosting mà chúng ta đã đi qua: bắt đầu từ server vật lý (Chapter 14 đã nhắc — đắt, khó bảo trì, khó khôi phục), tới máy ảo (cô lập mạnh nhưng nặng nề), tới container (nhẹ hơn, portable, nhưng vẫn cần một lớp orchestration như Kubernetes để vận hành — chính là điều bạn vừa trải nghiệm ở Chapter 14). Serverless là bước tiếp theo trong chuỗi tiến hóa đó: bỏ hẳn luôn cả lớp orchestration, chỉ còn lại code nghiệp vụ thuần túy.

### Analogy: Từ thuê cả căn nhà đến chỉ trả tiền điện theo giờ bật đèn

Nếu server vật lý giống như mua đứt một căn nhà, máy ảo giống như thuê một căn hộ trong chung cư, và container giống như thuê một góc văn phòng chia sẻ (co-working space) — thì serverless giống như bạn chỉ trả tiền điện đúng theo số giờ bóng đèn thực sự bật sáng. Không dùng tới, không tốn một đồng nào. Cần dùng, đèn bật lên gần như ngay lập tức, sáng đúng độ cần thiết, rồi tắt đi khi xong việc — bạn không bao giờ phải lo "phòng này có ai thuê hay bỏ trống" như với chung cư hay văn phòng, vì về bản chất không có "phòng" cố định nào cả.

Trong serverless, developer viết code dưới dạng các **function** rời rạc — mô hình này gọi là **Functions as a Service (FaaS)**. Các function này được cấu hình để chạy dựa trên **trigger** (sự kiện kích hoạt), và về bản chất là **stateless** — nghĩa là chúng thực thi mà không có sẵn ngữ cảnh đầy đủ từ lần chạy trước. Cũng có những tùy chọn function có trạng thái (stateful), điều mà chúng ta sẽ đi sâu qua **Durable Functions** ở phần sau.

Một số đặc điểm cốt lõi của kiến trúc serverless:

- **Không cần quản lý hạ tầng**: developer không cấp phát hay quản lý server.
- **Tự động co giãn**: từng function co giãn độc lập tùy theo số lượng request tới.
- **Tính phí theo mức sử dụng**: bạn chỉ trả tiền cho đúng thời gian function thực sự chạy.
- **Thực thi hướng sự kiện (event-driven)**: function được kích hoạt bởi HTTP request, message queue, upload file, hay thay đổi trong database. Mô hình này tạo ra sự liên kết lỏng (loose coupling) giữa các service. Ví dụ, khi một bệnh nhân đăng ký trong hệ thống HealthCare, một event `PatientRegistered` có thể kích hoạt các function gửi thông báo chào mừng, khởi tạo hồ sơ ban đầu, và báo cho bác sĩ phụ trách — tất cả mà không cần service đăng ký bệnh nhân biết chi tiết gì về các bước phía sau.
- **Tính không trạng thái (statelessness)**: function serverless mặc định không giữ trạng thái — bất kỳ trạng thái nào cần thiết phải được truyền tường minh hoặc lấy từ nơi lưu trữ ngoài.

Có hai mô hình chính trong hệ sinh thái serverless:

- **FaaS (Functions as a Service)**: hình thức phổ biến nhất, cho phép deploy từng function riêng lẻ, được gọi bởi một sự kiện cụ thể. Các nhà cung cấp tiêu biểu: Azure Functions (Microsoft), AWS Lambda (Amazon), Google Cloud Functions (Google). Các function này stateless và tồn tại ngắn hạn (ephemeral) — lý tưởng cho endpoint microservice, xử lý biến đổi dữ liệu, hay orchestration nhẹ.
- **BaaS (Backend as a Service)**: các dịch vụ backend dựng sẵn trên cloud như xác thực (Firebase Auth), lưu trữ file (Azure Blob Storage), database (Cosmos DB), messaging thời gian thực (SignalR Service). Trong kiến trúc serverless microservices, BaaS thường bổ trợ cho FaaS bằng cách gánh vác những trách nhiệm backend thường nhật.

**Bài học cốt lõi: "Serverless" không phải là không có server — đó là việc chuyển giao hoàn toàn trách nhiệm quản lý server sang cho nhà cung cấp cloud, để bạn chỉ còn phải nghĩ về code nghiệp vụ và trigger kích hoạt nó.**

---

## Phần 2: Serverless so với microservices container hóa

Ở Chapter 14, chúng ta đã thấy container hóa là cách đóng gói microservices đảm bảo triển khai nhất quán qua các môi trường. Nhưng dù linh hoạt và portable, container vẫn đòi hỏi một mức độ quản lý hạ tầng nhất định — như quản lý orchestration với Kubernetes hay Docker Swarm mà chúng ta vừa trải qua.

Serverless bổ sung cho container hóa bằng cách trừu tượng hóa hẳn lớp orchestration đó. Thay vì deploy một container lên cluster, bạn deploy một function lên serverless runtime, và nó lập tức sẵn sàng phản hồi sự kiện đến. Sự đơn giản này chính là lý do serverless ngày càng được ưa chuộng cho các workload có đặc điểm:

- Ngắn hạn hoặc không thường xuyên.
- Tải trọng dao động thất thường, khó dự đoán.
- Hướng sự kiện (webhook, upload file, job theo lịch).
- Là một phần của workflow (xử lý nền, gửi thông báo).

Điều quan trọng cần lưu ý: serverless và container **không loại trừ nhau**. Nhiều ứng dụng cloud-native hiện đại dùng cả hai song song — container xử lý các tiến trình chạy dài, còn serverless function quản lý các thao tác ngắn hạn hoặc mang tính phản ứng (reactive).

**Nhắc lại từ Chapter 14**: nếu Kubernetes là "hệ thống lái tự động có giám sát" cho một đội xe luôn hoạt động, thì serverless giống như gọi taxi theo cuốc — bạn không sở hữu, không bảo trì, không phải lo xe đang đỗ ở đâu khi không dùng tới; chỉ gọi khi cần, trả tiền đúng cuốc đó.

---

## Phần 3: Bức tranh các nhà cung cấp cloud và dịch vụ serverless

Trong hệ thống HealthCare, chúng ta cần quản lý việc tiếp nhận bệnh nhân, đặt lịch hẹn, hồ sơ y tế, thông báo thời gian thực, và tích hợp với hệ thống bên ngoài — mỗi mảng này đều hưởng lợi từ các microservice hướng sự kiện, liên kết lỏng, có thể co giãn độc lập và vận hành với overhead hạ tầng tối thiểu.

Triết lý serverless nhất quán giữa các nhà cung cấp, nhưng cách triển khai và bộ công cụ khác biệt đáng kể. Hãy điểm qua hai cái tên lớn nhất: AWS và Microsoft Azure.

### AWS

AWS được chọn nhiều nhờ độ trưởng thành, hỗ trợ ngôn ngữ rộng, và hệ sinh thái tích hợp khổng lồ. Ra mắt từ 2006, AWS hiện có hơn 200 dịch vụ trải khắp compute, storage, AI, machine learning, analytics, và công cụ dev. Nhiều tổ chức y tế đã chọn AWS nhờ tính năng bảo mật, các dịch vụ đủ chuẩn HIPAA, và độ linh hoạt kiến trúc. Một số dịch vụ AWS hữu ích cho hệ thống của chúng ta:

- **AWS Lambda**: dịch vụ FaaS của Amazon — chạy code backend đáp ứng HTTP request, message trên queue, upload file, hay trigger theo lịch, không cần cấp phát hay quản lý server.
- **Amazon API Gateway**: dịch vụ quản lý toàn diện, expose Lambda function (hoặc dịch vụ khác) qua endpoint HTTP(S), đóng vai trò "cửa ngõ" cho API serverless, hỗ trợ cả REST lẫn WebSocket.
- **Nhóm dịch vụ messaging/event bus** — EventBridge/SNS/SQS:
  - **Amazon EventBridge**: event bus quản lý toàn diện, định tuyến sự kiện giữa các dịch vụ AWS, ứng dụng SaaS, và service tùy chỉnh — hỗ trợ liên kết lỏng theo kiến trúc hướng sự kiện.
  - **Amazon SNS**: dịch vụ messaging pub/sub, broadcast thông báo tới nhiều subscriber (Lambda, SMS, email, endpoint HTTP).
  - **Amazon SQS**: dịch vụ hàng đợi message bền vững, quản lý toàn diện, dùng để tách rời và làm bộ đệm giao tiếp giữa các microservice.
- **AWS Step Functions**: dịch vụ điều phối workflow, kết hợp nhiều dịch vụ AWS thành workflow serverless, hỗ trợ nhánh rẽ, retry, delay, và tương tác con người — lý tưởng để mô hình hóa luồng đặt lịch hẹn hay tiếp nhận bệnh nhân.
- **Amazon CloudWatch**: công cụ quan sát và giám sát cốt lõi trên AWS — thu thập log, metric, trace từ dịch vụ và ứng dụng, cho phép tạo dashboard, đặt alarm, phân tích hành vi hệ thống theo thời gian thực.
- **Nhóm dịch vụ backend serverless**:
  - **Amazon DynamoDB**: database NoSQL quản lý toàn diện, độ trễ thấp, lưu được cả dữ liệu có cấu trúc lẫn bán cấu trúc.
  - **Amazon S3**: kho lưu trữ object siêu bền vững cho file phi cấu trúc như tài liệu scan, báo cáo xét nghiệm, giấy tờ tùy thân.
  - **Amazon Aurora Serverless**: engine database quan hệ (tương thích MySQL/PostgreSQL) tự động co giãn theo tải — lý tưởng cho nhu cầu dữ liệu giao dịch và quan hệ như nhật ký kiểm toán hay yêu cầu bảo hiểm.

### Microsoft Azure

Azure là nền tảng cloud toàn diện của Microsoft, bao phủ compute, storage, network, database, AI, analytics, DevOps, và hosting ứng dụng, hỗ trợ .NET, Java, Python, Node.js, Go, cả IaaS lẫn PaaS. Azure nổi bật ở khả năng tích hợp chặt với công cụ doanh nghiệp, giải pháp hybrid cloud, và các tính năng identity/compliance mạnh mẽ. Một số dịch vụ Azure hữu ích:

- **Azure Functions**: nền tảng FaaS của Azure — viết code hướng sự kiện, kích hoạt bởi HTTP request, message queue, timer, sự kiện blob storage, v.v. Hỗ trợ .NET, Node.js, Python, Java, PowerShell.
- **Azure API Management (APIM)**: dịch vụ API gateway và quản lý toàn diện — expose Azure Functions và các dịch vụ khác thành RESTful API, áp policy (rate limiting, validation), bảo mật truy cập qua key, OAuth2, hoặc Entra ID.
- **Nhóm dịch vụ messaging** — Event Grid/Service Bus/Queue Storage:
  - **Azure Event Grid**: dịch vụ định tuyến sự kiện quản lý toàn diện, hỗ trợ reactive programming — chuyển sự kiện từ publisher (Blob Storage, app tùy chỉnh) tới subscriber (Functions, Logic Apps).
  - **Azure Service Bus**: dịch vụ messaging cấp doanh nghiệp đáng tin cậy, có queue, topic (pub/sub), session, và dead-lettering — lý tưởng cho giao tiếp liên microservice và điều phối sự kiện nghiệp vụ.
  - **Azure Queue Storage**: hệ thống hàng đợi đơn giản cho tác vụ nền nhẹ và retry.
- **Azure Durable Functions**: phần mở rộng của Azure Functions cho phép orchestration có trạng thái (stateful) — lý tưởng để mô hình hóa workflow như đặt lịch hẹn, tiếp nhận bệnh nhân, hay xác minh bảo hiểm nhiều bước. Hỗ trợ chaining function, fan-out/fan-in, và tác vụ chạy dài có tính bền vững.
- **Azure Monitor và Application Insights**: bộ công cụ toàn diện cho logging, tracing, thu thập metric. Azure Monitor giám sát hạ tầng và sức khỏe ứng dụng; Application Insights cung cấp telemetry chi tiết về hiệu năng function, cuộc gọi dependency, và chẩn đoán lỗi.
- **Nhóm backend serverless**:
  - **Azure Cosmos DB**: database NoSQL đa mô hình, phân tán toàn cầu, đọc/ghi độ trễ thấp — lý tưởng cho hồ sơ bệnh nhân, dữ liệu lịch hẹn, hay telemetry IoT y tế.
  - **Azure Blob Storage**: giải pháp lưu trữ object co giãn cao cho file nhị phân như đơn thuốc scan, giấy tờ tùy thân, file đính kèm.
  - **Azure SQL Database (Serverless tier)**: database quan hệ tự động co giãn, tạm dừng khi không hoạt động và tự phục hồi khi có nhu cầu — lý tưởng cho dữ liệu giao dịch có cấu trúc với hỗ trợ SQL và tích hợp Entra ID.
- **Azure Key Vault**: kho lưu trữ an toàn cho secret, key, và chứng chỉ — bảo vệ connection string, credential dịch vụ, dữ liệu mã hóa, đảm bảo giao tiếp an toàn giữa các microservice.

### So sánh nhanh triết lý hai nền tảng

AWS, với vị thế nhà tiên phong, cung cấp phổ dịch vụ rộng hơn với tùy chọn cấu hình chi tiết và hạ tầng toàn cầu — lựa chọn tốt cho team cần độ linh hoạt tối đa, đặc biệt khi xây dựng ứng dụng cloud-native hoàn toàn mới. Công cụ của AWS mô-đun hóa cao nhưng thường đòi hỏi kinh nghiệm cloud sâu hơn để thiết lập hiệu quả. Azure, ngược lại, vượt trội ở khả năng tích hợp liền mạch với hệ sinh thái Microsoft — mang lại trải nghiệm thống nhất hơn cho developer đã quen với .NET, C#, Visual Studio, Microsoft 365. Các tùy chọn serverless của Azure được xây với hỗ trợ mạnh cho workload doanh nghiệp — đặc biệt hấp dẫn với các ứng dụng ngành quy định chặt như hệ thống y tế.

GCP là lựa chọn đáng cân nhắc cho team ưu tiên sự đơn giản, ứng dụng thiên về dữ liệu, và prototype nhanh — Cloud Functions tương tự Lambda/Azure Functions, còn Cloud Run mang tới mô hình serverless dựa trên container độc đáo, hỗ trợ mọi ngôn ngữ và có thể scale về 0 instance.

Ngoài ba ông lớn, còn có Cloudflare Workers (serverless chạy ở edge, cold start dưới mili-giây), Netlify Functions và Vercel Serverless Functions (tích hợp frontend-backend cho ứng dụng JAMstack), cùng các framework mã nguồn mở như OpenFaaS, Knative, Kubeless (mang serverless lên chính Kubernetes — nhắc lại Chapter 14, cho thấy hai mô hình này thực ra có thể hội tụ). Mỗi lựa chọn đánh đổi khác nhau giữa quyền kiểm soát, khả năng co giãn, mức độ tích hợp hệ sinh thái, và độ phức tạp vận hành.

Vì đối tượng học đang tập trung vào hệ sinh thái .NET, phần thực hành còn lại của chương sẽ xoay quanh Microsoft Azure.

---

## Phần 4: Được và mất khi áp dụng serverless cho microservices

### Những gì bạn nhận được

Serverless là một sự khớp nối tự nhiên với microservices. Khi tổ chức áp dụng microservices để phân rã domain phức tạp thành các đơn vị tự trị, dễ quản lý, thì thách thức về quản lý hạ tầng, giao tiếp liên service, và overhead vận hành thường tăng lên theo. Serverless giải quyết trực tiếp vấn đề này:

Mô hình hạ tầng truyền thống yêu cầu cấp phát compute trước, dễ dẫn tới over-provisioning khi traffic thấp hoặc under-provisioning khi traffic tăng vọt — gây lãng phí chi phí hoặc suy giảm hiệu năng. Serverless tính phí theo mức sử dụng thực tế — bạn chỉ trả tiền cho đúng thời gian code chạy, giảm hẳn chi phí gắn với tài nguyên nhàn rỗi so với VM hay container chạy dài hạn (vốn tính phí cố định bất kể có traffic hay không). Function không dùng tới không tốn chi phí vận hành — lý tưởng cho các microservice có traffic thất thường hoặc theo mùa.

Quay lại ví dụ dịch vụ đăng ký bệnh nhân trong hệ thống HealthCare: dịch vụ này bận rộn trong giờ hành chính nhưng gần như im lìm ban đêm. Với mô hình serverless, bạn không phải trả tiền cho compute capacity trong lúc rảnh rỗi, mà vẫn đảm bảo sẵn sàng và hiệu năng tốt trong giờ cao điểm.

Một thách thức lớn khi áp dụng microservices là độ phức tạp vận hành phát sinh từ việc quản lý nhiều service triển khai độc lập. Container hóa mạnh mẽ, nhưng đòi hỏi nền tảng orchestration như Kubernetes — vốn có đường cong học tập dốc và chi phí bảo trì đi kèm (chính điều chúng ta đã trải nghiệm ở Chapter 14). Serverless loại bỏ hoàn toàn nhu cầu quản lý server, container, và orchestration — kèm theo khả năng chịu lỗi tích hợp sẵn, tự động vá lỗi, và tính sẵn sàng cao, tất cả do nhà cung cấp cloud đảm nhiệm. Điều này cho phép team dev tập trung vào tính năng và logic nghiệp vụ, thay vì xử lý sự cố hạ tầng.

Chuyển trọng tâm từ vận hành sang giá trị nghiệp vụ giúp tăng tốc pipeline triển khai và giảm gánh nặng nhận thức cho team, thông qua:

- **Prototype và lặp nhanh**: viết một function, deploy trong vài giây.
- **Triển khai chi tiết (granular)**: cập nhật một function mà không cần deploy lại toàn bộ service hay ảnh hưởng tới các thành phần không liên quan.
- **Công cụ dev phong phú**: Azure Functions hỗ trợ mô hình isolated process của .NET, dependency injection, và debug local.

Ví dụ, nếu một chính sách mới yêu cầu thêm một bước xác minh trong quy trình duyệt đơn thuốc, team có thể thêm function `VerifyInsuranceEligibility` chạy song song với luồng hiện có, deploy độc lập chỉ trong vài giờ.

### Cái giá phải trả

Function serverless mặc định là stateless. Điều này gây khó khăn trong việc duy trì ngữ cảnh workflow hay dữ liệu session người dùng xuyên suốt các function, buộc phải dựa vào external state store như Cosmos DB hay Redis để duy trì liên tục. Mỗi bước lưu trạng thái thêm vào là thêm độ trễ, chi phí, và điểm có thể gây lỗi. Cách khắc phục tốt nhất là chỉ lưu đúng phần dữ liệu tối thiểu cần cho bước tiếp theo — và may mắn thay, hầu hết nhà cung cấp FaaS đều có durable function được thiết kế sẵn để bảo toàn trạng thái mà không cần developer phải tự làm thêm nhiều việc.

Việc dùng công nghệ serverless đồng nghĩa với việc bạn bị ràng buộc vào hệ sinh thái của nhà cung cấp cloud đã chọn — dẫn tới sự gắn chặt (tight coupling) với API và mô hình sự kiện đặc thù của nền tảng, khiến việc di chuyển workload giữa các cloud khác nhau trở nên khó khăn. Truy cập log và trace cũng chỉ giới hạn trong những gì nhà cung cấp cloud phơi bày ra — dù hầu hết đều sinh ra log và telemetry phong phú, bạn vẫn bị giới hạn bởi vendor lock-in và công cụ sẵn có để truy cập log đó.

Một điểm bàn luận trung tâm khác là **độ trễ cold start**. Đây là độ trễ xảy ra khi một function serverless được kích hoạt sau một khoảng thời gian không hoạt động, khi mà môi trường thực thi liên quan đã bị scale về 0. Độ trễ này phát sinh khi nền tảng phải cấp phát hạ tầng cần thiết, tải runtime, và khởi tạo function trước khi xử lý request. Dù đây là biện pháp có chủ đích để tối ưu chi phí, cold start lại tạo ra sự khó lường có thể ảnh hưởng tiêu cực tới trải nghiệm người dùng và độ tin cậy hệ thống.

Trong hệ thống HealthCare, ở những endpoint nhạy cảm về hiệu năng như tra cứu bệnh nhân hay truy cập hồ sơ y tế điện tử (EMR), điều này có thể làm xói mòn niềm tin vào độ tin cậy của hệ thống. Đây là một đánh đổi cần được cân nhắc kỹ lưỡng — đặc biệt quan trọng khi phục vụ API thời gian thực hay quản lý các tiến trình quan trọng trong workflow đã được điều phối. Với các thao tác nhạy cảm về độ trễ, có thể cân nhắc dùng microservice container hóa hoặc gói function dự trữ sẵn (reserved plan); ngược lại, giải pháp serverless tiêu chuẩn phù hợp hơn cho tác vụ nền, batch job, hay workflow chạy không thường xuyên.

**Bài học cốt lõi: Serverless đánh đổi việc "trả tiền theo giờ đèn sáng" bằng cái giá là cold start latency và vendor lock-in — trước khi áp dụng cho mọi workload, phải tự hỏi: service này có thực sự hưởng lợi từ auto-scaling và event-driven execution hay không, và nó có bản chất ngắn hạn, stateless hay không?**

---

## Phần 5: Thiết kế microservices với tư duy serverless

Thiết kế microservices không đơn thuần là chia nhỏ một ứng dụng monolith — nó đòi hỏi tạo ra các thành phần triển khai độc lập, liên kết lỏng, và bám sát domain, có thể tiến hóa và co giãn riêng biệt. Khi đưa serverless vào khung kiến trúc này, các nguyên tắc thiết kế nền tảng vẫn giữ nguyên, nhưng mô hình thực thi thay đổi đáng kể. Serverless định nghĩa lại các khái niệm về ranh giới service, quản lý trạng thái, độ trễ, và quyền kiểm soát vận hành.

Thiết kế kiến trúc serverless đòi hỏi nhiều hơn việc chỉ di dời service hiện có sang nền tảng mới — nó đòi hỏi đánh giá lại độ chi tiết (granularity) của function, chiến lược orchestration, thực hành observability, và giao tiếp liên service. Quyết định có nên chain function, dùng durable workflow để điều phối, hay phát event bất đồng bộ, sẽ ảnh hưởng trực tiếp tới hiệu năng, khả năng bảo trì, và trải nghiệm người dùng của hệ thống.

Hãy dùng quy trình tiếp nhận bệnh nhân (patient onboarding) trong hệ thống HealthCare làm ví dụ điển hình: quy trình này bao gồm thu thập dữ liệu, đặt lịch hẹn, xác minh danh tính, xác thực bảo hiểm, và gửi thông báo. Mỗi bước có thể — và lý tưởng nên — được triển khai như một function serverless riêng lẻ. Nhưng làm sao duy trì tính toàn vẹn của transaction nghiệp vụ? Làm sao xử lý retry, timeout, và dependency giữa các bước?

### Chọn kích thước phù hợp cho function

Trong microservices truyền thống, mỗi service thường tương ứng với một business capability hay bounded context (nhắc lại DDD từ Chapter 2). Trong kiến trúc serverless, ánh xạ này thường được tinh chỉnh sâu hơn — mỗi function thường xử lý một tác vụ hoặc một bước duy nhất trong quy trình.

Khi xây dựng ứng dụng serverless với .NET và Azure Functions, độ chi tiết, kích thước, và phạm vi của function ảnh hưởng đáng kể tới hiệu quả, hiệu năng, khả năng bảo trì, và chi phí. Một số nguyên tắc cần cân nhắc:

- **Một function, một trách nhiệm**: mỗi function nên đóng gói một tác vụ đơn giản, rõ ràng — như tạo lịch hẹn, gửi thông báo, hay xác thực thanh toán. Điều này bám sát nguyên tắc single responsibility, giúp test, debug, và deploy đơn giản hơn.
- **Gộp theo tính cố kết (cohesion), không gộp theo tiện lợi**: tránh gộp các tác vụ không liên quan vào một function chỉ vì tiện. Thay vào đó, gộp các bước có tính cố kết cao — như validate và làm sạch dữ liệu đầu vào, nếu chúng luôn xảy ra cùng nhau.
- **Tránh chain function quá đà**: chain quá nhiều function nhỏ lẻ (ví dụ chuỗi function HTTP từng bước một) tạo ra overhead hiệu năng và làm phức tạp việc tracing. Nên cân nhắc dùng một orchestrator cho các luồng nhiều bước.
- **Dùng quy ước đặt tên rõ ràng**: đặt tên function mô tả *chúng làm gì* thay vì *chúng làm như thế nào* — ví dụ `RegisterPatient`, `CreateAppointment`, `SendAppointmentNotification`.

Chọn đúng kích thước function là yếu tố then chốt cho hiệu năng, khả năng bảo trì, và quản lý chi phí trong phát triển serverless .NET. Function nên tập trung, stateless, và bám sát nghiệp vụ — tránh cả hai thái cực: thiết kế quá đồ sộ (monolithic) lẫn quá manh mún (over-fragmented).

### Dependency injection trong serverless function

Ứng dụng ASP.NET Core truyền thống phụ thuộc nhiều vào DI tích hợp sẵn. Function serverless cũng cần áp dụng pattern này, nhưng với thêm ràng buộc do vòng đời và hành vi cold start. DI là một pattern kỹ thuật cốt lõi, thúc đẩy liên kết lỏng, hỗ trợ test dễ dàng hơn, và tăng khả năng bảo trì — cho phép component khai báo dependency qua constructor hay method, được cung cấp từ bên ngoài, tránh việc tự khởi tạo thủ công bên trong component.

Một số best practice bổ sung khi thiết kế function serverless:

- Dùng **isolated worker model** — hỗ trợ đầy đủ DI của ASP.NET Core qua `IHostBuilder`, `IServiceCollection`, và `ConfigureServices`.
- Chỉ đăng ký những gì thực sự cần. Đồ thị DI (DI graph) nặng nề làm tăng thời gian khởi động, khuếch đại độ trễ cold start.
- Chọn lifetime của service một cách có chủ đích:
  - **Singleton**: tồn tại xuyên suốt các lần gọi (nếu instance còn "warm") — phù hợp cho tài nguyên dùng chung như `HttpClient`.
  - **Scoped**: nên dùng cho mỗi lần gọi riêng nếu cần ngữ cảnh riêng theo tenant hay request.

Vì cần các dịch vụ serverless của mình vận hành tối ưu nhất, ta phải viết code theo hướng nhẹ nhàng và portable nhất có thể. Tiếp theo, hãy xem xét các tình huống cần tới durability và quản lý trạng thái.

---

## Phần 6: Thiết kế durable orchestration

Với những workflow trải qua nhiều bước, hoặc cần retry, timeout, hay logic bù trừ (compensation logic), việc dùng framework orchestration serverless như **Azure Durable Functions** được khuyến nghị mạnh mẽ. Durable Functions là phần mở rộng của Azure Functions, cho phép điều phối các workflow có trạng thái trong môi trường serverless — bạn viết tiến trình chạy dài như code thông thường, còn nền tảng Azure tự quản lý trạng thái workflow, lịch sử sự kiện, và độ tin cậy thực thi.

Function serverless truyền thống phù hợp nhất cho tác vụ đơn giản, đơn mục đích. Nhưng nhiều tình huống nghiệp vụ thực tế đòi hỏi:

- Thực thi tuần tự nhiều bước.
- Thực thi song song các tác vụ độc lập.
- Chờ đầu vào con người hoặc sự kiện bên ngoài.
- Thao tác chạy dài (hàng giờ hoặc hàng ngày).
- Chính sách retry và bù trừ lỗi.

Durable Functions giúp thực hiện điều này mà không cần workflow engine ngoài hay hệ thống điều phối message phức tạp. Có nhiều kiểu triển khai durable function, mỗi kiểu giải quyết một nhu cầu riêng:

### Fan-out/fan-in

Pattern này dùng khi nhiều tác vụ có thể chạy song song, và hệ thống phải đợi tất cả hoàn tất trước khi tiếp tục. **Fan-out** là việc khởi chạy đồng thời nhiều function; **fan-in** là việc gộp kết quả hoặc đơn giản là chờ tất cả hoàn tất.

Trong hệ thống HealthCare, pattern này lý tưởng cho các thao tác gồm nhiều tiến trình độc lập nhưng liên quan, cần chạy đồng thời:

- Sau khi có kết quả xét nghiệm cuối cùng, fan-out để: thông báo bệnh nhân, thông báo bác sĩ phụ trách, cập nhật hệ thống hồ sơ y tế.
- Khi tiếp nhận bệnh nhân mới: khởi tạo cổng thông tin bệnh nhân, tạo hồ sơ sức khỏe điện tử (EHR), sinh tin nhắn chào mừng.

Vì pattern này cho phép thực thi song song, thời gian chạy của function sẽ bằng đúng thời gian của tác vụ dài nhất trong chuỗi. Durable Functions đảm bảo ngay cả khi một tác vụ song song thất bại, orchestrator vẫn có thể retry hoặc thực hiện hành động bù trừ mà không làm mất trạng thái tổng thể của workflow.

### Function chaining

Pattern này cho phép các tác vụ thực thi theo thứ tự nghiêm ngặt, output của function này trở thành input của function tiếp theo. Durable orchestrator quản lý từng bước, đảm bảo đúng thứ tự và truyền giá trị chính xác giữa các tác vụ.

Trong bối cảnh HealthCare, function chaining hữu ích cho:

- **Workflow đăng ký bệnh nhân**: đăng ký thông tin bệnh nhân → xác minh danh tính và bảo hiểm → gán bác sĩ chăm sóc chính → gửi email kích hoạt cổng thông tin.
- **Workflow thanh toán**: tạo hóa đơn → gửi tới bảo hiểm → ghi nhận số dư bệnh nhân → lên lịch nhắc thanh toán.

Pattern này tăng cường tính rõ ràng và khả năng kiểm toán, cho phép dễ dàng truy vết từng bước trong hành trình của bệnh nhân, đồng thời đảm bảo tuân thủ quy tắc nghiệp vụ.

### Chờ tương tác con người / sự kiện bên ngoài

Nhiều quy trình y tế đòi hỏi quyết định hay phê duyệt của con người trước khi tiếp tục. Method `WaitForExternalEvent` trong durable function cho phép workflow tạm dừng cho tới khi nhận được một sự kiện cụ thể, mà không tiêu tốn tài nguyên compute trong lúc chờ.

Pattern này có giá trị trong các tình huống như:

- **Ủy quyền kê đơn thuốc**: hệ thống tạm dừng cho tới khi bác sĩ xác nhận đơn thuốc, đặc biệt với các chất kiểm soát.
- **Sự đồng ý của bệnh nhân**: orchestrator chờ bệnh nhân chấp nhận hoặc từ chối phác đồ điều trị đề xuất.
- **Tiền phê duyệt bảo hiểm**: hệ thống bên ngoài (API của công ty bảo hiểm hay webhook) thông báo cho orchestrator khi yêu cầu bồi thường được duyệt hoặc từ chối.

Với mô hình này, workflow vẫn ở trạng thái idle nhưng đáng tin cậy, tiếp tục ngay lập tức khi nhận được tín hiệu hay đầu vào cần thiết — loại bỏ việc polling không cần thiết và đảm bảo hệ thống chỉ tiến tới khi thực sự đã sẵn sàng.

### Monitor pattern

Pattern này cho phép durable function liên tục kiểm tra một điều kiện, lặp lại cho tới khi điều kiện được thỏa mãn hoặc hết thời gian chờ (timeout). Điều này rất quan trọng trong y tế để đảm bảo theo dõi đúng hạn các tác vụ có deadline.

Use case gồm:

- **Xác nhận lịch hẹn**: sau khi đặt lịch, hệ thống kiểm tra mỗi giờ xem bệnh nhân đã xác nhận chưa, hủy slot nếu không nhận được phản hồi trong 24 giờ.
- **Theo dõi lấy thuốc**: giám sát log nhà thuốc để xác nhận việc lấy thuốc trong khung thời gian định sẵn.
- **Tuân thủ sau xuất viện**: xác minh các hành động theo dõi (như nộp kết quả xét nghiệm hay khảo sát) đã hoàn tất sau khi bệnh nhân xuất viện.

Bằng cách mô hình hóa các kiểm tra này như vòng lặp có điều phối, pattern này cung cấp sự giám sát tự động cho các hoạt động quan trọng, giảm thiểu rủi ro sai sót con người hay bỏ lỡ deadline.

### Asynchronous HTTP API

Durable Functions cũng hỗ trợ pattern HTTP API bất đồng bộ, cho phép client kích hoạt workflow chạy dài và nhận về một endpoint trạng thái để poll cho tới khi hoàn tất. Đặc biệt hữu ích với các thao tác vượt quá ngưỡng timeout HTTP thông thường.

Use case trong y tế:

- **Xuất dữ liệu bệnh nhân**: tổng hợp và xuất toàn bộ hồ sơ y tế, kết quả xét nghiệm, lịch hẹn thành file tải về cho bệnh nhân hoặc kiểm toán viên.
- **Báo cáo hàng loạt**: sinh báo cáo tổng hợp cuối tháng, trích xuất hóa đơn, hay kiểm toán đảm bảo chất lượng.
- **Pipeline ẩn danh hóa dữ liệu**: chuẩn bị bộ dữ liệu đã khử danh tính cho nghiên cứu lâm sàng hay chia sẻ dữ liệu.

Luồng chung của kiểu function này:

1. Client khởi tạo workflow bằng HTTP POST.
2. Server phản hồi ngay lập tức với `202 Accepted` cùng URL để kiểm tra trạng thái.
3. Workflow chạy độc lập, cập nhật tiến độ.
4. Khi hoàn tất, client có thể tải kết quả cuối cùng hoặc nhận webhook thông báo hoàn thành.

Pattern này đảm bảo trải nghiệm không chặn (non-blocking) cho client, và cho phép hệ thống serverless quản lý các tiến trình tốn nhiều tài nguyên một cách đáng tin cậy.

**Bài học cốt lõi: Durable Functions không phải là "một tính năng thêm cho vui" — nó là mảnh ghép bắt buộc để serverless (vốn stateless theo bản chất) có thể mô hình hóa các quy trình nghiệp vụ thực tế nhiều bước, cần chờ đợi, và cần đảm bảo tính toàn vẹn xuyên suốt.**

---

## Phần 7: Observability, bảo mật, và tích hợp trong serverless

### Observability, logging, và monitoring

Ứng dụng serverless chia nhỏ logic nghiệp vụ thành các đơn vị riêng biệt, khiến observability trở thành khía cạnh sống còn. Bản chất tạm thời (ephemeral) và phân tán của Azure Functions cùng Durable Functions làm phức tạp hóa việc theo dõi lỗi, đo hiệu năng, và chẩn đoán workflow.

Azure Functions tích hợp liền mạch với **Application Insights**, cho phép thu thập telemetry thời gian thực về lần thực thi function, lỗi, dependency, và sự kiện tùy chỉnh. Khi dùng Durable Functions, các trace orchestration — bao gồm điểm bắt đầu function, thực thi activity, lỗi, và retry — được log kèm correlation ID (nhắc lại correlation ID từ Chapter 9 và Chapter 12), cho phép developer theo dõi toàn bộ vòng đời của workflow chạy dài từ đầu tới cuối:

- **Trace và metric**: mỗi lần thực thi durable orchestration sinh ra sự kiện và log tùy chỉnh gửi tới Application Insights, có thể truy vấn bằng Kusto Query Language (KQL) để tìm xu hướng lỗi, vấn đề hiệu năng, hay đợt tăng đột biến sử dụng.
- **Live metrics stream**: developer có thể quan sát workflow diễn ra theo thời gian thực, theo dõi tần suất gọi, tỷ lệ thành công, cold start, và thời gian thực thi.

Trong hệ thống y tế, các thực hành này đảm bảo tuân thủ quy định, cải thiện phân tích nguyên nhân gốc rễ khi có gián đoạn dịch vụ, và giúp đáp ứng SLA vận hành trong môi trường phức tạp, được quản lý chặt.

### Cân nhắc bảo mật

Bảo mật là trụ cột nền tảng của mọi hệ thống, càng đúng hơn khi dữ liệu nhạy cảm và quy trình được quản lý chặt cần được bảo vệ nghiêm ngặt. Dù kiến trúc serverless trừu tượng hóa phần lớn hạ tầng, identity, kiểm soát truy cập, và quản lý secret vẫn là trách nhiệm của developer.

**Azure managed identity** cho phép function serverless xác thực an toàn với các resource Azure khác mà không cần quản lý credential thủ công. Ví dụ, một Azure Function có thể dùng identity của chính nó để:

- Đọc secret cấu hình (như key database) từ Key Vault.
- Truy cập Azure Storage hay Cosmos DB qua Role-Based Access Control (RBAC).
- Publish message tới queue hay topic của Service Bus.

Với các giá trị nhạy cảm như API key, encryption key, và chứng chỉ — không bao giờ nên hardcode hay lưu trong biến môi trường. Thay vào đó:

- Lưu trong Azure Key Vault.
- Truy cập an toàn qua managed identity của function.
- Cache theo từng lần gọi để giảm độ trễ.

Thiết kế microservice serverless với quyền truy cập tối thiểu (minimal-privilege access), ranh giới identity được định nghĩa rõ ràng, và audit toàn diện mọi service qua log của Entra ID.

### Công nghệ tích hợp và trigger

Việc xây dựng ứng dụng serverless đòi hỏi thiết kế cẩn thận, cân nhắc tới thực thi hướng sự kiện, các tùy chọn tích hợp sẵn có, và trigger liên kết function thành workflow lớn hơn. Làm tốt, ứng dụng serverless mang lại khả năng co giãn, khả năng phục hồi, và tiết kiệm chi phí với nỗ lực vận hành tối thiểu. Bỏ qua khâu này, chúng nhanh chóng trở thành một tập hợp function rời rạc, mong manh, khó bảo trì.

Cốt lõi của serverless computing là khái niệm: code chạy để phản hồi một sự kiện cụ thể — có thể đơn giản như một HTTP request, hoặc phức tạp như một message xuất hiện trên queue, một timer hết hạn, hay một file được upload lên storage. Mô hình hướng sự kiện này thay đổi cách ta nghĩ về việc điều phối service. Thay vì lời gọi đồng bộ truyền thống (một service gọi trực tiếp service khác), hệ thống serverless thường dùng dịch vụ tích hợp như Azure Queue Storage, Service Bus, hay Event Grid — tách rời producer khỏi consumer, hỗ trợ giao tiếp bất đồng bộ.

Sau khi xác định loại tích hợp, bạn chọn **trigger** — cơ chế quyết định khi nào function chạy. **Binding** cung cấp liên kết khai báo (declarative) tới tài nguyên bên ngoài như queue, table, hay blob storage. Cả hai giúp ẩn đi boilerplate code cho việc tích hợp, cho phép developer tập trung vào logic nghiệp vụ. Function hiếm khi tồn tại đơn độc — sức mạnh của chúng đến từ việc tích hợp với các dịch vụ Azure khác, và chọn đúng công nghệ tích hợp quan trọng không kém việc viết code cho function.

Ví dụ, một function với HTTP trigger có thể xác minh dữ liệu đăng ký của bệnh nhân, rồi gửi message vào queue. Một function thứ hai với queue trigger phản ứng lại message đó, điều phối các tác vụ phía sau như lưu dữ liệu vào Azure Table Storage và gửi email xác nhận. Kiểu chain trigger và binding này thiết lập sự phân tách trách nhiệm rõ ràng: function đầu tiên xử lý việc nhận và validate request, function thứ hai quản lý việc xử lý bất đồng bộ, bền vững.

**Bài học cốt lõi: Thiết kế microservices với tư duy serverless không chỉ là "chuyển code sang Azure Functions" — nó đòi hỏi tư duy lại độ chi tiết của service, DI cẩn trọng để tăng khả năng tái sử dụng/test, và áp dụng framework orchestration mạnh mẽ như Durable Functions để mô hình hóa đúng workflow thực tế.**

---

## Phần 8: Xây dựng serverless microservices trên Microsoft Azure — thực hành

Một trong những lợi ích chính của phát triển serverless microservices trên Azure là mô hình **pay-per-execution**: chỉ trả tiền khi function chạy, và Azure tự động co giãn dịch vụ theo tải. Điều này lý tưởng cho hệ thống y tế, nơi mức sử dụng khó lường — tăng vọt vào mùa đăng ký bảo hiểm hay mùa cúm, và tĩnh lặng hơn vào các thời điểm khác.

### Vì sao phát triển local trước khi lên Azure

Dù Azure cung cấp môi trường toàn cầu mạnh mẽ để chạy workload serverless, deploy trực tiếp lên cloud ngay trong giai đoạn phát triển hiếm khi là cách hiệu quả hay tiết kiệm chi phí nhất. Một quy trình phát triển kỷ luật, bắt đầu từ local, mang lại một số lợi ích riêng biệt:

**Tốc độ**: với các công cụ như Azure Functions Core Tools, Azurite (giả lập Azure Storage), và các emulator cho Service Bus hay Queue Storage, bạn có thể phát triển và test function mà không phải chờ deploy lên cloud hay chịu độ trễ mạng. Thay đổi được test tức thì, tạo ra vòng lặp phản hồi cực kỳ hiệu quả — quan trọng với workflow phức tạp, durable function, hay các điểm tích hợp như message queue và storage table, nơi thử-sai đóng vai trò lớn. Tuy vậy, cần nhớ rằng môi trường local không đại diện hoàn toàn cho môi trường production thực tế — trong production, bạn phải đối mặt với cold start, virtual network, độ trễ, và nhiều tình huống khác không thể tái tạo ở local.

**Hiệu quả chi phí**: dù function serverless tính phí theo lần chạy và có thể tiết kiệm chi phí ở quy mô lớn, việc redeploy và trigger lặp đi lặp lại trong quá trình phát triển có thể nhanh chóng tăng chi phí tài nguyên. Dùng emulator local nghĩa là tránh chi phí Azure cho tới khi ứng dụng sẵn sàng cho staging hay test production — đặc biệt hấp dẫn cho prototype giai đoạn đầu hay chu kỳ phát triển dài.

**Nhất quán đa nền tảng**: dù phát triển trên Windows, macOS, hay Linux, Azure Functions Core Tools và các emulator hỗ trợ hành xử giống nhau trên các hệ điều hành khác nhau. Toàn team có thể làm việc trong môi trường ưa thích mà không lo vấn đề đặc thù môi trường, và có thể làm việc offline — điều không thể nếu mỗi lần test đều cần deploy lên cloud.

### Bộ công cụ local được khuyến nghị

Để tối ưu lợi ích của phát triển local trước khi deploy lên Azure, cần một bộ công cụ miễn phí, đa nền tảng đáng tin cậy — giả lập các dịch vụ chính của Azure ngay trên máy local, mang lại trải nghiệm đồng nhất trên các hệ điều hành khác nhau:

- **Visual Studio Code**: editor nhẹ, mở rộng tốt, tích hợp phong phú với Azure Functions — lý tưởng cho phát triển serverless đa nền tảng. Cài extension Azure Functions để có template, debug, và tính năng deploy. Extension **Azure Tools** cung cấp toàn bộ các extension liên quan tới Azure mà bạn có thể cần cho các hình thức phát triển khác.
- **Azure Functions Core Tools**: runtime cho phép tạo, chạy, và debug Azure Functions ngay tại local, hỗ trợ đầy đủ trigger và binding, có mặt trên mọi hệ điều hành.
- **Azurite (Azure Storage Emulator)**: giả lập Azure Storage tại local — mô phỏng queue, table, và blob, test mà không cần cấp phát resource trên cloud. Có thể chạy qua Docker:

```bash
docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 \
  -v azurite_data:/data mcr.microsoft.com/azure-storage/azurite
```

*(Callback tới Chapter 14: đây chính là ví dụ điển hình cho việc dùng Docker để chạy một dịch vụ bên thứ ba — y hệt cách bạn từng `docker pull redis` để giả lập cache local.)*

- **Azure Storage Explorer**: giao diện để tương tác trực tiếp với resource trong emulator.

Tóm lại, dù nền tảng hosting của Azure là đích deploy chính, các công cụ phát triển local mới là xương sống cho một quy trình serverless nhanh, an toàn, và tiết kiệm chi phí.

### Xây dựng dịch vụ đặt lịch hẹn trên Azure Functions

Khởi tạo project mới:

```bash
mkdir AppointmentBooking && cd AppointmentBooking
func init --worker-runtime dotnet-isolated --target-framework net8.0
```

Cài các package cần thiết để hỗ trợ Azure Storage table, queue, và durable task:

```bash
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Storage.Queues
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Tables
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

Package durable task được thêm vào để hỗ trợ một tiến trình được orchestrate, ghi vào Azure Storage table và gửi hai email — giúp đạt được độ tin cậy, fan-out/fan-in, và lịch sử thực thi với độ phức tạp code tối thiểu.

### Cấu hình `local.settings.json`

File này cho phép chỉ định tham số cho phát triển local — không áp dụng khi ứng dụng đã publish:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    // Durable Functions storage (uses the same Azurite account)
    "AzureWebJobsStorage__serviceUri": "",
    "Host": {
      "LocalHttpPort": 7071,
      "CORS": "*"
    }
  }
}
```

Phân tích kỹ file này: `IsEncrypted: false` — hợp lý cho phát triển local; trong production, cấu hình được lưu an toàn qua Azure App Configuration hoặc Key Vault, để Azure tự quản lý mã hóa thay vì dựa vào file này. Mục `Values` chứa toàn bộ cặp key-value cần thiết cho Functions runtime và ứng dụng của bạn — khi chạy local, chúng được inject vào tiến trình y hệt biến môi trường trên cloud. Mục `Host` là cấu hình riêng cho chính Functions host (runtime local) — không liên quan tới logic ứng dụng, mà là cách emulator Functions hành xử khi bạn chạy lệnh `func start`.

`AzureWebJobsStorage` với giá trị `UseDevelopmentStorage=true` báo cho Functions runtime dùng Azurite — emulator Azure Storage local. Trên Azure thật, giá trị này sẽ là connection string thật của storage account.

`FUNCTIONS_WORKER_RUNTIME` định nghĩa mô hình runtime `dotnet-isolated`. Azure Functions hỗ trợ hai mô hình cho .NET:

- **In-process model**: function thực thi trong chính tiến trình host của Azure Functions.
- **Isolated worker model**: function chạy trong tiến trình riêng biệt, mang lại khả năng cô lập, quản lý dependency tốt hơn, và hỗ trợ đầy đủ .NET 8.

Vì đang tập trung vào phát triển serverless hiện đại, `dotnet-isolated` là lựa chọn được khuyến nghị.

`AzureWebJobsStorage__serviceUri` là cấu hình bổ sung đôi khi dùng bởi Durable Functions, cho phép ghi đè URI storage nơi lưu trạng thái orchestration. Ở local, để trống vì Durable Functions dùng Azurite; ở production, có thể trỏ tới một endpoint storage account riêng nếu muốn tách dữ liệu orchestration khỏi queue/blob thông thường.

### Định nghĩa model dữ liệu

```csharp
// Models/Patient.cs
public record Patient(string FirstName, string LastName,
    string Email, DateTime DateOfBirth);

// Models/Appointment.cs
public record Appointment(DateTime StartsAtUtc,
    TimeSpan Duration, string ProviderId, string? Location = null);

// Models/BookingRequest.cs
// The HTTP contract we accept and pass through the queue/orchestrator.
public record BookingRequest(Patient Patient, Appointment Appointment);
```

### Model entity cho Azure Table Storage

```csharp
// Models/PatientTableEntity.cs
using Azure;
using Azure.Data.Tables;

public class PatientTableEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "Patients";
    public string RowKey { get; set; } = Guid.NewGuid().ToString("N");
    public string FirstName { get; set; } = default!;
    public string LastName { get; set; } = default!;
    public string Email { get; set; } = default!;
    public DateTime DateOfBirth { get; set; }
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }
}

// Models/AppointmentTableEntity.cs
public class AppointmentTableEntity : ITableEntity
{
    public string PartitionKey { get; set; }
    public string RowKey { get; set; } = Guid.NewGuid().ToString("N");
    public string PatientEmail { get; set; } = default!;
    public DateTime StartsAtUtc { get; set; }
    public TimeSpan Duration { get; set; }
    public string ProviderId { get; set; } = default!;
    public string? Location { get; set; }
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }
}
```

`PartitionKey` là khái niệm quan trọng cần hiểu kỹ: Azure Table Storage kết hợp `PartitionKey` với `RowKey` để định danh duy nhất mỗi dòng. `PartitionKey` là chuỗi xác định partition mà entity được lưu — mọi entity cùng `PartitionKey` nằm chung một partition logic, và partition chính là đơn vị co giãn (scalability) của Table Storage.

**Analogy: Bảng dữ liệu như một thư viện, partition như từng kệ sách.** `PartitionKey` chính là nhãn nói rằng cuốn sách (dòng dữ liệu) này thuộc kệ nào. Cách tổ chức này cải thiện đáng kể tốc độ đọc/ghi, vì Azure có thể định vị đúng "kệ sách" cần tìm thay vì quét toàn bộ thư viện.

### Helper truy cập Table Storage dùng chung

```csharp
// Infrastructure/TableStorage.cs
using Azure.Data.Tables;

public static class TableStorage
{
    public static TableClient GetClient(string tableName)
    {
        var conn =
            Environment.GetEnvironmentVariable("AzureWebJobsStorage")
            ?? "UseDevelopmentStorage=true";
        var service = new TableServiceClient(conn);
        var client = service.GetTableClient(tableName);
        client.CreateIfNotExists();
        return client;
    }
}
```

### Function HTTP trigger — điểm vào của luồng đặt lịch

Function này nhận POST, validate, rồi đẩy message vào queue để xử lý nền:

```csharp
// Functions/StartBookingHttp.cs
using System.Net;
using System.Text.Json;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;

public class StartBookingHttp
{
    private const string QueueName = "appointments";
    private readonly QueueClient _queue;
    private bool isValid = true;

    public StartBookingHttp()
    {
        var conn =
            Environment.GetEnvironmentVariable("AzureWebJobsStorage");
        // Ensure consistent Base64 message encoding
        var options = new QueueClientOptions { MessageEncoding =
            QueueMessageEncoding.Base64 };
        _queue = new QueueClient(conn, QueueName, options);
        _queue.CreateIfNotExists();
    }

    [Function("StartBookingHttp")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post")]
        HttpRequestData req,
        FunctionContext ctx)
    {
        BookingRequest? payload;
        try
        {
            var body = await new StreamReader(req.Body).ReadToEndAsync();
            payload = JsonSerializer.Deserialize<BookingRequest>(body,
                new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        }
        catch
        {
            return await BadRequest(req, "Malformed JSON.");
        }

        if (string.IsNullOrWhiteSpace(payload.Patient.FirstName))
            isValid = false;
        if (string.IsNullOrWhiteSpace(payload.Patient.LastName))
            isValid = false;
        if (!isValid) return await BadRequest(req, "Validation failed");

        var json = JsonSerializer.Serialize(payload);
        await _queue.SendMessageAsync(json);

        var ok = req.CreateResponse(HttpStatusCode.Accepted);
        await ok.WriteStringAsync("Booking request accepted and queued for processing.");
        return ok;
    }

    private static async Task<HttpResponseData> BadRequest(
        HttpRequestData req, string msg)
    {
        var res = req.CreateResponse(HttpStatusCode.BadRequest);
        await res.WriteStringAsync(msg);
        return res;
    }
}
```

Chú ý: function này chỉ làm đúng một việc — nhận request, validate sơ bộ, và đẩy vào queue. Nó không tự mình tạo bệnh nhân hay gửi email — đây chính là nguyên tắc "một function, một trách nhiệm" đã bàn ở Phần 5, và cũng là ranh giới rõ ràng giữa xử lý đồng bộ (validate nhanh, trả response ngay) và xử lý bất đồng bộ (mọi việc nặng nhọc phía sau).

### Function queue trigger — khởi động orchestration

```csharp
// Functions/StartBookingQueue.cs
using System.Text.Json;
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask.Client;

public class StartBookingQueue
{
    [Function("StartBookingQueue")]
    public async Task Run(
        [QueueTrigger("appointments")] string message,
        [DurableClient] DurableTaskClient client,
        FunctionContext ctx)
    {
        BookingRequest? request = null;
        try
        {
            request = JsonSerializer.Deserialize<BookingRequest>(message,
                new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        }
        catch (Exception ex)
        {
            return;
        }

        if (request is null)
        {
            return;
        }

        var instanceId = await
            client.ScheduleNewOrchestrationInstanceAsync(
                nameof(BookingOrchestrator), request);
    }
}
```

Function này lắng nghe queue `appointments`, và khi có message, nó không tự xử lý nghiệp vụ mà giao việc cho một orchestrator — đây chính là điểm chuyển giao từ "phản ứng với một sự kiện đơn lẻ" sang "điều phối một workflow nhiều bước".

### Orchestrator — trái tim của workflow

```csharp
// Functions/BookingOrchestrator.cs
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;

public class BookingOrchestrator
{
    [Function(nameof(BookingOrchestrator))]
    public async Task Run([OrchestrationTrigger]
        TaskOrchestrationContext ctx)
    {
        var request = ctx.GetInput<BookingRequest>()!;

        // Create patient
        await ctx.CallActivityAsync(nameof(AddPatientActivity),
            request.Patient);

        // Create appointment
        await ctx.CallActivityAsync(nameof(AddAppointmentActivity),
            request);

        // Fan-out: notify patient & admin
        var t1 = ctx.CallActivityAsync(nameof(SendPatientEmailActivity),
            request);
        var t2 = ctx.CallActivityAsync(nameof(SendAdminEmailActivity),
            request);
        await Task.WhenAll(t1, t2);
    }
}
```

Đây chính là ví dụ sống động cho cả hai pattern đã học ở Phần 6: hai bước đầu (`AddPatientActivity`, `AddAppointmentActivity`) chạy **tuần tự theo kiểu function chaining** — vì tạo lịch hẹn cần dữ liệu bệnh nhân đã tồn tại trước. Hai bước cuối (gửi email cho bệnh nhân và admin) chạy theo **fan-out/fan-in** — vì chúng độc lập với nhau, `Task.WhenAll` đảm bảo orchestrator chỉ hoàn tất khi cả hai email đều đã gửi xong.

### Các activity function thực thi từng bước

```csharp
// Functions/AddPatientActivity.cs
using Azure.Data.Tables;
using Microsoft.Azure.Functions.Worker;

public class AddPatientActivity
{
    [Function(nameof(AddPatientActivity))]
    public async Task Run([ActivityTrigger] Patient patient,
        FunctionContext ctx)
    {
        var logger = ctx.GetLogger(nameof(AddPatientActivity));
        TableClient client = TableStorage.GetClient("Patients");
        var entity = new PatientTableEntity
        {
            FirstName = patient.FirstName,
            LastName = patient.LastName,
            Email = patient.Email,
            DateOfBirth = patient.DateOfBirth
        };
        await client.AddEntityAsync(entity);
        logger.LogInformation("Patient added: {Email}", patient.Email);
    }
}
```

```csharp
// Functions/AddAppointmentActivity.cs
using Azure.Data.Tables;
using Microsoft.Azure.Functions.Worker;

public class AddAppointmentActivity
{
    [Function(nameof(AddAppointmentActivity))]
    public async Task Run([ActivityTrigger] BookingRequest request,
        FunctionContext ctx)
    {
        var logger = ctx.GetLogger(nameof(AddAppointmentActivity));
        TableClient client = TableStorage.GetClient("Appointments");
        var dateKey = request.Appointment.StartsAtUtc.ToString("yyyyMMdd");
        var entity = new AppointmentTableEntity
        {
            PartitionKey = dateKey,
            PatientEmail = request.Patient.Email,
            StartsAtUtc = request.Appointment.StartsAtUtc,
            Duration = request.Appointment.Duration,
            ProviderId = request.Appointment.ProviderId,
            Location = request.Appointment.Location
        };
        await client.AddEntityAsync(entity);
        logger.LogInformation("Appointment added for {Email} at {Start}",
            request.Patient.Email, request.Appointment.StartsAtUtc);
    }
}
```

Chú ý cách `AddAppointmentActivity` chọn `PartitionKey` là ngày diễn ra lịch hẹn (`yyyyMMdd`) — đây là một quyết định thiết kế có chủ đích: mọi lịch hẹn cùng ngày nằm chung một "kệ sách", giúp truy vấn "tất cả lịch hẹn hôm nay" cực kỳ nhanh, vì Azure chỉ cần quét đúng một partition thay vì toàn bộ bảng.

```csharp
// Functions/SendPatientEmailActivity.cs
using Microsoft.Azure.Functions.Worker;

public class SendPatientEmailActivity
{
    [Function(nameof(SendPatientEmailActivity))]
    public async Task Run([ActivityTrigger] BookingRequest request,
        FunctionContext ctx)
    {
        var logger = ctx.GetLogger(nameof(SendPatientEmailActivity));
        // Send Email logic
    }
}

// Functions/SendAdminEmailActivity.cs
using Microsoft.Azure.Functions.Worker;

public class SendAdminEmailActivity
{
    [Function(nameof(SendAdminEmailActivity))]
    public async Task Run([ActivityTrigger] BookingRequest request,
        FunctionContext ctx)
    {
        var logger = ctx.GetLogger(nameof(SendAdminEmailActivity));
        // Send Email logic
    }
}
```

### Chạy và test local

Đảm bảo container Azurite đang chạy, sau đó khởi động Functions host:

```bash
func start
```

Test function HTTP trigger bằng cURL (hoặc Postman):

```bash
curl -X POST http://localhost:{PORT_NUMBER}/api/StartBookingHttp \
  -H "Content-Type: application/json" \
  -d '{
    "patient": { "firstName":"Ava","lastName":"Nguyen","email":"ava@example.com","dateOfBirth":"1992-05-16T00:00:00Z" },
    "appointment": { "startsAtUtc":"2025-08-25T14:00:00Z","duration":"01:00:00","providerId":"dr-42","location":"Room 3" }
  }'
```

Kiểm chứng luồng hoạt động bằng cách xem log trong console và xác minh dữ liệu đã được ghi vào Azure Table ở cuối quy trình.

**Bài học cốt lõi: Việc chia dịch vụ đặt lịch hẹn thành HTTP trigger (nhận + validate nhanh) → queue trigger (khởi động orchestration) → orchestrator (điều phối chaining + fan-out/fan-in) → các activity function (thực thi từng bước đơn lẻ) chính là hiện thân đầy đủ của mọi nguyên tắc thiết kế serverless đã học: function nhỏ, một trách nhiệm, liên kết lỏng qua trigger/binding, và trạng thái được durable orchestrator quản lý thay vì tự tay xử lý.**

---

## Tổng kết chương

Chương này khảo sát cách serverless computing cách mạng hóa việc phát triển microservices, mang lại một mô hình linh hoạt, tiết kiệm chi phí, và co giãn tốt so với các mô hình hosting truyền thống. Chúng ta thấy Azure Functions và Durable Functions có thể áp dụng vào các domain thực tế như hệ thống y tế — nơi workflow đòi hỏi cả hiệu quả lẫn khả năng phục hồi — và cân bằng giữa lợi ích của kiến trúc serverless với những thách thức cố hữu đi kèm, thông qua chiến lược, pattern, và ví dụ thực tế cho developer .NET.

Chúng ta nhấn mạnh rằng thiết kế cho serverless đòi hỏi nhiều hơn việc chỉ di dời workload hiện có sang môi trường mới. Độ chi tiết của function là quyết định thiết kế then chốt — function nên bám sát nguyên tắc single responsibility, có tính cố kết, và tránh chain quá đà (vốn gây độ trễ và khó khăn khi tracing). Chúng ta cũng nhấn mạnh quy ước đặt tên và thực hành dependency injection, đặc biệt chú ý tới isolated worker model trong Azure Functions — mô hình bám sát quy ước ASP.NET Core trong khi vẫn hỗ trợ đầy đủ tính năng .NET hiện đại.

Sau khi khảo sát các công nghệ serverless của Azure, chúng ta thiết lập môi trường phát triển local dùng Visual Studio Code với extension Azure, Azure Functions Core Tools, và emulator Azurite — cho phép xây dựng và test ứng dụng serverless đa nền tảng mà không tốn chi phí cloud. Quy trình local-first này tăng cường khả năng cộng tác và sự tự tin trước khi triển khai thật.

Hệ thống đặt lịch hẹn của HealthCare đóng vai trò case study xuyên suốt. Nó cho thấy cách function HTTP trigger thu thập đầu vào của bệnh nhân, thực hiện validate, và đẩy request vào queue để xử lý nền. Orchestration kích hoạt bởi queue sau đó điều phối các activity như tạo bệnh nhân, đặt lịch hẹn, và gửi email thông báo cho cả bệnh nhân lẫn quản trị viên. Azure Storage Table được dùng để lưu trữ dữ liệu, với partition key đảm bảo khả năng co giãn và truy vấn hiệu quả. Hệ thống minh họa cách kết hợp trigger, durable orchestration, và thực thi fan-out tạo ra các workflow có khả năng phục hồi cao.

**Bài học cốt lõi: Serverless microservices là một mô hình mạnh mẽ nhưng đầy sắc thái — lợi ích về khả năng co giãn, sự linh hoạt, và giảm gánh nặng hạ tầng là rõ ràng, nhưng đi kèm với thách thức về độ trễ (cold start), tính bền vững của trạng thái, và vendor lock-in. Thông qua Durable Functions, độ chi tiết function có chủ đích, cùng thực hành observability và bảo mật vững chắc, team có thể vượt qua những giới hạn này — và việc chọn dùng serverless hay container hóa (Chapter 14) hay kết hợp cả hai, luôn phải xuất phát từ đặc tính thực tế của từng workload, không phải từ trào lưu công nghệ.**

Ở chương tiếp theo, chúng ta sẽ chuyển sang khảo sát các mối quan tâm xuyên suốt (cross-cutting concerns) liên quan tới observability và giám sát microservices — một chủ đề đã được nhắc tới nhiều lần trong chương này (Application Insights, correlation ID, trace) nhưng giờ sẽ được đào sâu như một trụ cột kiến trúc độc lập.
