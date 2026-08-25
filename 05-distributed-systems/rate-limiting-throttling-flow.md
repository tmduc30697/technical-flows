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

## Client tôn trọng rate limit của đối tác khi gọi third-party API

**Repository:** `rate-limiting-client-third-party-api`

**Hệ thống:** Một hệ thống backend gọi API của đối tác bên ngoài (ví dụ API vận chuyển, API thanh toán) có giới hạn rate limit nghiêm ngặt từ phía đối tác.

**Vai trò của flow:** Chủ động throttle traffic đi ra (outbound) để không vượt giới hạn đối tác đặt ra, tránh bị đối tác tạm khóa truy cập hoặc phạt.

**Yêu cầu cụ thể:**
- Hệ thống phải tự giới hạn tốc độ gọi API đối tác đúng theo hạn mức họ công bố (ví dụ 100 request/giây), có buffer an toàn (ví dụ chỉ dùng 90% hạn mức) để tránh sát ngưỡng gây lỗi do đo đếm lệch nhau.
- Khi nhiều service nội bộ cùng gọi tới API của đối tác đó, phải có một điểm điều tiết tập trung (throttle chung) để tổng traffic không vượt hạn mức, không để mỗi service tự giới hạn riêng dẫn tới tổng vượt quá.
- Khi bị đối tác trả 429, phải đọc và tuân thủ đúng thời gian chờ họ yêu cầu (Retry-After), và có cơ chế giảm tốc độ gọi tạm thời (backoff) trong một khoảng thời gian sau đó để tránh lặp lại.
- Có hàng đợi ưu tiên cho các request quan trọng (ví dụ tạo đơn ship thật) hơn các request ít khẩn cấp (ví dụ đồng bộ dữ liệu nền) khi gần chạm hạn mức của đối tác.
- Giám sát usage thực tế so với hạn mức đối tác theo thời gian thực, cảnh báo sớm khi gần chạm ngưỡng để chủ động liên hệ đối tác tăng hạn mức nếu cần trước khi bị chặn.

---

## Giới hạn tốc độ gửi tin nhắn trong app chat để chống spam

**Repository:** `rate-limiting-chat-message-anti-spam`

**Hệ thống:** App chat/messenger cần giới hạn tốc độ user gửi tin nhắn để chống spam, đặc biệt trong group chat lớn.

**Vai trò của flow:** Rate limit theo user và theo conversation để giữ trải nghiệm chat lành mạnh, phân biệt giữa user gõ nhanh tự nhiên và hành vi spam tự động.

**Yêu cầu cụ thể:**
- Giới hạn số tin nhắn/giây theo user phải đủ rộng để không chặn nhầm người gõ nhanh bình thường (ví dụ cho phép burst ngắn), nhưng chặn được bot gửi hàng loạt liên tục.
- Với group chat lớn, cần thêm giới hạn tổng số tin nhắn/giây của cả group (không chỉ theo từng user) để tránh nhiều user cùng gửi dồn dập làm nghẽn hiển thị cho tất cả thành viên.
- User bị rate limit phải nhận phản hồi rõ ràng ngay trên UI (ví dụ "gửi quá nhanh, vui lòng chờ") thay vì tin nhắn bị âm thầm mất không rõ lý do.
- Phân biệt được rate limit áp dụng cho tin nhắn văn bản thường và các hành động khác (gửi file, tạo poll) — có thể cần ngưỡng riêng vì mức độ ảnh hưởng tài nguyên khác nhau.
- Có cơ chế phát hiện hành vi spam nâng cao hơn rate limit đơn giản (ví dụ gửi cùng nội dung lặp lại liên tục) để tự động cảnh cáo/tạm khóa tài khoản nghi ngờ là bot, không chỉ dựa vào tốc độ.

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
