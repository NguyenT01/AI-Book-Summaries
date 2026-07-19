# Bài học: Khi "đăng nhập một lần" không còn đủ — bảo mật trong một thế giới không biên giới cố định

> Chương 13 — Securing Microservices
> Ví dụ xuyên suốt: hệ thống quản lý phòng khám — lần này ta dựng một service xác thực trung tâm (HealthCare.Auth) dùng Duende IdentityServer, rồi bảo vệ cả API Appointments lẫn Gateway Ocelot đã xây ở Chapter 11 bằng OAuth 2.0/OIDC.

---

## Phần 1: Vấn đề thực tế — khi mỗi cánh cửa cần một chiếc chìa khóa riêng

Ở một hệ thống monolith, bảo mật thường tập trung: một lần đăng nhập, một session được tạo, và mọi phần của ứng dụng đều "biết" người dùng đó là ai vì tất cả nằm chung một tiến trình. Nhưng hãy hình dung ngày ta tách hệ thống quản lý phòng khám thành các service Patients, Appointments, Billing, Documents, Notifications — mỗi service giờ là một tiến trình độc lập, một điểm truy cập độc lập. Nếu một bác sĩ đăng nhập để xem lịch hẹn, rồi hệ thống cần gọi tiếp sang service Billing để tạo hóa đơn, thì AI xác nhận cho Billing biết yêu cầu này thực sự đến từ bác sĩ đó, chứ không phải từ một script độc hại giả mạo?

Đây chính là bản chất phức tạp của bảo mật microservices: những đặc tính khiến microservices hấp dẫn — độc lập, phi tập trung, linh hoạt về công nghệ (polyglot) — lại chính là những thứ làm phức tạp hóa các mô hình bảo mật truyền thống. Một hệ thống dựa trên microservices phải xác thực và ủy quyền request xuyên suốt một "chòm sao" các service, chứ không chỉ một lần duy nhất. Mỗi service không chỉ cần xác minh danh tính người dùng/client, mà còn phải tự quyết định thao tác được yêu cầu có được phép hay không — tất cả trong khi vẫn giữ độ trễ thấp và tính sẵn sàng cao.

Bảo mật ứng dụng (application security) nói chung là kỷ luật triển khai các biện pháp bảo vệ và thực hành tốt nhất để bảo vệ phần mềm khỏi các mối đe dọa và lỗ hổng xuyên suốt vòng đời — từ phát triển, triển khai, tới bảo trì. Mục đích là đảm bảo dữ liệu được bảo vệ khỏi truy cập, thay đổi, hay phá hủy trái phép, và ứng dụng hoạt động đáng tin cậy ngay cả khi đối mặt với các nỗ lực khai thác ác ý.

Trong một hệ thống monolith truyền thống, bảo mật thường tập trung và dễ thực thi hơn nhờ đơn vị triển khai duy nhất. Ngược lại, trong kiến trúc microservices, bảo mật ứng dụng vốn dĩ phức tạp hơn nhiều. Mỗi microservice là một tiến trình độc lập với dữ liệu, giao diện, và lỗ hổng tiềm ẩn riêng. Vì vậy, bảo mật phải được áp dụng không chỉ ở "rìa" hệ thống (gateway hay web UI), mà còn xuyên suốt các ranh giới giao tiếp nội bộ giữa các service.

### Analogy: Từ một tòa nhà có một bảo vệ ở cổng, tới một quần thể nhiều tòa nhà cần hộ chiếu riêng cho từng nơi

Nhớ lại analogy "tòa nhà văn phòng" ở Chapter 11: một quầy lễ tân chung giúp khách không cần biết công ty nào ở tầng nào. Nhưng bảo mật lại đặt ra một câu hỏi khác: quầy lễ tân đó xác minh danh tính khách bằng cách nào, và một khi khách đã vào bên trong, làm sao đảm bảo họ chỉ vào đúng phòng mình được phép, chứ không tự do đi lại khắp tòa nhà? Nếu tòa nhà monolith giống một khu phức hợp có MỘT cổng bảo vệ kiểm tra giấy tờ một lần, thì microservices giống một quần thể nhiều tòa nhà riêng biệt (mỗi tòa là một service) — bảo vệ ở tòa Kế toán không thể tự động tin tưởng ai đó chỉ vì họ đã qua được cổng tòa Nhân sự. Cần một "hộ chiếu" chung, được một cơ quan cấp phát trung ương công nhận, mà MỌI tòa nhà trong quần thể đều tin tưởng và biết cách xác minh.

---

## Phần 2: Ba trụ cột của bảo mật ứng dụng — nhắc lại nền tảng trước khi vào microservices

Trước khi đi sâu vào microservices, cần ôn lại các khái niệm bảo mật cốt lõi áp dụng cho MỌI ứng dụng, dù monolith hay phân tán:

- **Authentication (Xác thực)**: nằm ở trung tâm của mọi ứng dụng an toàn — là quá trình xác lập danh tính, thường dùng thông tin đăng nhập như username/password. Trong ứng dụng monolith, xác thực thường được triển khai MỘT LẦN và dùng chung xuyên suốt code base. Phương thức phổ biến nhất là xác thực dựa trên cookie: sau khi đăng nhập thành công, ứng dụng tạo một session hoặc phát hành một cookie đã ký, được lưu ở client và gửi kèm mọi request sau đó — token/session identifier này ánh xạ tới một session phía server duy trì ngữ cảnh của người dùng đã xác thực.

- **Authorization (Ủy quyền)**: một khi người dùng đã xác thực, lớp phòng thủ tiếp theo là ủy quyền — quyết định người dùng đó ĐƯỢC PHÉP làm gì. Trong ứng dụng monolith, ủy quyền thường thực thi qua role-based access control (RBAC) — người dùng được gán vào các vai trò (admin, doctor, nurse, clerk...), và quyền truy cập tính năng/tài nguyên được cấp dựa trên vai trò đó. Ví dụ, trong hệ thống y tế, một người dùng chỉ được xem hồ sơ bệnh nhân thuộc đúng khoa mình phụ trách — ràng buộc này có thể mã hóa qua custom claim (VD: department ID) và đánh giá qua một policy engine.

- **Data confidentiality và integrity**: bảo vệ dữ liệu cả khi lưu trữ (at rest) lẫn khi truyền (in transit) là yêu cầu nền tảng khác. Dữ liệu nhạy cảm — mật khẩu, hồ sơ y tế, thông tin thanh toán, PII (personally identifiable information) — phải được mã hóa hoặc hash. Mật khẩu, ví dụ, nên được hash bằng thuật toán an toàn để chống lại tấn công rainbow table. Với dữ liệu đang truyền, HTTPS (TLS) phải được thực thi để ngăn tấn công man-in-the-middle (MITM).

Ngoài ba trụ cột này, mức độ cần thiết của các biện pháp bảo mật bổ sung còn phụ thuộc vào ngành nghề và mục đích ứng dụng. Hệ thống quản lý phòng khám sẽ có yêu cầu bảo mật khác hẳn hệ thống quản lý trường học hay ngân hàng trực tuyến — cụ thể, ngành y tế chịu ràng buộc từ HIPAA (Health Insurance Portability and Accountability Act), ngành tài chính chịu PCI DSS (Payment Card Industry Data Security Standard), và bảo vệ dữ liệu nói chung chịu GDPR (General Data Protection Regulation). Một cách tiếp cận secure-by-design giúp đơn giản hóa việc tuân thủ và tránh các khoản phạt.

### Cái giá phải trả — bảo mật luôn đánh đổi với trải nghiệm và độ phức tạp

Bảo mật không miễn phí. Từ góc nhìn người dùng, nó thường đưa thêm ma sát vào luồng thao tác — xác thực đa yếu tố (MFA), session timeout, yêu cầu xác thực lại thường xuyên — không phải người dùng nào cũng thích các bước bổ sung này. Từ góc nhìn developer, bảo mật thường đòi hỏi thêm các lớp gián tiếp làm phức tạp hóa thiết kế hệ thống và tăng chi phí bảo trì — dev cần hiểu sâu về nguyên lý bảo mật, luồng OAuth 2.0, cách lưu token an toàn, thuật toán mã hóa. Cấu hình sai bảo mật — cổng mở, cấu hình CORS lỏng lẻo, vai trò quá rộng quyền — là một trong những lỗ hổng hàng đầu ở các ứng dụng cloud-native và microservices, thường phát sinh từ setup phức tạp và lỗi con người. Bảo mật triển khai kém đôi khi còn rủi ro hơn cả không có bảo mật, vì nó tạo cảm giác an toàn giả.

**Bài học cốt lõi: Bảo mật không phải một tính năng đơn lẻ mà là một kỷ luật xuyên suốt — càng phân tán hệ thống bao nhiêu, càng cần thiết kế bảo mật có chủ đích bấy nhiêu, thay vì "vá thêm" sau cùng.**

---

## Phần 3: Thách thức riêng khi bảo mật Microservices

Bảo mật microservices đặt ra một tập thách thức vượt xa phạm vi bảo mật ứng dụng truyền thống. Trong khi ứng dụng monolith thường tập trung hóa logic xác thực, ủy quyền, và kiểm soát truy cập, kiến trúc microservices lại phi tập trung hóa các mối quan tâm này ra nhiều service — khiến việc thực thi, quan sát, và duy trì tính nhất quán trở nên phức tạp hơn. Hơn nữa, vì microservices thường trải rộng trên nhiều mạng, môi trường thực thi, và đơn vị triển khai khác nhau, bề mặt tấn công tiềm tàng tăng lên đáng kể.

Vì mỗi microservice là một đơn vị tự trị thường phơi ra một hoặc nhiều endpoint qua HTTP hay gRPC, các endpoint này có thể là public hoặc internal — nhưng mỗi cái đều là một vector tấn công tiềm tàng. Khác với monolith có thể đứng sau MỘT gateway xác thực duy nhất, microservices có thể cần xác thực và ủy quyền không chỉ client bên ngoài, mà cả request service-to-service nội bộ — tạo ra một "ranh giới tin cậy phân tán" (distributed trust boundary) cần được quản lý chính xác.

Trong hệ thống quản lý phòng khám của ta, có các service cho hồ sơ bệnh nhân, đặt lịch hẹn, thanh toán, lưu trữ tài liệu, và thông báo. Mỗi service này cần xác minh danh tính người gọi và thực thi quy tắc kiểm soát truy cập tương ứng. Một request tạo hóa đơn chỉ nên thành công nếu người dùng/service gọi được ủy quyền xem thông tin thanh toán của đúng bệnh nhân được chỉ định. Ngay cả một service có vẻ vô hại như endpoint tải tài liệu lên cũng phải đảm bảo nội dung tải lên không thể bị khai thác để chèn malware hay ghi đè file trái phép.

### Ba mô hình xác thực phổ biến — và tại sao hai mô hình đầu không đủ

| Mô hình | Cách hoạt động | Hạn chế |
|---|---|---|
| API key authentication | Mỗi client được cấp một key duy nhất, gửi kèm mỗi request | Static, thiếu ngữ cảnh người dùng, dễ rò rỉ nếu bị log lộ, khó xoay vòng (rotate) mà không gián đoạn dịch vụ |
| Basic Authentication | Gửi username/password mã hóa Base64 kèm mỗi request | Credential thường sống lâu, lộ transport layer là lộ luôn secret; dù kèm HTTPS vẫn không cung cấp kiểm soát truy cập chi tiết hay khả năng audit |
| OAuth 2.0 với bearer token | Tách biệt xác thực khỏi ủy quyền, token được cấp bởi một identity provider (IdP) trung tâm; client trình token này cho microservice, service xác minh chữ ký và đọc claim để quyết định quyền truy cập | Phức tạp hơn để triển khai đúng chuẩn, nhưng mang lại khả năng mở rộng, chuẩn hóa, và kiểm soát chi tiết cao hơn nhiều |

Với OAuth 2.0 và bearer token, người dùng xác thực với IdP và nhận về một JSON Web Token (JWT) đã ký. Client trình token này cho microservices, các service xác minh tính xác thực chữ ký và đọc claim để xác định quyền truy cập. JWT cung cấp một phương thức gọn nhẹ, phi trạng thái (stateless) để truyền tải danh tính và metadata ủy quyền — claim có thể bao gồm ID người dùng, vai trò, scope, hay thậm chí thuộc tính tùy chỉnh như khoa/phòng ban hay hạng giấy phép. Service có thể xác minh tính xác thực của token mà KHÔNG cần truy vấn một database trung tâm — khiến phương pháp này có khả năng mở rộng cao.

---

## Phần 4: Bearer Token — cơ chế "phi trạng thái" cho một thế giới không session dùng chung

Trong một ứng dụng web điển hình, các yếu tố xác thực thường triển khai qua form authentication — thu thập thông tin định danh và kiểm tra khớp trong database. Khi tìm thấy khớp, ta khởi tạo một cơ chế lưu trữ tạm thời để đánh dấu người dùng đã xác thực. Cơ chế lưu trữ tạm này thường ở hai dạng:

- **Session**: lưu thông tin vào một biến dùng được xuyên suốt website. Khác với biến thông thường mất giá trị sau mỗi request, session giữ giá trị trong một khoảng thời gian cho tới khi hết hạn hoặc bị hủy. Session variable thường lưu ở server — có thể gây vấn đề bộ nhớ trên server kém mạnh khi có quá nhiều người dùng đăng nhập cùng lúc.
- **Cookie**: một lựa chọn thay thế, nơi một file nhỏ được tạo và lưu trên thiết bị người dùng — giảm tải cho server, chuyển trách nhiệm sang thiết bị người dùng.

Cả hai đều hoạt động tuyệt vời khi ta chắc chắn đang xử lý một ứng dụng web duy trì trạng thái (stateful). Nhưng chuyện gì xảy ra khi ta cần xác thực với API? Về bản chất, API không duy trì trạng thái — nó không cố gắng "nhớ" người dùng đang truy cập, vì API được thiết kế cho truy cập rời rạc từ bất kỳ kênh nào, bất kỳ lúc nào. Vì lý do này, ta triển khai bearer/access token — những mẩu thông tin tự chứa (self-contained), thường được ký mã hóa, có thể mang các claim xác thực và ủy quyền. Chúng được gửi kèm request API và xác minh bởi service nhận mà KHÔNG cần tham vấn một cơ quan trung tâm hay session store — mô hình này hỗ trợ khả năng mở rộng, phi trạng thái, và hiệu năng.

Tuy nhiên, quản lý xác thực bằng token cũng mang theo thách thức riêng: vì JWT thường tự chứa và KHÔNG THỂ thu hồi (non-revocable), chúng cần thời gian sống ngắn để giảm thiểu cửa sổ rủi ro nếu bị lộ — điều này đòi hỏi cơ chế refresh token, có thể phức tạp với luồng silent re-authentication. Ngoài ra, cần cẩn trọng để không làm lộ token trong log hay response lỗi.

---

## Phần 5: Giải phẫu một JWT — cấu trúc, claim, và cách đọc

Một JWT là cấu trúc được dùng rộng rãi trong các kịch bản xác thực/ủy quyền cho API phi trạng thái. Bearer token dựa trên một chuẩn mở của ngành, giúp việc chia sẻ thông tin người dùng đã xác thực giữa server và client trở nên dễ dàng. Khi API được truy cập, một trạng thái tạm thời được tạo trong suốt chu kỳ request-response — sau khi response được trả về, ta không còn "ký ức" nào về request đó, nguồn gốc của nó, hay ai đã gửi nó.

Bearer token là một loại access token cho phép "người mang" (bearer) token — tức bên đang sở hữu token — truy cập tài nguyên được bảo vệ. Cái tên "bearer" xuất phát từ khái niệm: bất kỳ ai đang mang token đều được giả định là bên hợp lệ và được cấp quyền truy cập mà không cần chứng minh thêm. Token được phát hành sau một request xác thực thành công, thường chứa các trường:

| Trường | Ý nghĩa |
|---|---|
| Subject (sub) | Định danh duy nhất của người dùng, thường là user ID |
| Issuer (iss) | Tên dịch vụ đã phát hành token |
| Audience (aud) | Tên ứng dụng client dự kiến sẽ tiêu thụ token |
| Username | Tên đăng nhập duy nhất trong hệ thống |
| Email address | Email của người dùng |
| Role | Vai trò hệ thống quyết định người dùng được phép làm gì |
| Claims | Các thông tin khác nhau về người dùng, hỗ trợ ủy quyền hoặc hiển thị trên client |
| Expiry date (exp) | Ngày hết hạn — nên vừa phải, không quá ngắn cũng không tồn tại mãi mãi |

Luồng đăng nhập điển hình giữa client và API:

1. Người dùng đăng nhập qua ứng dụng client.
2. Client chuyển tiếp thông tin từ form đăng nhập tới endpoint API xác thực.
3. API xác minh tài khoản, tổng hợp thông tin liên quan, và trả về một chuỗi mã hóa chứa các bit thông tin quan trọng nhất về người dùng.
4. Client nhận response thành công cùng chuỗi token, và lưu lại để dùng sau.

Ta thường tránh đưa thông tin nhạy cảm (như mật khẩu) vào token. Bearer token được MÃ HÓA (encoded) chứ KHÔNG được ENCRYPT — đây là điểm cực kỳ quan trọng cần phân biệt. Chúng là các khối thông tin tự chứa, không thể đọc trực tiếp bằng mắt thường, nhưng việc giải mã (decode) một token rất dễ dàng — chuỗi được nén, thường ở dạng biểu diễn Base64, dùng cho cả vận chuyển lẫn lưu trữ. Chuỗi token KHÔNG được thiết kế để bảo mật, vì việc decode để xem nội dung bên trong là chuyện đơn giản — đó là lý do ta không bao giờ nhét dữ liệu nhạy cảm/mang tính buộc tội vào token.

Ví dụ giải mã payload của một token qua công cụ như jwt.io:

```json
{
  "iss": "http://server.example.com",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1311281970,
  "iat": 1311280970,
  "name": "Jane Doe",
  "given_name": "Jane",
  "family_name": "Doe",
  "gender": "female",
  "birthdate": "0000-10-31",
  "email": "janedoe@example.com",
  "picture": "http://example.com/janedoe/me.jpg"
}
```

Đọc từng key: **iss** (issuer) là ứng dụng đã phát hành token; **sub** (subject) là giá trị định danh chuẩn cho người dùng (username hoặc user ID), mà cả client lẫn API tiêu thụ JWT đều dựa vào để thực hiện các thao tác riêng của người dùng đó; **aud** (audience) là tên ứng dụng đã yêu cầu và nhận token — dùng để xác nhận đúng ứng dụng đang cầm đúng token; **nonce** là một giá trị duy nhất luôn thay đổi mỗi token để ngăn tấn công replay (đôi khi gọi là claim `jti`); **exp** là thời điểm hết hạn dưới dạng Unix epoch; **iat** (issued at) là thời điểm phát hành.

Token được trình trong header HTTP `Authorization` của các request API tiếp theo:

```
Authorization: Bearer <token>
```

### Điểm yếu cố hữu — "sở hữu là đủ"

Sự đơn giản của bearer token cũng chính là điểm yếu của nó — chúng hoạt động theo nguyên tắc "sở hữu" (possession): nếu token bị chặn, rò rỉ, hay đánh cắp, BẤT KỲ bên nào đang giữ token đều có thể mạo danh người dùng gốc. Bearer token do đó rất dễ bị tổn thương trước tấn công đánh cắp token, và cần các cơ chế bổ sung như HTTPS và thời gian sống ngắn để giảm thiểu rủi ro này.

### Access Token — khái niệm rộng hơn trong khuôn khổ OAuth 2.0

Access token là một khái niệm rộng hơn, định nghĩa trong khuôn khổ OAuth 2.0 — đại diện cho quyền ủy quyền được cấp cho một client (như web app hay API consumer) để truy cập tài nguyên cụ thể thay mặt cho người dùng. Access token được phát hành bởi một authorization server (như IdentityServer, Microsoft Azure Entra ID, hay Auth0) sau khi client xác thực thành công và nhận được sự đồng ý của người dùng.

Access token CÓ THỂ là bearer token, nhưng không phải bearer token nào cũng được dùng làm access token. Access token thường tuân theo các chuẩn nghiêm ngặt (RFC 6749, RFC 6750) và đi kèm metadata:

| Metadata | Ý nghĩa |
|---|---|
| Scope | Xác định quyền truy cập được phép (VD: read:appointments, write:invoices) |
| Audience | Bên nhận dự kiến của token (VD: appointments-api) |
| Issuer | Cơ quan đã phát hành token |
| Expiration time | Đảm bảo token có thời gian sống ngắn |
| Subject | Danh tính người dùng hoặc client |

Lý do access token thường được ưu tiên hơn bearer token thông thường là vì chúng chuẩn hóa và cấu trúc hóa mô hình kiểm soát truy cập — chúng gắn chặt với đặc tả OAuth 2.0 và tích hợp liền mạch với các IdP hỗ trợ scope, ủy quyền được ủy thác (delegated authorization), và resource server.

---

## Phần 6: Triển khai Bearer Token đơn giản trong ASP.NET Core (chỉ dùng Identity Core — chưa có IdentityServer)

ASP.NET Core cung cấp hỗ trợ xác thực/ủy quyền native qua thư viện Identity — tích hợp trực tiếp với Entity Framework Core để tạo các bảng quản lý người dùng chuẩn trong database. Thư viện này hỗ trợ sẵn:

- **User registration**: qua User Manager, đơn giản hóa việc tạo và quản lý người dùng.
- **Login, session, và cookie management**: qua Sign-in Manager.
- **Two-factor authentication**: hỗ trợ MFA native qua email/SMS.
- **Third-party identity provider**: hỗ trợ social login dễ dàng.

### Bảo vệ API Appointments bằng bearer token cơ bản

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

Package đầu hỗ trợ tích hợp trực tiếp giữa Entity Framework Core và Identity Core. Package thứ hai chứa các phương thức mở rộng cho phép triển khai quy tắc phát hành và xác thực token trong cấu hình API.

Sửa `AppointmentContext` để kế thừa `IdentityDbContext`:

```csharp
public class AppointmentContext : IdentityDbContext
{
    /* Shortened for brevity*/
}
```

```bash
dotnet ef migrations add AddedIdentityTables
dotnet ef database update
```

Trong `Program.cs`:

```csharp
builder.Services.AddAuthorization();
builder.Services
    .AddIdentityApiEndpoints<IdentityUser>()
    .AddEntityFrameworkStores<AppointmentContext>();
```

Đoạn code này không chỉ đăng ký các service ủy quyền, mà còn khởi tạo hỗ trợ cho các service dựa trên identity, đăng ký các route mặc định ánh xạ tới các thao tác liên quan tới identity, và chỉ định nội dung dữ liệu identity sẽ dùng để lưu trữ.

```csharp
app.MapIdentityApi<IdentityUser>();
app.UseAuthorization();
```

### Kiểm thử — Register và Login

```http
###
# Register
POST http://localhost:5107/register
Content-Type: application/json

{
    "username": "test@email.com",
    "email": "test@email.com",
    "password": "P@ssword123"
}
```

```http
###
# Login to the system
POST http://localhost:5107/login
Content-Type: application/json

{
    "email": "test@test.com",
    "password": "P@ssword123"
}
```

Response trả về:

```json
{
    "tokenType": "Bearer",
    "accessToken": "CfDJ8ImTP4HvH2ZPqpmH9gjeoDExk6-pNg…",
    "expiresIn": 3600,
    "refreshToken": "CfDJ8ImTP4HvH2ZPqpmH9gjeoDFOGsza2…"
}
```

Ta có một bearer token hết hạn sau một giờ, cùng một refresh token dùng với endpoint `/refresh` để lấy access token mới.

### Giới hạn của cách tiếp cận này

Đây là giải pháp "ra lò sẵn" (out-of-the-box) tiện lợi, nhưng không lý tưởng cho nhiều kịch bản thực tế:

1. **Xác thực JWT thường được xác minh CỤC BỘ**, không cần liên hệ với cơ quan phát hành — tuyệt vời cho hiệu năng, nhưng gây mong manh trong môi trường động. Nếu vai trò người dùng thay đổi hay quyền truy cập bị thu hồi, các service không có cách nào biết được điều đó cho tới khi token hết hạn. Việc thu hồi (revocation) về cơ bản KHÔNG được hỗ trợ trừ khi triển khai một revocation list trung tâm — điều này lại tái tạo một dạng shared state, phá vỡ chính thiết kế phi trạng thái vốn là điểm hấp dẫn ban đầu của JWT. Nói cách khác, xác thực bearer token per-service mang lại TỐC ĐỘ, đánh đổi bằng khả năng kiểm soát truy cập THỜI GIAN THỰC.

2. **Token vốn tự chứa, không có cơ chế delegation gốc**. Hãy hình dung một service Appointments cần liên hệ với service Billing thay mặt cho một bác sĩ vừa tạo cuộc hẹn khám mới. Với bearer token thuần túy, service phải hoặc chuyển tiếp nguyên token gốc của người dùng (có thể phơi ra quyền rộng hơn cần thiết), hoặc hành động bằng thẩm quyền của chính nó (phá vỡ nguyên tắc "quyền tối thiểu" — principle of least privilege). Không có cơ chế gốc nào để khẳng định "service này đang hành động thay mặt cho người dùng này, cho đúng mục đích cụ thể này". Thiếu mô hình bảo mật nhận biết delegation, tương tác service-to-service trở nên mờ đục và có nguy cơ dư thừa quyền, khiến việc thực thi ủy quyền chi tiết theo ngữ cảnh trở nên khó khăn.

3. **Gánh nặng dồn lên từng service riêng lẻ**. Khi mỗi microservice tự chịu trách nhiệm xử lý logic xác thực của mình, một gánh nặng đáng kể đè lên chính service đó — mỗi service phải triển khai parse token, xác thực, kiểm tra hết hạn, xác minh chữ ký, và nếu áp dụng, thực thi scope. Logic này phải được lặp lại NHẤT QUÁN trên MỌI service trong hệ thống. Xác thực bearer token per-service cũng không cung cấp quản lý identity tập trung, audit, hay sự đồng ý (consent) của người dùng — trong các ngành có quy định như y tế, người dùng phải đồng ý rõ ràng cho việc truy cập/chia sẻ dữ liệu; các service dùng xác thực token độc lập không có cách nào theo dõi hay thực thi sự đồng ý đó, cũng không có góc nhìn thống nhất về danh tính người dùng xuyên suốt các service.

**Bài học cốt lõi: Bearer token xác thực từng service riêng lẻ giống như phát cho mỗi tòa nhà một cuốn sổ tay riêng để tự kiểm tra giấy tờ khách — hoạt động được, nhưng không ai trong quần thể biết được khi nào một giấy tờ đã bị thu hồi, và mỗi tòa phải tự viết lại logic kiểm tra từ đầu.**

---

## Phần 7: OAuth 2.0 và OpenID Connect — chuẩn hóa danh tính cho hệ thống phân tán

Nhận ra giới hạn của xác thực bearer token per-service, ta chuyển sang một cách tiếp cận mạnh mẽ và được chấp nhận rộng rãi trong ngành: xác thực và ủy quyền tập trung dùng OAuth 2.0 và OpenID Connect (OIDC). Hai giao thức này tạo thành nền tảng của quản lý danh tính và kiểm soát truy cập hiện đại trong hệ thống phân tán.

### Lịch sử ngắn gọn

Phiên bản đầu tiên của OAuth ra mắt năm 2007 bởi một nhóm dev từ Twitter, Ma.gnolia, và các nền tảng khác — OAuth 1.0 được thiết kế để giải quyết bài toán "ủy quyền được ủy thác" (delegated authorization), cho phép ứng dụng bên thứ ba hành động thay mặt người dùng mà không cần biết username/password của họ. Nó dùng ký mã hóa (HMAC-SHA1) để bảo mật request API.

OAuth 2.0 ra mắt năm 2012 bởi IETF OAuth Working Group (RFC 6749), giới thiệu một mô hình thân thiện và dễ mở rộng hơn với dev. Nó tách biệt hành động ủy quyền khỏi xác thực — cho phép client lấy được một access token, dùng để gọi API được bảo vệ. Token này đại diện cho một tập quyền (scope) được cấp bởi người dùng hoặc resource owner.

Giao thức cũng định nghĩa nhiều "grant type" (luồng cấp quyền) khác nhau, quyết định cách client lấy được token:

| Grant Type | Dùng cho |
|---|---|
| Authorization Code | Web app; đổi một mã tạm thời lấy token |
| Implicit | App chạy trong trình duyệt (giờ không khuyến khích vì lo ngại bảo mật) |
| Resource Owner Password Credentials | Truyền trực tiếp username/password (không khuyến khích) |
| Client Credentials | Xác thực máy-với-máy (machine-to-machine) |
| Refresh Token | Lấy access token mới mà không cần người dùng xác thực lại |

OAuth 2.0 mang lại một giải pháp mạnh mẽ cho kiểm soát truy cập và delegation, nhưng CHỦ ĐÍCH không xử lý vấn đề xác thực (authentication) — khoảng trống này dẫn tới việc OAuth bị lạm dụng rộng rãi làm giao thức đăng nhập, dù nó chưa bao giờ được thiết kế cho mục đích đó.

Để lấp khoảng trống giữa xác thực và ủy quyền, OpenID Foundation giới thiệu OIDC năm 2014. OIDC là một lớp danh tính (identity layer) xây trên nền OAuth 2.0, cung cấp phương thức chuẩn để đăng nhập, lấy danh tính người dùng, và thông tin hồ sơ. OIDC tận dụng các luồng ủy quyền của OAuth để xác thực người dùng, rồi phát hành một ID token — một JWT chứa các claim về người dùng đã xác thực. Nó cũng hỗ trợ discovery (cho phép client lấy metadata cấu hình về IdP) và đăng ký client động.

**Bài học cốt lõi: OAuth 2.0 trả lời câu hỏi "client này được PHÉP làm gì?" — OIDC trả lời câu hỏi "người dùng này LÀ AI?". Hai giao thức bổ trợ nhau chứ không thay thế nhau.**

---

## Phần 8: Identity Provider — cơ quan trung tâm phát hành "hộ chiếu"

Một Identity Provider (IdP) là một dịch vụ chuyên trách xác thực người dùng và phát hành danh tính số dưới dạng token. Đây là cơ quan đáng tin cậy chịu trách nhiệm:

- Xác minh danh tính người dùng/client
- Quản lý thông tin đăng nhập, hồ sơ, và vai trò
- Phát hành, xác thực, và thu hồi token xác thực
- Hỗ trợ single sign-on (SSO) xuyên suốt nhiều hệ thống/ứng dụng

Giá trị cốt lõi của IdP nằm ở quản lý danh tính TẬP TRUNG. Bằng cách chuyển việc xác thực ra một endpoint duy nhất, đã được kiểm chứng an toàn (hardened), các ứng dụng không còn cần tự quản lý thông tin đăng nhập, chính sách mật khẩu, hay cơ chế khóa tài khoản — giảm bề mặt tấn công và cho phép đội ngũ đầu tư vào một dịch vụ identity an toàn duy nhất, thay vì lặp lại công sức ở nhiều service.

Bằng cách trừu tượng hóa xác thực, ứng dụng trở nên "vô cảm" (agnostic) với cơ chế xác thực — bạn có thể thay thế hoặc nâng cấp hệ thống identity mà không cần viết lại các service nội bộ. Sự decoupling này nâng cao khả năng bảo trì, đặc biệt khi hệ thống ngày càng phức tạp.

IdP hiện đại tuân theo các giao thức mở như OAuth 2.0, OIDC, và SAML (Security Assertion Markup Language) — cho phép tương thích rộng với công cụ bên thứ ba, dịch vụ cloud, và nền tảng mobile. IdP hỗ trợ SSO xuyên suốt nhiều ứng dụng — người dùng xác thực MỘT LẦN và nhận token dùng được cho nhiều thành phần khác nhau của hệ thống.

### Hai nhóm chính của IdP

| Loại | Đặc điểm |
|---|---|
| Enterprise IdP | Quản lý danh tính và truy cập trong nội bộ tổ chức, kiểm soát tập trung xác thực/ủy quyền/vòng đời người dùng — dùng trong môi trường doanh nghiệp, tích hợp hệ thống nội bộ, đáp ứng yêu cầu bảo mật cấp doanh nghiệp |
| Social IdP | Quản lý danh tính gắn với tài khoản cá nhân trên nền tảng như Google, Facebook — cho phép đăng nhập nhanh, tiện lợi cho ứng dụng hướng người dùng cuối mà không cần tạo tài khoản mới |

Các lựa chọn IdP phổ biến gồm: cloud-based (Okta, Microsoft Entra ID — trước đây là Azure Active Directory), on-premises (Duende IdentityServer), và social provider (Google, Facebook).

Ta sẽ dùng **IdentityServer** làm ví dụ triển khai — một framework mã nguồn mở xây trên ASP.NET Core, mở rộng khả năng của Identity Core, cho phép dev triển khai chuẩn OIDC và OAuth 2.0 trong ứng dụng web. Nó tuân thủ chuẩn ngành, hỗ trợ sẵn xác thực dựa trên token, SSO, và kiểm soát truy cập API — dù là sản phẩm thương mại, có một phiên bản community cho tổ chức nhỏ hoặc dự án cá nhân.

### Mô hình triển khai khuyến nghị

1. Tạo một microservice mới dành riêng cho xác thực
2. Tạo một database mới chỉ chứa bảng liên quan tới xác thực (tùy chọn)
3. Cấu hình scope sẽ được đưa vào thông tin token
4. Cấu hình các service để biết scope nào được phép truy cập chúng

IdentityServer đứng giữa client và service, quản lý luồng xác thực và trao đổi token: client gửi username/password qua OAuth tới IdentityServer, nhận về token; sau đó client gửi API request kèm token tới API Service; API Service gửi thông tin về service và token tới IdentityServer để xác minh, và nhận về validation result.

---

## Phần 9: Triển khai Duende IdentityServer — dựng service xác thực trung tâm HealthCare.Auth

Duende cung cấp các project template quick-start giúp bootstrap nhanh chức năng IdentityServer trong một project ASP.NET Core. Các bước tổng quát để thiết lập một IdentityServer service:

1. Thêm hỗ trợ Duende IdentityServer vào một project ASP.NET Core chuẩn
2. Thêm hỗ trợ lưu trữ dữ liệu, ưu tiên dùng cấu hình Entity Framework
3. Thêm hỗ trợ ASP.NET Identity Core
4. Cấu hình phát hành token cho ứng dụng client
5. Bảo vệ ứng dụng client bằng IdentityServer

### Cài đặt template

```bash
dotnet new install Duende.IdentityServer.Templates
```

Lệnh này đăng ký thêm các CLI command mới:

- `isapi`: IdentityServer với in-memory storage (API-first)
- `isui`: IdentityServer với UI và cấu hình in-memory
- `isef`: IdentityServer với UI và EF Core store

### Tạo project HealthCare.Auth

```bash
dotnet new isef -n IdentityServerEf -o HealthCare.Auth
```

Khi được hỏi có seed project không, chọn 'Y'. Cấu trúc project được sinh ra gồm:

- **wwwroot**: tài nguyên tĩnh (JS, CSS)
- **Migrations**: các migration có sẵn để tạo bảng hỗ trợ trong data store
- **Pages**: các Razor Page mặc định hỗ trợ nhu cầu UI của thao tác xác thực (login, register, grants, quản lý dữ liệu người dùng)
- **appsettings.json**: cấu hình logging và connection string database
- **buildschema.bat**: chứa lệnh Entity Framework để chạy các migration script trong thư mục Migrations
- **Config.cs**: class static đóng vai trò cơ quan cấu hình — khai báo `IdentityResources`, `Scopes`, `Clients`
- **HostingExtension.cs**: chứa các phương thức mở rộng đăng ký service và middleware, được gọi trong Program.cs lúc khởi động
- **Program.cs**: file thực thi chính
- **SeedData.cs**: chứa phương thức đảm bảo migration và seed data được thực hiện lúc khởi động

### Đăng ký IdentityServer trong HostingExtension.cs

IdentityServer dùng HAI database context: một cho configuration store, một cho operational store.

```csharp
var isBuilder = builder.Services
    .AddIdentityServer(options =>
    {
        options.Events.RaiseErrorEvents = true;
        options.Events.RaiseInformationEvents = true;
        options.Events.RaiseFailureEvents = true;
        options.Events.RaiseSuccessEvents = true;
        options.EmitStaticAudienceClaim = true;
    })
    .AddTestUsers(TestUsers.Users)
    .AddConfigurationStore(options =>
    {
        options.ConfigureDbContext = b =>
            b.UseSqlite(connectionString, dbOpts => dbOpts
                .MigrationsAssembly(typeof(Program).Assembly.FullName));
    })
    .AddOperationalStore(options =>
    {
        options.ConfigureDbContext = b =>
            b.UseSqlite(connectionString, dbOpts =>
                dbOpts.MigrationsAssembly(typeof(Program).Assembly.FullName));
        options.EnableTokenCleanup = true;
        options.RemoveConsumedTokens = true;
    });
```

Ta thêm `TestUsers` vào cấu hình, rồi thêm `ConfigurationStoreDbContext` và `OperationalStoreDbContext`. Các cài đặt khác quản lý cách xử lý cảnh báo và token — mặc định thường ổn, nhưng có thể chỉnh theo nhu cầu cụ thể.

Nếu dùng công nghệ database khác SQLite:

```bash
dotnet ef database update --context PersistedGrantDbContext
dotnet ef database update --context ConfigurationDbContext
```

### Cấu hình IdentityResources — dữ liệu nào sẽ nằm trong token

```csharp
public static IEnumerable<IdentityResource> IdentityResources =>
    new IdentityResource[]
    {
        new IdentityResources.OpenId(),
        new IdentityResources.Profile(),
        new IdentityResource("roles", "User role(s)",
            new List<string> { "role" })
    };
```

Ta đã thêm danh sách roles vào tập resource. Cần đảm bảo người dùng có đủ dữ liệu kỳ vọng cũng như danh sách claim mà họ được kỳ vọng sở hữu — claim chính là thông tin mà ứng dụng client sẽ có được thông qua token, vì đó là cách duy nhất client theo dõi được người dùng nào đang online và họ được phép làm gì.

### Cấu hình ApiScopes — client hay user?

```csharp
public static IEnumerable<ApiScope> ApiScopes =>
    new ApiScope[]
    {
        new ApiScope("healthcareApiUser", "HealthCare API User"),
        new ApiScope("healthcareApiClient", "HealthCare API Client"),
    };
```

Ta hỗ trợ hai loại scope xác thực cho hai kịch bản khác nhau: **client authentication** đại diện cho nỗ lực truy cập tài nguyên KHÔNG có giám sát, thường bởi một chương trình/API khác; **user authentication** là quá trình người dùng xác minh danh tính bằng thông tin đăng nhập.

### Cấu hình Clients — machine-to-machine và interactive

```csharp
public static IEnumerable<Client> Clients =>
    new Client[]
    {
        // m2m client credentials flow client
        new Client
        {
            ClientId = "m2m.client",
            ClientName = "Client Credentials Client",
            AllowedGrantTypes = GrantTypes.ClientCredentials,
            ClientSecrets = { new Secret("511536EF-F270-4058-80CA-1C89C192F69A".Sha256()) },
            AllowedScopes = { "healthcareApiClient" }
        },
        // interactive client using code flow + pkce
        new Client
        {
            ClientId = "interactive",
            ClientSecrets = { new Secret("49C1A7E1-0C79-4A89-A3D6-A37998FB86B0".Sha256()) },
            AllowedGrantTypes = GrantTypes.Code,
            RedirectUris = { "https://localhost:5001/signin-oidc" },
            FrontChannelLogoutUri = "https://localhost:5001/signout-oidc",
            PostLogoutRedirectUris = { "https://localhost:5001/signout-callback-oidc" },
            AllowOfflineAccess = true,
            AllowedScopes = { "openid", "profile", "healthcareApiUser", "roles" }
        },
    };
```

Ta đã định nghĩa `ClientId` và `ClientSecret` cho từng client. Bằng cách định nghĩa nhiều client, ta có thể hỗ trợ các ứng dụng dự kiến ở mức độ chi tiết hơn, và định nghĩa `AllowedScopes`/`AllowedGrantTypes` riêng biệt. Ở đây, `m2m.client` đại diện cho một API — có thể là một microservice cần xác thực với dịch vụ xác thực, thường xảy ra mà KHÔNG có tương tác người dùng. `interactive` đại diện cho một ứng dụng hướng người dùng, đòi hỏi cấu hình URL sign-in/sign-out để redirect người dùng trong luồng xác thực hoặc đăng xuất. Việc thêm giá trị `"roles"` vào `AllowedScopes` đảm bảo thông tin vai trò được đưa vào token khi người dùng xác thực.

### Chạy thử

```bash
dotnet run
```

Truy cập `https://localhost:5001` — trang chủ hiển thị IdentityServer đang hoạt động, kèm liên kết tới discovery document, claim của session hiện tại, các grant đã lưu, session phía server, các yêu cầu login CIBA đang chờ, và trang admin đơn giản.

Trong file `TestUsers.cs` (thư mục Pages), dùng `alice` làm cả username và password để test nhanh.

### Discovery Document — bản đồ toàn bộ endpoint của IdP

Hầu hết nhà cung cấp OAuth 2.0/OIDC có khái niệm discovery document — vạch ra các route tích hợp sẵn trong API, claim được hỗ trợ, loại token, cùng các thông tin quan trọng khác:

```json
{
    "issuer": "https://localhost:5001",
    "jwks_uri": "https://localhost:5001/.well-known/openid-configuration/jwks",
    "authorization_endpoint": "https://localhost:5001/connect/authorize",
    "token_endpoint": "https://localhost:5001/connect/token",
    "userinfo_endpoint": "https://localhost:5001/connect/userinfo",
    "end_session_endpoint": "https://localhost:5001/connect/endsession",
    "check_session_iframe": "https://localhost:5001/connect/checksession",
    "revocation_endpoint": "https://localhost:5001/connect/revocation",
    "introspection_endpoint": "https://localhost:5001/connect/introspect",
    "device_authorization_endpoint": "https://localhost:5001/connect/deviceauthorization",
    "backchannel_authentication_endpoint": "https://localhost:5001/connect/ciba",
    ...
    "scopes_supported": [
        "openid",
        "profile",
        "roles",
        "healthcareApiUser",
        "healthcareApiClient",
        "offline_access"
    ],
    ...
}
```

### Lấy token bằng Client Credentials flow qua Postman

Dùng Postman, chọn Authorization type OAuth 2.0, Grant Type Client Credentials, điền Access Token URL (`https://localhost:5001/connect/token`), Client ID (`m2m.client`), Client Secret tương ứng, nhấn "Get New Access Token" — lưu ý cần tắt SSL verification trong môi trường dev.

Payload của token trả về:

```json
{
    "iss": "https://localhost:5001",
    "nbf": 1668216814,
    "iat": 1668216814,
    "exp": 1668220414,
    "aud": "https://localhost:5001/resources",
    "scope": [
        "healthcareApiClient"
    ],
    "client_id": "m2m.client",
    "jti": "FF86E33B7756615B609A61109BE0D4C8"
}
```

Vì IdentityServer tuân theo chuẩn OAuth và OIDC, ta không cần tự thêm các claim cơ bản như `sub`, `exp`, `jti`, `iss` — chúng tự động được đưa vào.

---

## Phần 10: Bảo vệ API bằng IdentityServer — hai lựa chọn kiến trúc

Ta đối mặt với thách thức riêng: triển khai giải pháp bảo mật hiệu quả nhất xuyên suốt ứng dụng microservices. Có nhiều service cần bảo vệ, và tùy vào pattern kiến trúc đã áp dụng, có thể còn có một gateway đang định tuyến traffic:

| Lựa chọn | Ưu điểm | Nhược điểm |
|---|---|---|
| Bảo vệ TỪNG service riêng | Kiểm soát chi tiết theo từng service | Mỗi service có yêu cầu khác nhau, có thể cần coi là client riêng biệt cho mỗi request — dễ thành cơn ác mộng bảo trì khi quản lý scope/client cho từng service; người dùng có thể phải xác thực nhiều lần khi truy cập các tính năng dựa trên service khác nhau |
| Bảo vệ API GATEWAY | Gateway điều phối luồng xác thực cho client rồi quản lý việc chia sẻ token giữa các lời gọi service — cách tiếp cận hợp lý nhất, đặc biệt hiệu quả khi kết hợp BFF Pattern | Cần thiết kế đúng để tránh gateway trở thành điểm nghẽn bảo mật duy nhất |

**Bài học cốt lõi: Bảo vệ tại gateway thay vì từng service riêng lẻ chính là áp dụng lại đúng tinh thần tập trung hóa cross-cutting concern đã học ở Chapter 11 — chỉ khác là lần này cross-cutting concern đó là bảo mật.**

### Bảo vệ trực tiếp API Appointments

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://localhost:5001/";
        options.Audience = "appointments_api";
        options.TokenValidationParameters.ValidateAudience = true;
        options.TokenValidationParameters.ValidTypes = new[] { "at+jwt" };
    });
```

Cấu hình này chỉ dẫn service tham chiếu tới URL trong tùy chọn `Authority` để lấy hướng dẫn xác thực.

Thực thi chính sách ủy quyền toàn cục để đảm bảo không endpoint nào có thể truy cập nếu thiếu bearer token hợp lệ do IdentityServer phát hành:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAuth", policy =>
    {
        policy.RequireAuthenticatedUser();
    });
});
```

```csharp
app.MapControllers().RequireAuthorization("RequireAuth");
```

Giờ đây, mọi nỗ lực tương tác với endpoint API Appointments mà thiếu bearer token hợp lệ trong header `Authorization` sẽ trả về 401 Unauthorized. Với token hợp lệ lấy từ bước xác thực client credentials trước đó, ta gọi được endpoint bình thường.

---

## Phần 11: Bảo vệ Ocelot API Gateway bằng IdentityServer — nối lại toàn bộ mạch từ Chapter 11

Đây chính là điểm hội tụ trực tiếp của Chapter 11 và Chapter 13. Nhớ lại khi ta thiết lập gateway ở Chapter 11, ta đã thêm một mục `AuthenticationOptions` vào `ocelot.json` — mục này ĐÃ SẴN SÀNG hỗ trợ bearer authentication, chỉ chưa có một IdentityServer thật đứng sau nó để phát hành/xác minh token:

```json
{
    "DownstreamPathTemplate": "/api/appointments",
    ...
    "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": [ "appointments.read", "appointments.write" ]
    },
    ...
}
```

Cấu hình này đòi hỏi hai scope bổ sung để truy cập tài nguyên appointments. Cần chỉnh `Config.cs` trong project `HealthCare.Auth` để thêm các scope này:

```csharp
public static class Config
{
    public static IEnumerable<IdentityResource> IdentityResources => …

    public static IEnumerable<ApiScope> ApiScopes =>
        new ApiScope[]
        {
            …
            // Add these new scopes:
            new ApiScope("appointments.read", "Read access to Appointments API"),
            new ApiScope("appointments.write", "Write access to Appointments API")
        };

    public static IEnumerable<Client> Clients =>
        …
        // Update to allow the new write scope as well:
        AllowedScopes = { "healthcareApiClient", "appointments.write" }
        },
        // interactive client using code flow + pkce
        new Client
        {
            …
            AllowedScopes =
            {
                "openid",
                "profile",
                "healthcareApiUser",
                "roles",
                "appointments.read", // Add
                "appointments.write" // Add
            }
        },
        };
}
```

Những thay đổi này cho phép **client machine-to-machine** (client credentials — chính gateway sẽ dùng khi cần gọi API Appointments) thêm claim `appointments.write` vào `AllowedScopes`. Nếu không, khi gateway đi lấy token, scope đó sẽ bị từ chối, và các lời gọi downstream API sẽ thất bại với lỗi "insufficient scope".

**Client interactive** đại diện cho frontend chạy trên trình duyệt hoặc MVC — nó dùng luồng authorization code kết hợp PKCE (Proof Key for Code Exchange), kết thúc bằng một access token có thể trình cho bất kỳ downstream API nào. Bằng cách thêm cả `appointments.read` lẫn `appointments.write` vào đây, người dùng đã đăng nhập có thể lấy token mang các claim đó — nhờ vậy UI có thể tiêu thụ cả endpoint chỉ đọc (liệt kê lịch hẹn) lẫn endpoint ghi (tạo/sửa lịch hẹn) mà không cần đăng ký client riêng biệt.

### Cấu hình gateway để xác thực token

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://localhost:5001/";
        options.Audience = "appointments_api";
        options.TokenValidationParameters.ValidateAudience = true;
        options.TokenValidationParameters.ValidTypes = new[] { "at+jwt" };
    });
```

Ta vừa hoàn tất việc bảo vệ gateway bằng IdentityServer. Đây, một lần nữa, có thể là giải pháp bảo mật tốt hơn cho toàn bộ tập microservice sẽ được truy cập qua gateway, và nó giúp tập trung hóa quyền truy cập vào các service của ta.

Với thay đổi đơn giản này, ta không còn cần API Appointments phải bận tâm về việc chứa bảng xác thực trong database của nó, hay logic biên dịch JWT bearer phức tạp. Ta chỉ đơn giản trỏ service tới `Authority` (dịch vụ xác thực) và bao gồm giá trị `Audience` để nó tự nhận diện với dịch vụ xác thực.

Bằng cách đăng ký JWT bearer authentication trong `Program.cs`, ta cấu hình gateway để xác thực token đến do IdentityServer phát hành. Sự tích hợp này đảm bảo chỉ những request có access token hợp lệ do IdentityServer trung tâm phát hành, chứa đúng audience và loại, mới được phép truy cập endpoint được bảo vệ.

Với cấu hình này, người dùng phải cung cấp một token (như đã lấy ở phần trước) để thực hiện bất kỳ lời gọi nào tới API. Nếu token thiếu, trả về 401 Unauthorized. Nếu token có mặt nhưng thiếu scope/claim thiết yếu, trả về 403 Forbidden.

**Bài học cốt lõi: Bảo vệ tại gateway biến bài toán "N service cần N bộ logic xác thực" thành "1 gateway + 1 IdentityServer trung tâm" — mỗi microservice phía sau chỉ cần tin tưởng gateway đã xác thực đúng, thay vì tự mình gánh vác toàn bộ độ phức tạp của OAuth/OIDC.**

---

## Tổng kết chương

Chương này trình bày một cuộc khảo sát toàn diện về bảo mật trong bối cảnh kiến trúc microservices, với trọng tâm đặc biệt vào các mô hình xác thực và ủy quyền hiện đại triển khai trong ứng dụng .NET. Ta bắt đầu bằng việc xác lập tầm quan trọng của bảo mật như một yếu tố nền tảng của phát triển ứng dụng, đặc biệt trong môi trường phân tán — khi microservices nhân rộng xuyên suốt mạng lưới và đội ngũ, bề mặt tấn công tăng lên, đòi hỏi các cơ chế bảo mật mạnh mẽ, nhất quán, và có khả năng mở rộng.

Ta so sánh trách nhiệm bảo mật giữa monolith và microservices: monolith có thể tập trung hóa xác thực/ủy quyền nhờ đơn vị triển khai thống nhất, trong khi microservices đòi hỏi mỗi service tự xử lý các mối quan tâm này một cách độc lập — sự phi tập trung hóa này tạo ra thách thức trong quản lý danh tính, thực thi chính sách nhất quán, và bảo mật giao tiếp liên service.

Ta khám phá các khái niệm bảo mật cốt lõi — xác thực, ủy quyền, tính bảo mật và toàn vẹn dữ liệu — và giới thiệu bearer token cùng JWT như giải pháp cho xác thực phi trạng thái, đặc biệt trong bối cảnh bảo mật API. Nhận ra giới hạn của bearer token đơn giản khi dùng độc lập ở mỗi microservice — đặc biệt là việc không hỗ trợ thu hồi thời gian thực và thiếu cơ chế delegation gốc — chương tiến tới một mô hình tinh vi hơn: xác thực tập trung qua OAuth 2.0 và OIDC. Hai giao thức này giới thiệu khái niệm access token, luồng ủy quyền, scope, và claim, đồng thời xác lập vai trò của một Identity Provider — cơ quan trung tâm xác thực người dùng và phát hành token, cho phép quản lý danh tính nhất quán, SSO, và ủy quyền được ủy thác xuyên suốt nhiều service và client.

Ta đã đi qua triển khai thực tế dùng Duende IdentityServer như một IdP tuân thủ chuẩn trong hệ sinh thái .NET — từ việc thiết lập project HealthCare.Auth với ASP.NET Core và Entity Framework store, cấu hình client/scope/test user, cho tới việc chứng minh IdentityServer đủ linh hoạt để hỗ trợ cả client machine-to-machine lẫn client tương tác trên trình duyệt (dùng luồng PKCE và authorization code).

Ta cấu hình Ocelot API Gateway (nối trực tiếp từ Chapter 11) để bảo vệ bằng bearer token, cho phép nó đóng vai trò trung gian an toàn — xác thực token và thực thi ủy quyền trước khi chuyển tiếp request xuống downstream. Pattern tập trung hóa này đơn giản hóa việc thực thi chính sách truy cập và giảm thiểu trùng lặp logic bảo mật xuyên suốt hệ thống.

**Bài học cốt lõi: Bảo mật microservices không nằm ở việc mỗi service tự "khóa cửa" riêng — nó nằm ở việc xây dựng một cơ quan trung tâm đáng tin cậy (IdentityServer) mà toàn bộ hệ thống, từ gateway tới từng service, đều tin tưởng và biết cách xác minh cùng một "hộ chiếu" chung, giúp gánh nặng bảo mật không bị nhân bản N lần theo số lượng service.**

Chương tiếp theo sẽ chuyển hướng sang một khía cạnh hạ tầng khác: sử dụng container và orchestration để phát triển một ứng dụng microservices cloud-native.
