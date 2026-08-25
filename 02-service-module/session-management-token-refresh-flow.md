# Session management & token refresh flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (ngân hàng số mobile-first, SaaS web với SSO, game multiplayer, streaming giới hạn thiết bị, cổng quản trị nội bộ) để luyện việc quản lý session và refresh token an toàn, đúng và chịu được race condition.

---

## Session cho ứng dụng ngân hàng số trên mobile

**Repository:** `session-management-digital-bank-mobile`

**Hệ thống:** Một app ngân hàng số cho phép user xem số dư, chuyển tiền, đăng nhập bằng access token ngắn hạn + refresh token dài hạn.

**Vai trò của flow:** Session management đảm bảo user không phải đăng nhập lại liên tục nhưng access token bị lộ cũng chỉ gây rủi ro trong thời gian rất ngắn, phù hợp với yêu cầu bảo mật cao của ngành ngân hàng.

**Yêu cầu cụ thể:**
- Access token có thời hạn ngắn (ví dụ 5-15 phút); refresh token có thời hạn dài hơn nhưng phải áp dụng refresh token rotation — mỗi lần refresh phát ra refresh token mới và vô hiệu hóa refresh token cũ ngay lập tức.
- Xử lý race condition khi app gọi refresh hai lần gần như đồng thời (do nhiều tab/request song song trong app) bằng cùng một refresh token — chỉ một request được cấp token mới thành công, request còn lại phải nhận lỗi rõ ràng và dùng lại token mới vừa cấp, không tạo ra hai session song song không đồng bộ.
- Nếu một refresh token đã bị dùng (rotate) mà bị dùng lại lần nữa (dấu hiệu refresh token bị đánh cắp và dùng song song với thiết bị thật), hệ thống phải revoke toàn bộ chuỗi session liên quan và bắt buộc đăng nhập lại, đồng thời cảnh báo cho user qua kênh khác (email/SMS).
- Cho phép user xem danh sách thiết bị/session đang đăng nhập và chủ động revoke một session cụ thể từ xa (ví dụ mất điện thoại) — revoke phải có hiệu lực ngay, không chờ access token cũ tự hết hạn.
- Đảm bảo access token không được lưu ở nơi kém an toàn trên thiết bị (không log ra file, không lưu plaintext ngoài secure storage của OS), và refresh token không bao giờ được gửi kèm trong mọi API call ngoài API refresh chuyên biệt.

---

## Session dùng chung qua SSO cho nền tảng SaaS B2B nhiều sản phẩm

**Repository:** `session-management-b2b-saas-sso-shared`

**Hệ thống:** Một công ty có 3 sản phẩm SaaS con (CRM, Helpdesk, Analytics) dùng chung một hệ thống đăng nhập trung tâm (SSO), user đăng nhập một lần dùng được cả 3 sản phẩm.

**Vai trò của flow:** Session/token phải được chia sẻ và refresh nhất quán giữa các sản phẩm, để user không phải đăng nhập lại ở mỗi sản phẩm và không bị đăng xuất không đồng bộ.

**Yêu cầu cụ thể:**
- Thiết kế token trung tâm (ví dụ id_token/access_token từ SSO) mà cả 3 sản phẩm đều tin dùng, kèm cơ chế mỗi sản phẩm tự refresh token con của mình khi cần mà không phải bắt user quay lại màn hình SSO mỗi lần.
- Khi user logout ở một sản phẩm (ví dụ CRM), phải định nghĩa rõ đây là "logout khỏi sản phẩm đó" hay "logout toàn bộ SSO" (single logout) — và nếu là single logout, phải đảm bảo cả 3 sản phẩm đều nhận được tín hiệu và vô hiệu hóa session gần như đồng thời.
- Xử lý trường hợp một sản phẩm phát hiện token đã bị revoke (ví dụ admin công ty disable user) trong khi sản phẩm khác vẫn đang có session hoạt động — cần một cơ chế lan truyền trạng thái revoke (webhook/event) không dựa hoàn toàn vào access token tự hết hạn.
- Đảm bảo mỗi sản phẩm không lưu trữ trực tiếp thông tin nhạy cảm của token trung tâm mà chỉ giữ token phạm vi (scoped token) riêng cho mình, giảm thiệt hại nếu một sản phẩm bị lộ dữ liệu.
- Xử lý clock skew giữa các server của 3 sản phẩm khi validate thời hạn token (exp/iat) để tránh trường hợp một sản phẩm coi token đã hết hạn sớm hơn thực tế do lệch giờ hệ thống.

---

## Quản lý session cho game multiplayer thời gian thực

**Repository:** `session-management-game-multiplayer`

**Hệ thống:** Một backend game nhiều người chơi, mỗi trận đấu kéo dài 15-30 phút, người chơi giữ một session kết nối liên tục qua WebSocket trong suốt trận.

**Vai trò của flow:** Session phải được giữ ổn định suốt trận đấu dù access token hết hạn giữa lúc đang chơi, và phải xử lý được việc người chơi mất mạng tạm thời rồi reconnect.

**Yêu cầu cụ thể:**
- Access token hết hạn giữa trận không được làm ngắt kết nối WebSocket đang chơi — cần cơ chế refresh "ngầm" (silent refresh) chạy song song mà không làm gián đoạn luồng game data đang truyền.
- Khi người chơi mất mạng và reconnect trong một khoảng thời gian ngắn (ví dụ dưới 30 giây), phải cho phép dùng lại đúng session/token cũ để vào lại đúng trận đang chơi, không bị coi là một lần đăng nhập mới và mất vị trí/trạng thái trong trận.
- Nếu người chơi không reconnect được trong thời gian cho phép, session phải được đóng dứt điểm ở phía server (giải phóng slot trong trận cho người khác vào thay nếu là chế độ có thể thay người), tránh giữ "ghế trống" vô thời hạn.
- Xử lý gian lận: refresh token của một tài khoản không được dùng đồng thời để mở nhiều session game cùng lúc từ hai thiết bị khác nhau vào cùng một trận (phát hiện và chặn multi-login bất thường trong lúc trận đang diễn ra).
- Đảm bảo việc refresh token diễn ra không gây lag/giật hình cho người chơi — token refresh phải chạy trên một kênh riêng, không chia sẻ băng thông/luồng xử lý với kênh truyền dữ liệu game real-time.

---

## Giới hạn số thiết bị đăng nhập đồng thời cho dịch vụ streaming

**Repository:** `session-management-streaming-device-limit`

**Hệ thống:** Một dịch vụ streaming nhạc/video theo subscription, giới hạn tối đa N thiết bị đăng nhập/stream đồng thời trên một tài khoản.

**Vai trò của flow:** Session management phải theo dõi chính xác thiết bị nào đang hoạt động để enforce giới hạn, đồng thời vẫn cho phép refresh token mượt mà trên các thiết bị đang dùng hợp lệ.

**Yêu cầu cụ thể:**
- Mỗi lần đăng nhập trên một thiết bị mới phải tạo một session riêng biệt (không dùng chung refresh token giữa các thiết bị), và hệ thống phải biết chính xác có bao nhiêu session/thiết bị đang active tại một thời điểm.
- Khi user đăng nhập trên thiết bị thứ N+1 (vượt giới hạn), phải có chính sách rõ ràng (chặn đăng nhập mới, hoặc tự động đăng xuất thiết bị inactive lâu nhất) và thông báo minh bạch cho user, không âm thầm đăng xuất thiết bị đang xem gây trải nghiệm xấu.
- Xử lý race condition khi hai thiết bị mới cố gắng đăng nhập gần như đồng thời khi tài khoản đang ở đúng ngưỡng giới hạn (N-1 thiết bị active) — không được để cả hai đều thành công vượt quá N.
- Refresh token của một thiết bị phải hết hạn/vô hiệu ngay khi thiết bị đó bị đăng xuất (chủ động hoặc do vượt giới hạn), không để access token cũ còn dùng được tới khi tự hết hạn tự nhiên.
- Cung cấp cho user màn hình quản lý thiết bị đang đăng nhập, với thông tin đủ để nhận diện (loại thiết bị, thời điểm đăng nhập gần nhất, vị trí địa lý gần đúng) và khả năng đăng xuất từ xa một thiết bị cụ thể.

---

## Session cho cổng quản trị nội bộ có yêu cầu bảo mật khác nhau theo vai trò

**Repository:** `session-management-admin-portal-role-based`

**Hệ thống:** Một cổng quản trị nội bộ doanh nghiệp cho nhân viên (xem báo cáo) và admin (thay đổi cấu hình hệ thống nhạy cảm) cùng dùng chung một hệ thống đăng nhập.

**Vai trò của flow:** Session/token phải áp dụng chính sách thời hạn và bảo mật khác nhau theo mức độ nhạy cảm của hành động, không dùng một chính sách chung cho mọi vai trò.

**Yêu cầu cụ thể:**
- Session của tài khoản có quyền admin (có thể thay đổi cấu hình hệ thống) phải có thời hạn ngắn hơn và bắt buộc re-authentication (nhập lại mật khẩu/MFA) trước khi thực hiện các hành động nhạy cảm cụ thể (step-up authentication), dù access token tổng thể vẫn còn hạn.
- Với nhân viên thường chỉ xem báo cáo, session có thể có thời hạn dài hơn và refresh mượt mà hơn để không làm gián đoạn công việc hàng ngày.
- Khi một admin refresh token, hệ thống phải kiểm tra lại xem quyền hạn của tài khoản đó có thay đổi từ lần đăng nhập trước không (ví dụ vừa bị hạ quyền) và cấp access token mới phản ánh đúng quyền hiện tại, không giữ nguyên quyền cũ đã lỗi thời trong token.
- Đảm bảo access token mang theo đủ thông tin về vai trò/quyền hạn để các service nội bộ tự kiểm tra (authorization) mà không phải gọi lại tới auth service mỗi request, nhưng vẫn phải có cơ chế revoke ngay lập tức khi cần (ví dụ nhân viên bị đuổi việc) mà không chờ token tự hết hạn.
- Ghi log chi tiết mọi lần refresh token và mọi hành động step-up authentication của các tài khoản admin, phục vụ audit bảo mật nội bộ định kỳ.
