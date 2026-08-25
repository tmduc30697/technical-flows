# Backpressure & flow control trong streaming pipeline — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần điều tiết luồng dữ liệu — pipeline clickstream phân tích hành vi, chat livestream viral, ingest telemetry IoT, tổng hợp log incident, và xử lý event đơn hàng flash sale — nhằm luyện chiến lược buffer/drop, điều tiết producer, và cách ly consumer chậm trong web app thực tế.

---

## Pipeline thu thập clickstream cho phân tích hành vi người dùng

**Repository:** `backpressure-clickstream-analytics-pipeline`

**Hệ thống:** Trang web lớn thu thập event clickstream (click, scroll, view) từ hàng triệu user, đẩy qua pipeline streaming để xử lý real-time.

**Vai trò của flow:** Backpressure đảm bảo khi consumer (bộ xử lý) chậm hơn producer (lượng event dội vào), hệ thống không bị OOM hay crash mà điều tiết tốc độ nhận/xử lý hợp lý.

**Yêu cầu cụ thể:**
- Khi consumer xử lý chậm lại (do downstream DB nghẽn), producer phải được thông báo/điều tiết để giảm tốc độ gửi, không đơn giản buffer vô hạn trong memory gây OOM.
- Định nghĩa rõ chiến lược khi buffer đầy: drop event mới nhất, drop event cũ nhất, hay block producer — và giải thích lựa chọn phù hợp với việc "một số event bị mất không nghiêm trọng bằng sập cả pipeline".
- Có cơ chế đo lag cụ thể (số event đang chờ xử lý, thời gian event cũ nhất chưa xử lý) và expose làm metric để alerting khi lag vượt ngưỡng.
- Test khả năng phục hồi khi consumer bị chậm trong 10 phút rồi tăng tốc trở lại — pipeline phải tự "rút" hết backlog mà không cần can thiệp thủ công.
- Đảm bảo backpressure ở một consumer chậm không làm ảnh hưởng tới các consumer khác đang xử lý tốt (isolation giữa các luồng xử lý song song).

---

## Pipeline tin nhắn chat trong livestream có hàng trăm nghìn viewer

**Repository:** `backpressure-livestream-chat-pipeline`

**Hệ thống:** Nền tảng livestream nơi viewer gửi chat/comment với tốc độ rất cao trong các khoảnh khắc "viral" (drop hàng nghìn message/giây).

**Vai trò của flow:** Điều tiết luồng tin nhắn để hệ thống render/broadcast không bị nghẽn khi lượng chat tăng đột biến, ưu tiên trải nghiệm mượt hơn là hiển thị 100% mọi tin nhắn.

**Yêu cầu cụ thể:**
- Khi tốc độ gửi chat vượt khả năng broadcast tới toàn bộ viewer, hệ thống phải sample/giảm tải theo chiến lược rõ ràng (ví dụ chỉ broadcast X tin nhắn/giây, ưu tiên tin nhắn từ mod/donate) thay vì cố gửi hết và làm trễ toàn bộ.
- Viewer client phải nhận được tín hiệu khi có tin nhắn bị bỏ qua (ví dụ đếm "X tin nhắn khác") thay vì im lặng làm mất mà không ai biết.
- Producer (client gửi chat) phải bị rate-limit ngay từ đầu vào (per-user) để tránh 1 user spam làm nghẽn pipeline chung của cả stream.
- Có cơ chế "priority lane" riêng cho message quan trọng (donation alert, mod action) không bị ảnh hưởng bởi backpressure của luồng chat thường.
- Đo lường: end-to-end latency từ lúc viewer gửi chat tới lúc mọi người khác thấy, đảm bảo dưới ngưỡng (ví dụ 2 giây) ngay cả lúc traffic đỉnh.

---

## Pipeline ingest dữ liệu telemetry từ hàng triệu thiết bị IoT

**Repository:** `backpressure-iot-telemetry-ingest`

**Hệ thống:** Nền tảng IoT nhận dữ liệu cảm biến liên tục từ số lượng thiết bị lớn, tốc độ gửi có thể tăng vọt (ví dụ khi mạng lưới thiết bị mất kết nối rồi cùng gửi lại dữ liệu tồn đọng).

**Vai trò của flow:** Backpressure bảo vệ tầng xử lý/storage phía sau khỏi bị dội tải (thundering herd) khi thiết bị đồng loạt gửi lại dữ liệu backlog sau khi mất kết nối.

**Yêu cầu cụ thể:**
- Thiết bị phải tuân theo tín hiệu điều tiết từ server (ví dụ HTTP 429/MQTT flow control) và tự giãn tốc độ gửi lại dữ liệu tồn đọng, không đẩy hết trong vài giây.
- Server phải phân biệt được dữ liệu "mới" (gần real-time, ưu tiên xử lý ngay) và dữ liệu "backlog" (đến muộn, có thể xử lý batch chậm hơn) để không để backlog chiếm hết tài nguyên xử lý real-time.
- Có giới hạn buffer rõ ràng ở tầng ingest (queue) và hành vi cụ thể khi đầy (reject với lỗi rõ ràng để thiết bị tự retry sau, theo exponential backoff).
- Đo lường khả năng chịu tải đỉnh (burst capacity) cụ thể — ví dụ hệ thống chịu được gấp bao nhiêu lần tải trung bình trong bao lâu trước khi phải áp backpressure.
- Giám sát theo từng nhóm thiết bị/khu vực để cách ly (isolate) khi một nhóm thiết bị lỗi gửi dữ liệu dồn dập, tránh ảnh hưởng tới toàn hệ thống.

---

## Pipeline tổng hợp log từ hàng trăm service (log aggregation)

**Repository:** `backpressure-log-aggregation-pipeline`

**Hệ thống:** Hạ tầng nội bộ thu thập log từ hàng trăm service để phục vụ debug/observability, lượng log tăng vọt khi có incident (service log lỗi liên tục).

**Vai trò của flow:** Backpressure đảm bảo khi một service bị lỗi và log dồn dập (log storm), pipeline aggregation không bị sập kéo theo ảnh hưởng tới log của các service khác đang bình thường.

**Yêu cầu cụ thể:**
- Phải có per-service rate limit ở tầng thu thập log để một service bị lỗi log storm không chiếm hết băng thông/tài nguyên xử lý log của service khác.
- Khi vượt rate limit, log bị drop phải được đếm và báo lại cho service nguồn biết (ví dụ metric "log dropped count") để team đó biết log của họ không đầy đủ trong lúc incident.
- Consumer phía sau (indexing engine) chậm phải khiến hệ thống tạm giữ log ở buffer trung gian (disk-backed queue) có giới hạn dung lượng, không giữ toàn bộ trong RAM.
- Ưu tiên log mức ERROR/CRITICAL hơn log DEBUG/INFO khi phải áp dụng backpressure và buộc phải drop một phần.
- Đo lường: thời gian từ lúc log được sinh ra tới lúc searchable trong hệ thống (ingestion lag) theo điều kiện tải bình thường và tải đỉnh lúc incident.

---

## Pipeline xử lý event đơn hàng lúc flash sale

**Repository:** `backpressure-flash-sale-order-events`

**Hệ thống:** E-commerce có pipeline xử lý event đơn hàng (tạo đơn, trừ kho, gửi email) chạy qua message queue, lượng event tăng đột biến trong giờ flash sale.

**Vai trò của flow:** Backpressure ngăn consumer (worker xử lý đơn hàng) bị quá tải khi số đơn dội vào vượt xa khả năng xử lý bình thường, tránh timeout/crash làm mất/trễ đơn hàng.

**Yêu cầu cụ thể:**
- Producer (API nhận đơn) phải biết được tín hiệu queue đang gần đầy để tự điều chỉnh (ví dụ trả về "đang xử lý, vui lòng chờ" cho user) thay vì tiếp tục nhận đơn vô hạn rồi mất đơn.
- Consumer worker phải scale động theo độ dài queue (autoscaling dựa trên lag) nhưng có giới hạn trên (max worker) để không làm sập database phía sau do quá nhiều connection đồng thời.
- Đơn hàng không được bị mất dù bị giữ trong queue lâu hơn bình thường lúc traffic đỉnh — nêu rõ giới hạn thời gian tối đa một đơn có thể chờ trong queue trước khi coi là cần escalate/alert.
- Có cơ chế phân loại ưu tiên (ví dụ đơn giá trị cao hoặc khách VIP xử lý trước) khi queue bị nghẽn, thay vì strict FIFO làm chậm tất cả như nhau.
- Đo lường và báo cáo: throughput xử lý đơn/giây tối đa hệ thống đạt được, và độ trễ trung bình/p99 xử lý đơn trong giờ flash sale so với giờ bình thường.
