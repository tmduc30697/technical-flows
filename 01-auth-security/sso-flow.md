# Single Sign-On (SSO) flow — Đề bài thực hành

Các đề bài dưới đây đi từ vai trò **Service Provider (SP)** tích hợp với Identity Provider (IdP) bên ngoài của khách hàng doanh nghiệp, đến vai trò **tự làm Identity Provider** cho một hệ sinh thái nhiều app con, qua các bối cảnh B2B SaaS, nội bộ doanh nghiệp, nền tảng công cộng đa app, y tế và fintech.

---

## B2B SaaS đa khách hàng tích hợp Okta/Azure AD của từng công ty

**Repository:** `sso-b2b-saas-okta-azure-ad`

**Hệ thống:** Một SaaS quản lý dự án bán cho nhiều công ty khách hàng (multi-tenant), mỗi công ty có hệ thống IdP riêng (Okta, Azure AD, Google Workspace).

**Vai trò của flow:** App đóng vai trò Service Provider, cho phép mỗi tenant tự cấu hình SSO (SAML hoặc OIDC) với IdP của chính họ để nhân viên đăng nhập bằng tài khoản công ty.

**Yêu cầu cụ thể:**
- Mỗi tenant tự nhập metadata IdP (SSO URL, certificate/issuer) qua trang admin riêng, không hard-code cho một provider cụ thể — phải trừu tượng hóa được cả SAML và OIDC sau một interface chung.
- Xác định đúng tenant nào cần redirect tới IdP nào khi user chỉ nhập email (email domain discovery), tránh gửi user tới IdP sai công ty.
- Validate chữ ký SAML response / ID token đúng chuẩn (không tin tưởng field chưa verify), chống replay bằng kiểm tra `InResponseTo`/nonce và thời gian hết hạn của assertion.
- Xử lý case một công ty bắt buộc SSO (disable đăng nhập bằng password) nhưng vẫn cần tài khoản "break-glass" cho admin trong trường hợp IdP sập.
- Tự động provision user mới lần đầu đăng nhập qua SSO (JIT provisioning) và deprovision/khóa account khi IdP báo user bị xóa (SCIM hoặc webhook), không để tài khoản "mồ côi" còn quyền truy cập.

---

## Cổng đăng nhập chung cho các app nội bộ doanh nghiệp

**Repository:** `sso-enterprise-internal-portal`

**Hệ thống:** Một công ty có nhiều app nội bộ tách biệt (HR portal, hệ thống chấm công, wiki nội bộ, công cụ báo cáo chi phí), mỗi app hiện có login riêng.

**Vai trò của flow:** Xây một Identity Provider nội bộ tập trung — nhân viên đăng nhập một lần, sau đó truy cập mọi app con mà không cần đăng nhập lại (SSO nội bộ giữa các subdomain/app riêng biệt).

**Yêu cầu cụ thể:**
- Session đăng nhập trung tâm phải chia sẻ được giữa các app nằm trên domain/subdomain khác nhau, kèm cơ chế chống session fixation khi chuyển giữa các app.
- Khi nhân viên nghỉ việc, revoke session ở IdP trung tâm phải làm mất quyền truy cập ở tất cả app con gần như ngay lập tức (không chờ từng app tự hết session).
- Hỗ trợ Single Logout (SLO): đăng xuất ở một app phải đăng xuất khỏi toàn bộ app con đã SSO vào, kể cả khi user chỉ đóng tab mà không bấm logout rõ ràng ở app đó.
- Phân quyền theo app phải dựa trên role/group đồng bộ từ hệ thống HR (nguồn dữ liệu nhân sự), không quản lý quyền riêng lẻ ở từng app.
- Có cơ chế audit log tập trung: ghi lại mọi lần đăng nhập SSO, app nào được truy cập, để phục vụ điều tra khi có sự cố an ninh.

---

## Fintech cho phép SSO giữa web, mobile app và đối tác nhúng (embedded finance)

**Repository:** `sso-fintech-embedded-finance`

**Hệ thống:** Một nền tảng fintech cung cấp dịch vụ thanh toán, có web app, mobile app riêng, và cho phép đối tác nhúng (embed) trải nghiệm đăng nhập vào app của đối tác (embedded finance).

**Vai trò của flow:** Nền tảng vừa là IdP cho web/mobile app của chính mình, vừa cấp một luồng SSO hạn chế cho đối tác thứ ba nhúng qua iframe/webview mà không lộ thông tin đăng nhập gốc.

**Yêu cầu cụ thể:**
- Session SSO giữa web và mobile app phải đồng bộ trạng thái đăng nhập (đăng xuất trên web phải phản ánh gần như ngay lập tức trên mobile) qua cơ chế push/token revocation, không chỉ chờ token tự hết hạn.
- Luồng nhúng cho đối tác phải giới hạn phạm vi truy cập (không cấp toàn quyền như đăng nhập trực tiếp), và không cho phép đối tác thấy hoặc lưu lại thông tin đăng nhập gốc của người dùng.
- Phải có Single Logout xuyên suốt: người dùng đăng xuất ở một nơi phải kết thúc session ở cả web, mobile và mọi phiên nhúng ở đối tác liên quan.
- Có cơ chế phát hiện và ngăn brute-force/token theft trong quá trình SSO (ví dụ nhiều lần thử với state/nonce sai từ cùng một IP).
- Ghi log audit đầy đủ cho mọi phiên SSO liên quan đến tiền (ai, từ thiết bị nào, qua kênh nào) để đáp ứng yêu cầu compliance tài chính.
