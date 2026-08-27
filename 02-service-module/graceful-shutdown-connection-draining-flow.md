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

---

## Server streaming video cần shutdown mà không cắt ngang luồng đang phát

**Repository:** `graceful-shutdown-video-streaming-server`

**Hệ thống:** Một server phát video theo yêu cầu (streaming) cho hàng nghìn client cùng lúc, mỗi client giữ một kết nối HTTP dài (long-lived, có thể kéo dài từ vài phút tới vài giờ tùy độ dài video) để nhận dữ liệu video liên tục.

**Vai trò của flow:** Khác với request ngắn thông thường, draining ở đây không thể chỉ chờ vài chục giây rồi cắt — phải xử lý đúng cách với các kết nối có thể còn kéo dài rất lâu, mà không làm gián đoạn trải nghiệm xem của người dùng đang giữa chừng video.

**Yêu cầu cụ thể:**
- Khi nhận tín hiệu shutdown, instance phải phân loại các kết nối streaming đang mở theo thời lượng còn lại ước tính — với các luồng sắp kết thúc tự nhiên trong vài phút, để chúng tự hoàn tất; với các luồng còn rất dài, phải có chiến lược chuyển giao (ví dụ báo client tự động reconnect sang instance khác tại đúng vị trí đang xem) thay vì chờ vô thời hạn.
- Grace period cho streaming không thể dùng một con số cố định ngắn như service API thông thường — phải định nghĩa theo phân vị của thời lượng video thực tế (ví dụ đủ để 95% luồng đang phát tự kết thúc), và với phần còn lại chấp nhận chủ động ngắt có kiểm soát kèm tín hiệu để client resume đúng vị trí, không để phát hiện lỗi im lặng.
- Đảm bảo khi một kết nối streaming bị buộc ngắt do hết grace period, response phải trả về theo cách client player hiểu được là "cần reconnect" (ví dụ đóng luồng kèm mã trạng thái/tín hiệu rõ ràng), chứ không phải một kết nối bị treo hoặc timeout im lặng khiến player hiển thị lỗi phát khó chịu cho người xem.
- Xử lý trường hợp nhiều instance cùng shutdown gần như đồng thời (ví dụ do autoscale scale-in loạt lớn cùng lúc với deploy) — tránh dồn toàn bộ client đang xem bị buộc reconnect cùng một thời điểm gây spike tải đột biến lên các instance còn lại hoặc lên CDN edge phía trước.
- Theo dõi và cảnh báo riêng tỷ lệ luồng bị ngắt giữa chừng do shutdown (khác với ngắt do người dùng chủ động dừng xem), để phân biệt được mức độ ảnh hưởng thực sự của quá trình draining lên trải nghiệm người dùng, không gộp chung vào số liệu drop-off thông thường.

---

## Background worker xử lý job dài đang consume message từ queue

**Repository:** `graceful-shutdown-background-worker-queue`

**Hệ thống:** Một cụm worker consume message từ hàng đợi để xử lý các job chạy lâu (export file dữ liệu lớn, gửi email hàng loạt, xử lý batch report), mỗi job có thể mất từ vài chục giây tới vài phút để hoàn tất.

**Vai trò của flow:** Khi worker bị shutdown (deploy, autoscale scale-in), phải đảm bảo message đang xử lý dở không bị mất và không bị ack sai thời điểm — ack quá sớm làm mất dữ liệu nếu worker chết giữa chừng, ack quá muộn hoặc không ack làm message bị xử lý lặp lại không cần thiết.

**Yêu cầu cụ thể:**
- Khi nhận tín hiệu shutdown, worker phải ngay lập tức ngừng nhận message mới từ queue (ngừng poll/prefetch) nhưng vẫn tiếp tục xử lý cho tới khi hoàn tất các message đã nhận trước đó, tránh tình huống nhận thêm việc mới ngay trước lúc tắt rồi phải bỏ dở.
- Message chỉ được ack (xác nhận đã xử lý xong, xóa khỏi queue) sau khi job đã hoàn tất toàn bộ và kết quả đã được ghi bền vững — nếu worker bị kill cứng giữa lúc xử lý (hết grace period), message chưa ack phải tự động được đưa trở lại queue để worker khác xử lý lại, không được để mất.
- Với các job có thể để lại tác động một phần khi bị ngắt giữa chừng (ví dụ export file đã ghi được nửa file, gửi email hàng loạt đã gửi được một số người trong danh sách), phải thiết kế idempotent hoặc có checkpoint để lần xử lý lại sau không lặp lại phần đã hoàn tất (không gửi trùng email cho người đã nhận, không tạo file export bị lỗi nửa vời).
- Định nghĩa grace period đủ dài để job dài nhất trong hệ thống có cơ hội hoàn tất bình thường (không cắt ngang job đang chạy gần xong), nhưng vẫn có ngưỡng tối đa hợp lý để không làm chậm quá trình deploy/scale toàn cụm — cân nhắc cho phép job vượt quá một độ dài nhất định tự chủ động lưu checkpoint giữa chừng thay vì cố chờ nó chạy xong trong grace period.
- Xử lý trường hợp toàn bộ worker trong cụm đều nhận tín hiệu shutdown gần như đồng thời (ví dụ rolling deploy toàn cụm) — phải đảm bảo còn ít nhất một số worker tiếp tục nhận message mới trong lúc số còn lại đang drain, tránh để hàng đợi bị dồn ứ không ai xử lý trong lúc chuyển giao.
