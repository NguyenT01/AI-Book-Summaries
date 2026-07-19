# Chapter 13 — Securing Microservices: Ghi Chép Ôn Tập

> Dùng file này để: (1) ôn lại khái niệm, (2) checklist áp dụng thực tế, (3) chuẩn bị phỏng vấn lên level.

---

## 1. Bảng khái niệm cốt lõi

| Khái niệm | Định nghĩa ngắn gọn |
|---|---|
| Authentication | Xác thực — quá trình xác lập danh tính (ai đang truy cập) |
| Authorization | Ủy quyền — quyết định danh tính đó được phép làm gì |
| RBAC (Role-Based Access Control) | Kiểm soát truy cập dựa trên vai trò (admin, doctor, nurse...) |
| Bearer Token | Token cho phép "người mang" truy cập tài nguyên bảo vệ, dựa trên nguyên tắc sở hữu |
| JWT (JSON Web Token) | Token tự chứa, mã hóa (không mã hóa nội dung — chỉ encode), mang các claim |
| Claim | Thông tin về người dùng chứa trong token (sub, role, email...) |
| Access Token | Khái niệm rộng hơn bearer token, định nghĩa trong OAuth 2.0, đại diện quyền ủy quyền cụ thể |
| Scope | Phạm vi quyền truy cập được cấp trong token (VD: appointments.read) |
| OAuth 2.0 | Giao thức ủy quyền (authorization), tách biệt khỏi xác thực, định nghĩa các grant type |
| OpenID Connect (OIDC) | Lớp danh tính xây trên OAuth 2.0, bổ sung xác thực (authentication) và ID token |
| Identity Provider (IdP) | Dịch vụ chuyên trách xác thực người dùng, phát hành/xác minh/thu hồi token |
| Grant Type | Cách client lấy được token (Authorization Code, Client Credentials, Refresh Token...) |
| PKCE (Proof Key for Code Exchange) | Cơ chế bảo mật bổ sung cho luồng Authorization Code, dùng cho client tương tác |
| Discovery Document | Tài liệu metadata của IdP, liệt kê endpoint, scope, claim được hỗ trợ |
| SSO (Single Sign-On) | Đăng nhập một lần, dùng được xuyên suốt nhiều ứng dụng/service |
| Delegation | Cơ chế một service hành động thay mặt người dùng/service khác với quyền hạn cụ thể |
| Revocation | Thu hồi token trước khi hết hạn tự nhiên |

---

## 2. So sánh các mô hình xác thực

| Mô hình | Cách hoạt động | Ưu điểm | Nhược điểm |
|---|---|---|---|
| API Key | Key tĩnh gửi kèm mỗi request | Đơn giản, dễ triển khai | Thiếu ngữ cảnh người dùng, khó rotate, dễ rò rỉ |
| Basic Authentication | Username/password mã hóa Base64 mỗi request | Đơn giản cho tool nội bộ | Credential sống lâu, không kiểm soát chi tiết, không audit được |
| OAuth 2.0 + Bearer Token | Token JWT ký, cấp bởi IdP trung tâm | Chuẩn hóa, scope/claim chi tiết, stateless, scale tốt | Phức tạp hơn để triển khai đúng, cần xử lý refresh/revocation |

## 3. So sánh Bearer Token per-service vs OAuth/OIDC tập trung

| Tiêu chí | Bearer Token per-service | OAuth 2.0/OIDC tập trung (IdP) |
|---|---|---|
| Nơi xác thực | Mỗi service tự parse/validate token | Một IdentityServer trung tâm phát hành/xác minh |
| Revocation | Không hỗ trợ tốt (JWT tự chứa, không kiểm tra lại) | Có thể qua introspection endpoint/revocation list |
| Delegation | Không có cơ chế gốc | OAuth hỗ trợ qua scope, client credentials |
| Trùng lặp logic | Mỗi service tự viết lại parse/validate | Tập trung một nơi, service chỉ cần trỏ Authority |
| Audit/Consent | Không có góc nhìn thống nhất | Tập trung, dễ audit và quản lý consent |
| SSO | Không hỗ trợ | Có hỗ trợ native |

## 4. So sánh OAuth 2.0 vs OpenID Connect

| Tiêu chí | OAuth 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| Mục đích chính | Ủy quyền (authorization) — client được phép làm gì | Xác thực (authentication) — người dùng là ai |
| Token phát hành | Access Token | Access Token + ID Token (JWT chứa claim người dùng) |
| Ra đời | 2012 (RFC 6749) | 2014, xây trên nền OAuth 2.0 |
| Câu hỏi trả lời | "Client này có quyền gì?" | "Ai đang đăng nhập?" |
| Discovery | Không có chuẩn discovery | Có chuẩn discovery document |

## 5. Các Grant Type của OAuth 2.0

| Grant Type | Dùng cho | Khuyến nghị |
|---|---|---|
| Authorization Code | Web app, đổi code lấy token | Khuyến khích (đặc biệt kèm PKCE) |
| Implicit | App chạy trong trình duyệt | Không khuyến khích (lo ngại bảo mật) |
| Resource Owner Password Credentials | Truyền trực tiếp username/password | Không khuyến khích |
| Client Credentials | Machine-to-machine | Khuyến khích cho service-to-service |
| Refresh Token | Lấy access token mới không cần đăng nhập lại | Khuyến khích, kết hợp Authorization Code |

## 6. Hai lựa chọn nơi bảo vệ API trong hệ microservices

| Tiêu chí | Bảo vệ từng service riêng | Bảo vệ tại API Gateway |
|---|---|---|
| Độ phức tạp quản lý | Cao — mỗi service một bộ scope/client | Thấp hơn — tập trung một nơi |
| Trải nghiệm người dùng | Có thể phải xác thực nhiều lần | Xác thực một lần, gateway điều phối |
| Kết hợp BFF | Khó | Rất phù hợp |
| Rủi ro | Trùng lặp logic, dễ cấu hình sai | Gateway trở thành điểm cần bảo vệ kỹ nhất |

---

## Checklist kỹ thuật khi triển khai

### Nguyên tắc bảo mật ứng dụng chung
- [ ] Đã xác định rõ 3 trụ cột: Authentication, Authorization, Data confidentiality/integrity
- [ ] Mật khẩu được hash (không lưu plaintext), dữ liệu nhạy cảm mã hóa khi lưu trữ
- [ ] HTTPS/TLS bắt buộc cho mọi endpoint, redirect từ HTTP sang HTTPS
- [ ] Đã kiểm tra yêu cầu tuân thủ ngành (HIPAA/PCI DSS/GDPR) nếu áp dụng
- [ ] Không hardcode secret/connection string — dùng secrets manager/biến môi trường

### Khi thiết kế JWT/Bearer Token
- [ ] Token có thời gian sống (exp) hợp lý — không quá dài, không quá ngắn
- [ ] Không đưa dữ liệu nhạy cảm (mật khẩu, số thẻ...) vào claim của token
- [ ] Đã có cơ chế refresh token cho luồng cần re-authenticate mượt
- [ ] Không log token đầy đủ trong log ứng dụng (tránh rò rỉ)
- [ ] Nếu chỉ dùng bearer token per-service, đã cân nhắc giới hạn (không hỗ trợ revocation thời gian thực)

### Khi dựng Identity Server (Duende IdentityServer)
- [ ] Đã tạo project riêng (`dotnet new isef`) cho service xác thực, tách biệt khỏi các microservice nghiệp vụ
- [ ] Đã cấu hình đủ 2 DbContext: ConfigurationStore và OperationalStore
- [ ] Đã định nghĩa IdentityResources (openid, profile, roles...) theo đúng claim cần có trong token
- [ ] Đã định nghĩa ApiScopes riêng cho từng nhóm quyền (VD: appointments.read, appointments.write)
- [ ] Đã định nghĩa Client riêng cho machine-to-machine (ClientCredentials) và client tương tác (Code + PKCE)
- [ ] Client tương tác có đủ RedirectUris, FrontChannelLogoutUri, PostLogoutRedirectUris
- [ ] Đã test lấy token qua Postman/API tool với đúng Client ID/Secret/Scope

### Khi bảo vệ API bằng IdentityServer
- [ ] Đã thêm package `Microsoft.AspNetCore.Authentication.JwtBearer`
- [ ] Đã cấu hình đúng `Authority` (URL IdentityServer) và `Audience`
- [ ] Đã set `ValidateAudience = true` và `ValidTypes` phù hợp (VD: "at+jwt")
- [ ] Đã thêm AuthorizationPolicy yêu cầu `RequireAuthenticatedUser()` và áp dụng cho route/controller

### Khi bảo vệ tại API Gateway (nối Chapter 11)
- [ ] Đã cấu hình `AuthenticationOptions` trong ocelot.json với đúng `AuthenticationProviderKey` và `AllowedScopes`
- [ ] Đã đăng ký đủ scope tương ứng (VD: appointments.read/write) trong Config.cs của IdentityServer
- [ ] Client machine-to-machine của gateway đã được cấp đúng scope write cần thiết để gọi downstream
- [ ] Client interactive đã được cấp đủ scope read+write để UI hoạt động không cần đăng ký client riêng
- [ ] Đã xác nhận response 401 khi thiếu token, 403 khi thiếu scope/claim

---

## Anti-patterns / Lưu ý cần tránh

1. **Nhầm lẫn Encoding và Encryption**: JWT chỉ được ENCODE (Base64), KHÔNG được ENCRYPT — ai cũng decode đọc được nội dung. Không bao giờ đưa dữ liệu nhạy cảm vào claim.
2. **Dựa hoàn toàn vào bearer token per-service mà không có kế hoạch revocation**: khi vai trò/quyền người dùng thay đổi, các service không biết cho tới khi token hết hạn tự nhiên — rủi ro bảo mật thực sự nếu token sống quá lâu.
3. **Forward nguyên token gốc của người dùng cho service-to-service call mà không cân nhắc phạm vi quyền**: vi phạm nguyên tắc "quyền tối thiểu" (least privilege), token có thể mang quyền rộng hơn cần thiết cho thao tác cụ thể.
4. **Mỗi service tự viết logic parse/validate JWT riêng**: dẫn tới trùng lặp code, không nhất quán, khó bảo trì khi có thay đổi chuẩn bảo mật.
5. **Dùng Implicit Grant hoặc Resource Owner Password Credentials cho ứng dụng mới**: cả hai đều không còn được khuyến khích do rủi ro bảo mật — ưu tiên Authorization Code + PKCE.
6. **Cấu hình CORS quá lỏng lẻo hoặc để cổng/API mở không cần thiết**: là một trong các lỗ hổng phổ biến nhất ở ứng dụng cloud-native/microservices.
7. **Bỏ qua thiết lập Audience validation**: nếu không validate đúng audience, một token phát hành cho service A có thể bị dùng sai mục đích ở service B.
8. **Thời gian sống token quá dài**: tăng cửa sổ rủi ro nếu token bị đánh cắp — vì JWT không dễ thu hồi.

---

## Từ vựng chuyên ngành (Anh - Việt)

| Thuật ngữ | Nghĩa |
|---|---|
| Authentication | Xác thực |
| Authorization | Ủy quyền |
| Bearer Token | Token người mang (dựa trên nguyên tắc sở hữu) |
| Claim | Thông tin/tuyên bố chứa trong token |
| Identity Provider (IdP) | Nhà cung cấp danh tính |
| Access Token | Token truy cập |
| Refresh Token | Token làm mới |
| Scope | Phạm vi quyền |
| Grant Type | Loại cấp quyền |
| Single Sign-On (SSO) | Đăng nhập một lần |
| Revocation | Thu hồi |
| Delegation | Ủy thác/hành động thay mặt |
| Role-Based Access Control (RBAC) | Kiểm soát truy cập theo vai trò |
| Man-in-the-Middle (MITM) | Tấn công người đứng giữa |
| Personally Identifiable Information (PII) | Thông tin định danh cá nhân |
| Discovery Document | Tài liệu khám phá (metadata IdP) |
| Proof Key for Code Exchange (PKCE) | Khóa chứng minh cho trao đổi mã |
| Client Credentials | Thông tin xác thực client (machine-to-machine) |
| Least Privilege | Nguyên tắc quyền tối thiểu |

---

## Câu hỏi phỏng vấn thường gặp + hướng trả lời

**Q1: Phân biệt Authentication và Authorization?**
→ Authentication là quá trình xác lập danh tính — trả lời câu hỏi "bạn là ai?". Authorization là bước tiếp theo, sau khi đã xác thực, quyết định danh tính đó được PHÉP làm gì — trả lời câu hỏi "bạn được làm gì?". Một hệ thống có thể xác thực đúng người nhưng vẫn từ chối thao tác nếu người đó không có quyền phù hợp.

**Q2: JWT có được mã hóa (encrypted) không?**
→ Không. JWT chỉ được ENCODE (thường ở dạng Base64), không được ENCRYPT. Bất kỳ ai có token đều có thể decode và đọc nội dung claim bên trong (dù không thể giả mạo vì có chữ ký). Vì vậy, không bao giờ nên đưa dữ liệu nhạy cảm (mật khẩu, số thẻ tín dụng...) vào claim của token.

**Q3: Tại sao bearer token đơn lẻ (không qua IdP tập trung) lại có giới hạn khi dùng trong microservices?**
→ Ba giới hạn chính: (1) không hỗ trợ tốt việc thu hồi thời gian thực — nếu quyền người dùng thay đổi, service không biết cho tới khi token hết hạn; (2) không có cơ chế delegation gốc, khiến service-to-service call phải chọn giữa forward nguyên token (dư quyền) hoặc tự hành động (vi phạm least privilege); (3) mỗi service phải tự triển khai logic parse/validate/audit riêng, dẫn tới trùng lặp và thiếu nhất quán.

**Q4: OAuth 2.0 và OpenID Connect khác nhau ở điểm nào?**
→ OAuth 2.0 là giao thức ỦY QUYỀN (authorization) — nó giải quyết bài toán client được phép truy cập tài nguyên nào, không quan tâm tới việc xác thực người dùng. OIDC là một lớp danh tính XÂY TRÊN OAuth 2.0, bổ sung khả năng XÁC THỰC (authentication) — cho biết người dùng thực sự là ai, thông qua ID token (một JWT chứa claim về người dùng).

**Q5: Vì sao nên bảo vệ tại API Gateway thay vì từng microservice riêng lẻ?**
→ Bảo vệ tại gateway tập trung hóa logic xác thực/ủy quyền vào một điểm duy nhất, giúp gateway điều phối luồng xác thực cho client và quản lý việc chia sẻ token giữa các lời gọi service phía sau — giảm số lần người dùng phải xác thực, giảm trùng lặp logic bảo mật ở từng service, và đặc biệt hiệu quả khi kết hợp với BFF Pattern (đã học ở Chapter 11).

**Q6: PKCE là gì và tại sao cần dùng cùng Authorization Code Grant?**
→ PKCE (Proof Key for Code Exchange) là cơ chế bảo mật bổ sung cho luồng Authorization Code, đặc biệt quan trọng với client không thể giữ bí mật client secret an toàn (như ứng dụng chạy trên trình duyệt hoặc mobile). Nó ngăn chặn kiểu tấn công đánh cắp authorization code bằng cách yêu cầu client tạo một "code verifier" ngẫu nhiên và gửi kèm "code challenge" tương ứng khi bắt đầu luồng, rồi xác minh lại verifier khi đổi code lấy token.

**Q7: Access Token và Bearer Token khác nhau ở điểm nào?**
→ Bearer Token là một LOẠI access token, hoạt động theo nguyên tắc "ai sở hữu token thì được cấp quyền truy cập". Access Token là khái niệm RỘNG HƠN, định nghĩa trong khuôn khổ OAuth 2.0, đại diện cho quyền ủy quyền cụ thể được cấp cho client, tuân theo chuẩn nghiêm ngặt (RFC 6749/6750) và đi kèm metadata như scope, audience, issuer, expiration. Nói cách khác: Access Token CÓ THỂ là Bearer Token, nhưng không phải Bearer Token nào cũng đóng vai trò Access Token theo chuẩn OAuth.

**Q8: Khi cấu hình gateway (Ocelot) để xác thực qua IdentityServer, cần lưu ý gì về scope?**
→ Cần đảm bảo scope khai báo trong `AllowedScopes` của route ocelot.json (VD: appointments.read/write) phải khớp với scope đã cấp cho đúng client tương ứng trong Config.cs của IdentityServer — client machine-to-machine của gateway cần scope write để gọi downstream thay mặt hệ thống, còn client interactive cần cả read lẫn write để UI hoạt động đầy đủ mà không cần đăng ký client riêng cho từng thao tác.

---

## Bài tập thực hành đề xuất

1. Dựng một project Duende IdentityServer (`dotnet new isef`) độc lập, cấu hình 2 client: một Client Credentials (machine-to-machine) và một Authorization Code + PKCE (interactive).
2. Dùng Postman lấy access token qua Client Credentials flow, decode token bằng jwt.io để xác nhận các claim (iss, aud, scope, client_id, jti) đúng như kỳ vọng.
3. Bảo vệ một API đơn giản (VD: Appointments API giả lập) bằng JWT Bearer authentication, trỏ Authority về IdentityServer vừa dựng, xác nhận request thiếu token trả về 401.
4. Thêm 2 scope mới (VD: patients.read, patients.write) vào IdentityServer, cập nhật AllowedScopes cho client tương ứng, và cấu hình một route Ocelot gateway yêu cầu đúng 2 scope này.
5. Thử nghiệm kịch bản: gọi API trực tiếp (không qua gateway) so với gọi qua gateway đã cấu hình AuthenticationOptions — so sánh trải nghiệm và độ phức tạp cấu hình giữa hai cách.

---

*Ghi chú: Chương tiếp theo (Chapter 14, dự kiến) sẽ chuyển sang chủ đề Containers và Orchestration để xây dựng ứng dụng microservices cloud-native — một Part mới trong cấu trúc sách, tiếp nối các pattern về resiliency, security, và infrastructure đã hoàn tất từ Chapter 10 đến Chapter 13.*
