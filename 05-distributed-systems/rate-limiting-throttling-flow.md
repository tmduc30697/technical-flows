# Rate limiting/throttling flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần kiểm soát tốc độ request — API gateway public đa khách hàng, chống brute-force login, tôn trọng rate limit đối tác bên ngoài, chống spam chat, và throttling checkout flash sale — nhằm luyện thuật toán rate limit cụ thể, xử lý counter phân tán nhiều instance, và cân bằng giữa bảo vệ hệ thống với trải nghiệm người dùng trong web app thực tế.

---

## API gateway public cho SaaS đa khách hàng (per-API-key rate limit)

**Repository:** `rate-limiting-saas-api-gateway-per-key`

**Hệ thống:** SaaS cung cấp public API cho khách hàng dùng qua API key, cần giới hạn số request theo từng gói dịch vụ (free/pro/enterprise).

**Vai trò của flow:** Rate limiting đảm bảo công bằng tài nguyên giữa các khách hàng, tránh một khách hàng dùng quá nhiều làm ảnh hưởng khách khác, đồng thời đúng với gói dịch vụ họ trả tiền.

**Yêu cầu cụ thể:**
- Mỗi API key có giới hạn request/giây và request/tháng riêng theo gói, và response khi vượt limit phải trả đúng chuẩn (HTTP 429 kèm header Retry-After, X-RateLimit-Remaining).
- Dùng thuật toán rate limit cụ thể (ví dụ token bucket hoặc sliding window log) để cho phép burst ngắn hợp lý mà không cho phép lạm dụng liên tục vượt giới hạn trung bình.
- Rate limit counter phải chính xác trong môi trường nhiều instance API gateway chạy song song (không được để mỗi instance tự đếm riêng dẫn tới tổng vượt xa giới hạn thực).
- Khi Redis/coordinator lưu counter bị chậm/down tạm thời, phải có chiến lược rõ ràng (fail-open cho phép qua tạm hay fail-closed chặn tạm) và giải thích lý do chọn theo bối cảnh SaaS đó.
- Có dashboard cho khách hàng tự xem usage hiện tại so với limit gói của họ, để họ tự chủ động nâng cấp gói trước khi bị chặn.

---

## Chống brute-force ở endpoint đăng nhập

**Repository:** `rate-limiting-login-brute-force`

**Hệ thống:** Một web app cần bảo vệ endpoint login khỏi tấn công dò mật khẩu (brute force/credential stuffing).

**Vai trò của flow:** Rate limiting theo nhiều chiều (theo IP, theo username, theo device) để làm chậm/ngăn tấn công dò mật khẩu mà không gây khó cho user hợp lệ gõ sai vài lần.

**Yêu cầu cụ thể:**
- Giới hạn số lần login sai theo username cụ thể (ví dụ 5 lần/15 phút) độc lập với giới hạn theo IP (ví dụ 20 lần/15 phút toàn bộ username khác nhau từ 1 IP) để bắt được cả 2 kiểu tấn công (dò 1 tài khoản nhiều lần, và dò nhiều tài khoản từ 1 nguồn).
- Sau khi vượt ngưỡng, phải có cơ chế tăng dần độ khó (progressive delay/CAPTCHA) trước khi chặn hoàn toàn, tránh chặn cứng ngay gây trải nghiệm xấu cho user thật chỉ gõ nhầm vài lần.
- Rate limit không được dựa hoàn toàn vào IP đơn lẻ vì tấn công thực tế thường dùng nhiều IP (botnet) — phải kết hợp thêm fingerprint thiết bị hoặc pattern hành vi bất thường.
- Login thành công phải reset counter rate limit của username đó, không giữ user hợp lệ trong trạng thái gần bị khóa do vài lần gõ sai trước đó.
- Log và alert khi phát hiện pattern tấn công rõ ràng (số lượng lớn login fail trong thời gian ngắn từ nhiều IP nhắm vào nhiều username) để team security theo dõi real-time.

---

## Throttling request checkout trong flash sale để bảo vệ hệ thống backend

**Repository:** `rate-limiting-flash-sale-checkout-throttle`

**Hệ thống:** E-commerce cần throttle lượng request checkout đồng thời trong giờ flash sale để tránh backend (DB, payment) bị quá tải sập hoàn toàn.

**Vai trò của flow:** Giới hạn số lượng request được xử lý đồng thời ở tầng vào (ví dụ qua hàng đợi ảo/virtual waiting room) để bảo vệ hệ thống, đổi lại một số user phải chờ theo lượt.

**Yêu cầu cụ thể:**
- Khi lượng request checkout vượt khả năng xử lý an toàn của backend, hệ thống phải chuyển user vào "phòng chờ" (queue) hiển thị vị trí/thời gian ước tính, không để request dồn thẳng vào backend gây sập.
- Thứ tự vào phòng chờ phải công bằng (FIFO theo thời điểm request tới) trừ khi có chính sách ưu tiên rõ ràng được công bố trước (ví dụ khách VIP), tránh cảm giác không công bằng cho user thường.
- Hệ thống throttle phải điều chỉnh động số lượng cho qua dựa trên tình trạng thực tế của backend (ví dụ giảm số cho qua nếu latency DB tăng cao), không dùng số cố định tĩnh không phản ánh tải thật.
- Đảm bảo user đã vào được luồng xử lý (qua khỏi hàng chờ) không bị rate limit lần nữa giữa đường gây trải nghiệm xấu (đã chờ xong lại bị chặn tiếp).
- Đo lường và báo cáo: throughput checkout thành công/giây tối đa hệ thống đạt được an toàn, và tỉ lệ user phải vào hàng chờ so với tổng traffic trong giờ flash sale, dùng để lên kế hoạch capacity cho các sự kiện sau.

---

## Tôn trọng rate limit của đối tác bên ngoài khi gửi SMS/OTP

**Repository:** `rate-limiting-external-partner-sms-gateway`

**Hệ thống:** Nhiều service nội bộ (OTP đăng nhập, thông báo đơn hàng, marketing) cùng gọi tới một đối tác SMS gateway bên ngoài có giới hạn rate nghiêm ngặt (request/giây) áp dụng chung cho toàn bộ tài khoản đối tác.

**Vai trò của flow:** Throttling ở tầng gọi ra (outbound) đảm bảo tổng lưu lượng từ tất cả service nội bộ cộng lại không vượt giới hạn của đối tác, tránh bị đối tác chặn/phạt hoặc rớt message.

**Yêu cầu cụ thể:**
- Giới hạn của đối tác là giới hạn tổng (global) áp dụng chung cho toàn công ty, không phải cho riêng từng service — nếu mỗi service tự throttle độc lập theo giả định "được chia đều" thì tổng thực tế vẫn có thể vượt ngưỡng khi nhiều service cùng gửi cao điểm cùng lúc; cần một điểm throttle tập trung (hoặc counter chia sẻ) mà mọi service phải đi qua.
- Khi tổng lưu lượng vượt giới hạn đối tác cho phép tại một thời điểm, phải có cơ chế ưu tiên rõ ràng giữa các loại tin nhắn (OTP đăng nhập cần độ trễ thấp, ưu tiên hơn SMS marketing có thể trì hoãn vài phút), tránh để OTP bị xếp hàng chung với marketing gây trải nghiệm đăng nhập tệ.
- Khi bị đối tác trả về lỗi rate limit dù đã throttle, phải có backoff tăng dần và không được retry ngay lập tức theo kiểu spam lại, vì retry dồn dập có thể khiến đối tác giảm hạn mức hoặc tạm khóa tài khoản của cả công ty.
- Request bị throttle chờ tới lượt gửi phải có hàng đợi có TTL/hết hạn hợp lý (ví dụ OTP hết hạn nếu chờ quá vài chục giây thì bỏ, báo lỗi cho user thử lại) thay vì giữ trong queue vô thời hạn khiến user chờ OTP không tới nhưng cũng không được báo lỗi.
- Phải giám sát được usage thực tế so với hạn mức đối tác theo thời gian gần thực, cảnh báo sớm khi xu hướng traffic đang tiến gần ngưỡng (ví dụ do một service lỗi gửi lặp) trước khi bị đối tác chặn hoàn toàn ảnh hưởng tất cả service khác dùng chung gateway.

---

## Chống spam tin nhắn trong app chat/group

**Repository:** `rate-limiting-chat-spam-prevention`

**Hệ thống:** Ứng dụng chat/group cho phép user gửi tin nhắn tự do, cần chặn bot/spam gửi hàng loạt tin nhắn mà không làm phiền user thật đang gõ nhanh trong cuộc trò chuyện sôi nổi.

**Vai trò của flow:** Rate limiting theo user/theo cuộc trò chuyện giới hạn tần suất gửi tin để phát hiện và chặn hành vi spam, đồng thời đủ khoan dung cho tốc độ gõ/gửi tự nhiên của người dùng bình thường.

**Yêu cầu cụ thể:**
- Ngưỡng rate limit cứng (ví dụ giới hạn tin/giây cố định) dễ chặn nhầm user thật đang nhắn tin dồn dập trong lúc trò chuyện sôi nổi — cần thuật toán cho phép burst ngắn tự nhiên (vài tin liên tiếp trong 1-2 giây) nhưng vẫn bắt được pattern đều đặn máy móc kéo dài của bot.
- Phân biệt rate limit theo từng cuộc trò chuyện một-một và theo group lớn (hàng trăm/nghìn thành viên) — cùng một user gửi nhanh trong group đông có thể là bình thường (đang thảo luận sôi nổi), trong khi cùng tần suất đó nhắm vào nhiều user khác nhau (spam hàng loạt) phải bị đánh dấu khác.
- Khi user bị giới hạn do gửi quá nhanh, phản hồi (ẩn tạm nút gửi, thông báo chờ vài giây) không được làm mất nội dung tin nhắn đang gõ dở — user không được yêu cầu gõ lại từ đầu chỉ vì bị rate limit tạm thời.
- Phát hiện spam không chỉ dựa vào tần suất mà còn pattern nội dung lặp lại (cùng một tin gửi liên tục tới nhiều group/user khác nhau trong thời gian ngắn) — một tài khoản gửi đúng nhịp một tin/giây liên tục hàng giờ tới nhiều đối tượng khác nhau là dấu hiệu bot dù tần suất từng tin không vượt ngưỡng cứng.
- Rate limit phải áp dụng được động theo uy tín tài khoản (tài khoản mới tạo/chưa xác thực bị giới hạn chặt hơn tài khoản lâu năm có lịch sử hành vi tốt), tránh chặn quá nghiêm với user thật lâu năm trong khi vẫn kiểm soát chặt tài khoản mới dễ bị lợi dụng để spam.
