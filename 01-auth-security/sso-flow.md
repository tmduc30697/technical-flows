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

## Nền tảng giáo dục trực tuyến với nhiều app con dùng chung tài khoản

**Repository:** `sso-edtech-shared-account`

**Hệ thống:** Một nền tảng học trực tuyến có nhiều sản phẩm riêng biệt (app học viên, app giảng viên, cổng thi thử, forum thảo luận) dùng chung một hệ thống tài khoản người dùng.

**Vai trò của flow:** Nền tảng tự làm Identity Provider cho hệ sinh thái sản phẩm của chính mình, cho phép người dùng chuyển qua lại giữa các app con mà chỉ cần đăng nhập một lần trên trình duyệt.

**Yêu cầu cụ thể:**
- Một tài khoản có thể mang nhiều vai trò khác nhau ở các app con (vừa là học viên ở app A, vừa là giảng viên ở app B) — token/session phải mang đủ thông tin để mỗi app tự quyết định quyền truy cập.
- Xử lý được trường hợp người dùng mở nhiều app con cùng lúc trên nhiều tab, đảm bảo không tạo ra nhiều session trùng lặp không cần thiết ở IdP.
- Access token dùng cho các app con phải có thời hạn ngắn, kèm refresh token/refresh session để duy trì trải nghiệm "luôn đăng nhập" mà không lộ token dài hạn cho từng app.
- Khi người dùng đổi email/password ở app chính, tất cả session SSO đang hoạt động ở app con phải được yêu cầu xác thực lại.
- Thiết kế cho khả năng mở rộng: thêm một app con mới vào hệ sinh thái không cần sửa logic ở IdP trung tâm, chỉ cần đăng ký app như một client mới.

---

## Cổng thông tin y tế cho nhân viên nhiều bệnh viện trong một hệ thống y tế

**Repository:** `sso-healthcare-multi-hospital-portal`

**Hệ thống:** Một hệ thống y tế có nhiều bệnh viện/chi nhánh, mỗi chi nhánh dùng Active Directory riêng, nhưng cần một cổng thông tin chung để nhân viên y tế truy cập hồ sơ bệnh án điện tử.

**Vai trò của flow:** Cổng thông tin đóng vai trò Service Provider tích hợp SSO với AD Federation Services (ADFS/SAML) của từng chi nhánh, đồng thời phải đáp ứng yêu cầu tuân thủ nghiêm ngặt về bảo mật dữ liệu y tế.

**Yêu cầu cụ thể:**
- Session SSO phải có thời gian hết hạn ngắn hơn nhiều so với SSO thông thường (ví dụ 15 phút không hoạt động) và tự động đăng xuất khi máy trạm khoá màn hình.
- Phải hỗ trợ step-up: một số hành vi nhạy cảm (xem hồ sơ bệnh nhân VIP, xuất dữ liệu) yêu cầu xác thực lại dù session SSO còn hạn.
- Nếu ADFS của một chi nhánh gặp sự cố, nhân viên chi nhánh đó không được tự động fallback sang đăng nhập bằng password không an toàn — phải có quy trình dự phòng được kiểm soát riêng.
- Mọi lần SSO thành công/thất bại phải ghi log đầy đủ (ai, chi nhánh nào, thời gian, IP) để đáp ứng audit trail theo quy định y tế.
- Định danh nhân viên từ SAML assertion phải được map đúng với vai trò chuyên môn (bác sĩ, điều dưỡng, admin) đã lưu sẵn trong hệ thống, không suy ra quyền chỉ từ group AD một cách máy móc.

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
