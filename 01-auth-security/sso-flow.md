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

---

## Hệ sinh thái nhiều app tiêu dùng dùng chung một tài khoản đăng nhập kiểu Google

**Repository:** `sso-consumer-platform-multi-app-ecosystem`

**Hệ thống:** Một công ty vận hành nhiều app tiêu dùng riêng biệt (ví dụ app email, app lưu trữ file, app thanh toán, app bản đồ) mà người dùng cuối dùng chung một tài khoản để đăng nhập vào tất cả.

**Vai trò của flow:** Nền tảng tự làm Identity Provider trung tâm cho toàn bộ hệ sinh thái, quản lý một danh tính người dùng nhưng vai trò/quyền truy cập khác nhau ở từng app, và phải chịu tải xác thực ở quy mô rất lớn.

**Yêu cầu cụ thể:**
- Một người dùng có thể có vai trò khác nhau hoàn toàn ở từng app, ví dụ là admin ở app lưu trữ file của tổ chức họ nhưng chỉ là user thường ở app bản đồ — token SSO trung tâm không thể mang theo toàn bộ quyền của mọi app, mỗi app cần tự truy vấn quyền riêng của nó sau khi xác thực danh tính thành công, tránh token phình to hoặc lộ thông tin quyền không liên quan.
- Khi người dùng đổi thông tin nhạy cảm ở tài khoản trung tâm (đổi email, đổi mật khẩu, bật thêm MFA), mọi app con đang có session hoạt động phải nhận được tín hiệu để yêu cầu xác thực lại ở mức phù hợp, tránh trường hợp kẻ tấn công vừa chiếm được tài khoản trung tâm vẫn tiếp tục dùng session cũ ở các app khác vô thời hạn.
- Người dùng có thể dùng cùng một tài khoản cho cả mục đích cá nhân và mục đích qua một tổ chức (ví dụ tài khoản cá nhân nhưng cũng là thành viên workspace công ty) — cần phân tách rõ ngữ cảnh đăng nhập cá nhân/tổ chức để tránh nhầm lẫn quyền, đặc biệt khi chính sách bảo mật giữa hai ngữ cảnh khác nhau đáng kể.
- Ở quy mô hệ sinh thái lớn, một app con bị lỗi hoặc quá tải khi gọi về IdP trung tâm để xác thực không được phép làm sập khả năng đăng nhập của toàn bộ hệ sinh thái — cần cách ly lỗi và có chiến lược cache/fallback hợp lý cho việc xác thực mà không hy sinh khả năng revoke quyền kịp thời khi cần.
- Khi người dùng thu hồi quyền truy cập của một app cụ thể trong hệ sinh thái mà không muốn ảnh hưởng các app khác, hệ thống phải hỗ trợ revoke theo phạm vi từng app riêng lẻ, không cần đăng xuất toàn bộ hệ sinh thái chỉ vì muốn ngắt kết nối một app.

---

## SSO giữa hệ thống bệnh viện trung tâm và mạng lưới phòng khám liên kết

**Repository:** `sso-healthcare-hospital-clinic-network`

**Hệ thống:** Một hệ thống bệnh viện trung tâm cung cấp hồ sơ bệnh án điện tử (EHR) dùng chung cho một mạng lưới các phòng khám liên kết độc lập về vận hành nhưng cần truy cập chung một nguồn dữ liệu bệnh nhân.

**Vai trò của flow:** Bệnh viện trung tâm đóng vai trò Identity Provider, cho phép nhân viên y tế ở các phòng khám liên kết SSO vào hệ thống hồ sơ bệnh án trung tâm, đồng thời phải đáp ứng yêu cầu compliance nghiêm ngặt về audit truy cập dữ liệu bệnh nhân.

**Yêu cầu cụ thể:**
- Mỗi lần truy cập hồ sơ bệnh nhân qua phiên SSO phải được ghi log gắn liền với danh tính nhân viên y tế cụ thể, không chỉ gắn với phòng khám, cùng với hồ sơ bệnh nhân nào được xem, để đáp ứng yêu cầu truy vết "ai xem hồ sơ của ai, khi nào, vì lý do gì" khi có thanh tra hoặc khiếu nại.
- Quyền truy cập hồ sơ bệnh nhân qua SSO phải giới hạn theo mối quan hệ điều trị thực tế, tức là nhân viên phòng khám chỉ xem được hồ sơ bệnh nhân đang hoặc đã điều trị tại phòng khám đó, không phải cứ SSO thành công là xem được toàn bộ hồ sơ trong hệ thống trung tâm.
- Khi một phòng khám liên kết chấm dứt hợp đồng với bệnh viện trung tâm, toàn bộ quyền SSO của nhân viên phòng khám đó phải bị thu hồi ngay lập tức và đồng loạt, kể cả với những nhân viên đang có session hoạt động, để tránh truy cập trái phép hồ sơ bệnh nhân sau khi quan hệ đã chấm dứt.
- Phải xử lý được tình huống khẩn cấp lâm sàng, ví dụ bệnh nhân chuyển viện gấp từ phòng khám lên bệnh viện trung tâm cần chia sẻ hồ sơ ngay, bằng một luồng truy cập mở rộng có kiểm soát và giới hạn thời gian, tách biệt khỏi quyền truy cập SSO thông thường và luôn kèm log để review sau.
- Vì thông tin sức khỏe là dữ liệu đặc biệt nhạy cảm, phiên SSO cho nhân viên y tế cần thời gian sống ngắn hơn và bắt buộc xác thực lại thường xuyên hơn so với hệ thống nội bộ thông thường, đặc biệt trên các máy trạm dùng chung tại phòng khám qua nhiều ca kíp nhân viên khác nhau, để tránh một nhân viên vô tình thao tác dưới session còn sót lại của người trước.
