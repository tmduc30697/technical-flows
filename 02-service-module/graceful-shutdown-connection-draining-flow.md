# Graceful shutdown & connection draining flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (e-commerce checkout, streaming video, chat real-time, background worker, API gateway) để luyện việc tắt service một cách an toàn mà không làm rơi request hoặc mất dữ liệu đang xử lý.

---

## Checkout service của sàn e-commerce trong lúc deploy

**Repository:** `graceful-shutdown-ecommerce-checkout-deploy`

**Hệ thống:** Một service xử lý checkout (tạo order, trừ tồn kho, gọi cổng thanh toán) cho sàn e-commerce, được deploy nhiều lần mỗi ngày.

**Vai trò của flow:** Khi rolling deploy loại bỏ instance cũ, instance đó phải hoàn tất các giao dịch checkout đang xử lý trước khi tắt, không được cắt ngang giữa lúc đang trừ tồn kho hoặc gọi cổng thanh toán.

**Yêu cầu cụ thể:**
- Khi nhận SIGTERM, instance phải ngay lập tức báo readiness = false (để load balancer ngừng gửi request mới) nhưng vẫn tiếp tục xử lý các request đang chạy cho tới khi xong hoặc hết grace period.
- Định nghĩa rõ grace period tối đa (ví dụ 30 giây) — nếu một giao dịch checkout chưa xong khi hết grace period, phải có chiến lược rõ ràng (log lại state, không để order ở trạng thái "nửa vời" — ví dụ đã trừ tồn kho nhưng chưa tạo order) thay vì kill cứng.
- Xử lý trường hợp instance đang giữ một distributed lock (ví dụ lock tồn kho sản phẩm) khi bị shutdown — phải release lock hoặc để lock tự hết TTL, không được để lock treo vĩnh viễn làm nghẽn các checkout khác.
- Đảm bảo trong lúc drain, instance không nhận thêm request mới (health check đã báo not-ready) nhưng vẫn phải trả lời được health check probe để không bị hạ tầng kill ngay lập tức trước khi drain xong.
- Viết test mô phỏng gửi 100 request checkout đồng thời rồi trigger shutdown giữa chừng, verify không có request nào bị mất kết nối nửa đường (mỗi request phải nhận được response, kể cả response lỗi rõ ràng, không bao giờ là timeout im lặng).

---

## Chat server real-time cho ứng dụng nhắn tin

**Repository:** `graceful-shutdown-chat-server-realtime`

**Hệ thống:** Một backend chat real-time giữ kết nối WebSocket cho hàng chục nghìn user, mỗi kết nối gắn với một session hội thoại.

**Vai trò của flow:** Đảm bảo khi một chat server instance cần tắt (maintenance/deploy), user không bị mất tin nhắn đang gửi và được chuyển êm sang instance khác.

**Yêu cầu cụ thể:**
- Tin nhắn đang trong hàng đợi gửi (chưa được ACK bởi client) tại thời điểm shutdown phải được đảm bảo không mất — chuyển giao lại cho hàng đợi bền (message queue/DB) để instance khác tiếp tục gửi, không được chỉ giữ trong memory rồi mất khi tắt.
- Khi ngắt kết nối WebSocket để drain, phải gửi kèm mã lý do (close code) riêng cho "server draining" khác với "lỗi bất thường", để client phân biệt và tự reconnect ngay, không hiển thị lỗi gây hoang mang cho người dùng.
- Đảm bảo tính idempotent: nếu một tin nhắn đã được gửi thành công tới client trước khi server bị ngắt giữa lúc gửi ACK, khi reconnect vào instance mới, tin nhắn không được gửi lại trùng lặp.
- Xử lý trường hợp user đang gõ (typing indicator) hoặc đang trong cuộc gọi voice/video khi server drain — trạng thái này phải được chuyển giao hoặc kết thúc rõ ràng, không để lại trạng thái "đang gõ" treo vĩnh viễn ở phía người nhận.
- Thiết kế cơ chế rolling restart cho toàn cluster chat server sao cho tại mọi thời điểm, tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online, tránh gây spike tải lên hệ thống presence/auth khi hàng loạt client reconnect cùng lúc.

---

## API Gateway thực hiện rolling restart toàn cụm backend

**Repository:** `graceful-shutdown-api-gateway-rolling-restart`

**Hệ thống:** Một API Gateway đứng trước nhiều microservice của một nền tảng SaaS, cần rolling restart lần lượt các service trong giờ ít traffic mà không gây downtime cho khách hàng.

**Vai trò của flow:** Gateway phải phối hợp với connection draining của từng service để đảm bảo không có request nào bị route tới instance đang trong quá trình tắt.

**Yêu cầu cụ thể:**
- Gateway phải nhận được tín hiệu "instance X sắp shutdown" trước khi instance đó thực sự ngừng nhận connection (không dựa hoàn toàn vào health check polling định kỳ vốn có độ trễ phát hiện), để loại instance ra khỏi pool routing ngay lập tức.
- Với các kết nối HTTP keep-alive đang mở tới instance sắp tắt, gateway phải đảm bảo request tiếp theo trên kết nối đó (nếu có) được route sang instance khác, không tái sử dụng kết nối cũ tới instance đang chết.
- Trong lúc một service đang rolling restart (một số instance đã restart, một số chưa), phải xử lý được trường hợp version mới và version cũ trả về response có schema khác nhau nhẹ — gateway hoặc client không được crash vì field mới thiếu/thừa.
- Đặt cơ chế circuit breaker tạm thời tăng ngưỡng chịu lỗi (error budget) trong lúc rolling restart đang diễn ra, để tránh gateway hoảng loạn ngắt toàn bộ traffic chỉ vì vài request timeout do đúng lúc instance đang drain.
- Viết test toàn trình: rolling restart lần lượt 5 instance của một service trong lúc có traffic liên tục chạy nền, đo tỷ lệ lỗi/timeout trong toàn bộ quá trình phải bằng 0 hoặc dưới một ngưỡng SLA đã định nghĩa trước.
