# Notification fan-out flow (push/email/SMS đa kênh) — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — e-commerce, fintech, mạng xã hội, SaaS B2B vận hành hệ thống, ứng dụng gọi xe, và nền tảng marketing — nhằm luyện đủ các góc của flow gửi thông báo đa kênh (ưu tiên kênh, throughput lớn, retry/fallback, chống spam, tuân thủ preference người dùng).

---

## E-commerce thông báo trạng thái đơn hàng

**Repository:** `notification-fanout-ecommerce-order-status`

**Hệ thống:** Một sàn e-commerce cần thông báo cho khách hàng mỗi khi đơn hàng chuyển trạng thái (đã xác nhận, đang giao, đã giao).

**Vai trò của flow:** Nhận sự kiện thay đổi trạng thái đơn hàng và fan-out thông báo tới các kênh phù hợp (push app, email, SMS) theo preference của từng khách hàng.

**Yêu cầu cụ thể:**
- Mỗi loại sự kiện đơn hàng phải map tới một tập kênh mặc định khác nhau (ví dụ "đã giao" ưu tiên push + SMS, "xác nhận đơn" chỉ cần email), và khách hàng có thể tùy chỉnh kênh nhận theo từng loại sự kiện.
- Nếu gửi push thất bại (app đã bị gỡ, token hết hạn), phải tự động fallback sang kênh khác (SMS/email) theo thứ tự ưu tiên đã định, không để khách hàng bị bỏ sót thông báo quan trọng.
- Xử lý được trường hợp trạng thái đơn hàng đổi liên tiếp rất nhanh (ví dụ do lỗi hệ thống vận chuyển cập nhật lặp) — không gửi thông báo trùng lặp gây spam cho khách hàng.
- Đảm bảo thứ tự thông báo hiển thị đúng theo thứ tự sự kiện thực tế xảy ra, dù các kênh khác nhau có độ trễ gửi khác nhau (ví dụ SMS chậm hơn push).
- Có cơ chế đo tỷ lệ gửi thành công/thất bại theo từng kênh để phát hiện sớm khi một nhà cung cấp SMS/push gặp sự cố, và tự động route sang provider dự phòng nếu có.

---

## Fintech gửi cảnh báo giao dịch và OTP

**Repository:** `notification-fanout-fintech-transaction-otp`

**Hệ thống:** Một ứng dụng ngân hàng số cần gửi cảnh báo mỗi khi có giao dịch phát sinh và mã OTP xác thực cho các hành động nhạy cảm.

**Vai trò của flow:** Fan-out thông báo giao dịch/OTP tới đúng kênh với độ tin cậy và tốc độ cao, đáp ứng yêu cầu bảo mật và tuân thủ ngành ngân hàng.

**Yêu cầu cụ thể:**
- OTP phải được gửi qua kênh có độ trễ thấp và đảm bảo nhất (SMS hoặc push xác thực) trong vòng vài giây; nếu kênh chính thất bại phải fallback ngay sang kênh khác trước khi OTP hết hạn hiệu lực.
- Không được để hai OTP hợp lệ cùng tồn tại cho một hành động của một user tại cùng thời điểm — OTP mới phát sinh phải vô hiệu hóa OTP cũ chưa dùng, dù việc gửi OTP cũ có thể đang chạy dở trên hàng đợi.
- Cảnh báo giao dịch phải được gửi độc lập với luồng xử lý giao dịch chính — lỗi ở hệ thống thông báo không được làm rollback hoặc chậm trễ giao dịch đang xử lý.
- Toàn bộ log gửi OTP/cảnh báo (thời gian, kênh, trạng thái gửi) phải được lưu đủ để phục vụ tra soát khi khách hàng khiếu nại không nhận được cảnh báo cho một giao dịch gian lận.
- Phải giới hạn số lượng OTP gửi cho một số điện thoại/tài khoản trong một khoảng thời gian để chống lạm dụng (kẻ gian spam OTP để tấn công), nhưng không được chặn nhầm người dùng hợp lệ đang thực sự cần xác thực nhiều lần.

---

## Mạng xã hội fan-out thông báo like/comment/follow

**Repository:** `notification-fanout-social-engagement`

**Hệ thống:** Một mạng xã hội cần thông báo cho user khi có người like/comment bài viết của họ hoặc follow họ, với khả năng một bài viết có thể viral và nhận hàng trăm nghìn tương tác.

**Vai trò của flow:** Gom nhóm và fan-out thông báo tương tác tới chủ bài viết/người bị follow theo kênh phù hợp, xử lý được trường hợp tải cực lớn khi nội dung viral.

**Yêu cầu cụ thể:**
- Khi một bài viết nhận lượng like/comment tăng vọt trong thời gian ngắn, hệ thống phải gộp nhóm thông báo (ví dụ "A và 999 người khác đã thích bài viết của bạn") thay vì gửi từng thông báo riêng lẻ gây ngập inbox/push.
- Fan-out cho tài khoản có lượng follower rất lớn (celebrity) không được làm nghẽn hàng đợi thông báo chung của toàn hệ thống — cần cơ chế xử lý tách riêng cho các trường hợp fan-out quy mô lớn.
- Người dùng phải tùy chỉnh được loại tương tác nào muốn nhận push/email, và thay đổi preference phải có hiệu lực gần như ngay lập tức cho các thông báo tiếp theo, không cần chờ đồng bộ theo batch.
- Thông báo phải khử trùng đúng cách khi có nhiều sự kiện liên quan đến cùng một đối tượng xảy ra gần nhau (ví dụ user vừa like vừa comment cùng lúc) — quyết định rõ có gộp thành một thông báo hay giữ riêng.
- Đo lường được thời gian từ lúc sự kiện xảy ra đến lúc thông báo tới tay người dùng (fan-out latency) và đảm bảo latency không tăng phi tuyến theo số lượng follower của đối tượng phát sinh sự kiện.

---

## SaaS B2B cảnh báo sự cố hệ thống (incident alerting)

**Repository:** `notification-fanout-b2b-saas-incident-alerting`

**Hệ thống:** Một nền tảng giám sát hệ thống (monitoring/observability) cho khách hàng doanh nghiệp, cần cảnh báo khi phát hiện sự cố (alert) trên hệ thống họ đang theo dõi.

**Vai trò của flow:** Fan-out cảnh báo sự cố tới đúng người chịu trách nhiệm (on-call) qua nhiều kênh với cơ chế leo thang (escalation) nếu không có phản hồi.

**Yêu cầu cụ thể:**
- Alert phải được gửi tới người on-call hiện tại theo lịch trực (rotation schedule) đã cấu hình, không gửi cố định tới một người bất kể ai đang trực.
- Nếu người on-call không xác nhận (acknowledge) trong khoảng thời gian quy định, hệ thống phải tự động leo thang gửi tới người tiếp theo trong chuỗi escalation, và tiếp tục leo thang cho đến khi có người xác nhận.
- Alert mức độ nghiêm trọng khác nhau phải dùng kênh khác nhau (ví dụ mức critical dùng gọi điện tự động + SMS, mức warning chỉ cần email/Slack) theo cấu hình mỗi khách hàng.
- Phải chống được "alert storm" — khi nhiều alert liên quan phát sinh cùng lúc từ một nguyên nhân gốc, cần gộp nhóm thành một thông báo tổng hợp thay vì làm ngập kênh liên lạc của người on-call.
- Khi sự cố được đánh dấu đã khắc phục (resolved), phải gửi thông báo resolved tới đúng những người/kênh đã nhận alert gốc, và dừng mọi escalation đang chờ xử lý liên quan đến alert đó.

---

## Ứng dụng gọi xe/giao hàng thông báo ghép nối real-time

**Repository:** `notification-fanout-ride-hailing-matching-realtime`

**Hệ thống:** Một app gọi xe/giao hàng cần thông báo real-time cho khách và cho tài xế trong suốt quá trình ghép nối và di chuyển chuyến đi.

**Vai trò của flow:** Fan-out thông báo tức thời (đã tìm được tài xế, tài xế đang đến, đã đến điểm đón) tới cả hai bên khách và tài xế với độ trễ tối thiểu.

**Yêu cầu cụ thể:**
- Thông báo trong luồng một chuyến đi phải đến đúng thứ tự thời gian thực tế xảy ra (không thể để thông báo "tài xế đã đến" tới trước thông báo "đã ghép nối tài xế") dù đi qua các kênh có độ trễ khác nhau.
- Nếu app của khách/tài xế đang ở background hoặc mất kết nối tạm thời, thông báo quan trọng (đã ghép nối, hủy chuyến) phải fallback qua SMS để đảm bảo tới được, còn thông báo cập nhật vị trí liên tục thì không cần fallback (chấp nhận mất vài update).
- Khi chuyến đi bị hủy bởi một trong hai bên, thông báo hủy phải tới bên còn lại gần như ngay lập tức, có độ trễ chấp nhận được thấp hơn hẳn so với các thông báo thông thường khác trong hệ thống.
- Hệ thống phải xử lý được lượng thông báo tăng vọt tại giờ cao điểm (rush hour) ở một khu vực địa lý cụ thể mà không làm chậm thông báo ở các khu vực khác.
- Log lại đầy đủ chuỗi thông báo của mỗi chuyến đi (thời điểm gửi, kênh, trạng thái) để phục vụ giải quyết tranh chấp khi khách/tài xế khiếu nại không nhận được thông báo đúng lúc.

---

## Nền tảng marketing gửi chiến dịch broadcast tới hàng triệu người dùng

**Repository:** `notification-fanout-marketing-mass-broadcast`

**Hệ thống:** Một nền tảng SaaS cho phép doanh nghiệp gửi chiến dịch email/SMS marketing tới danh sách khách hàng lớn của họ (lên tới hàng triệu người nhận trong một chiến dịch).

**Vai trò của flow:** Fan-out một chiến dịch tới toàn bộ danh sách người nhận theo kênh đã chọn, với tốc độ gửi được kiểm soát để không vi phạm chính sách nhà cung cấp và không làm phiền người dùng.

**Yêu cầu cụ thể:**
- Việc gửi phải được throttle theo tốc độ phù hợp với hạn mức của từng provider email/SMS (tránh bị đánh dấu spam hoặc bị nhà cung cấp khóa tài khoản gửi), có thể điều chỉnh tốc độ gửi theo thời gian thực nếu provider báo lỗi rate-limit.
- Trước khi fan-out toàn bộ, phải loại bỏ khỏi danh sách gửi những người đã unsubscribe hoặc đã opt-out kênh đó trước đây — việc lọc phải chạy ngay tại thời điểm gửi, không dùng danh sách đã lọc từ nhiều ngày trước có thể đã lỗi thời.
- Nếu chiến dịch bị dừng giữa chừng (doanh nghiệp phát hiện lỗi nội dung sau khi đã gửi một phần), phải dừng được phần chưa gửi ngay lập tức và báo cáo chính xác số người đã/chưa nhận được.
- Phải theo dõi và báo cáo cho doanh nghiệp các chỉ số gửi theo thời gian thực (đã gửi, thất bại, bounce) trong khi chiến dịch đang chạy, không chỉ có báo cáo tổng kết sau khi hoàn tất toàn bộ.
- Cho phép chia nhỏ chiến dịch theo từng nhóm nhỏ (canary/staged rollout) để kiểm tra tỷ lệ lỗi/bounce trước khi fan-out ra toàn bộ danh sách lớn, tránh gửi sai tới toàn bộ triệu người nhận cùng lúc.
