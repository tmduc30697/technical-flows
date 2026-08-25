# Service discovery & health check flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (nền tảng microservices SaaS, hệ thống IoT, hạ tầng multi-region, service mesh nội bộ) để luyện việc đăng ký, phát hiện và theo dõi sức khỏe service một cách đúng đắn.

---

## Service registry cho nền tảng SaaS microservices

**Repository:** `service-discovery-saas-microservices-registry`

**Hệ thống:** Một nền tảng SaaS gồm khoảng 10 microservice (auth, billing, notification, ...) giao tiếp qua HTTP nội bộ, cần một registry trung tâm để các service tìm nhau.

**Vai trò của flow:** Mỗi service tự đăng ký khi khởi động và bị loại khỏi registry khi không còn healthy, để các service khác luôn gọi tới đúng instance đang sống.

**Yêu cầu cụ thể:**
- Mỗi instance khi start phải tự đăng ký vào registry kèm địa chỉ, port, version, và metadata (ví dụ zone/datacenter); khi nhận được shutdown signal phải tự deregister trước khi tắt.
- Registry phải phân biệt được health check "active" (registry tự ping endpoint `/health` định kỳ) và "passive" (instance tự gửi heartbeat) — chọn một chiến lược và giải thích trade-off, xử lý được trường hợp instance vẫn sống nhưng mạng tới registry bị nghẽn tạm thời (không được xóa nhầm).
- Nếu một instance crash mà không kịp deregister (kill -9, mất điện), registry phải tự phát hiện qua TTL/heartbeat timeout và loại nó ra trong một khoảng thời gian xác định (SLA phát hiện, ví dụ dưới 30 giây).
- Client (service gọi) phải cache danh sách instance để giảm tải cho registry, nhưng vẫn phải cập nhật kịp thời khi có thay đổi — định nghĩa rõ TTL cache và cơ chế invalidate.
- Xử lý race condition: hai instance cùng đăng ký gần như đồng thời với cùng service name nhưng version khác nhau trong lúc đang rolling deploy — client gọi phải nhận được danh sách nhất quán, không bị lẫn instance cũ/mới theo cách gây lỗi.

---

## Quản lý fleet thiết bị IoT online/offline

**Repository:** `service-discovery-iot-fleet-health`

**Hệ thống:** Một backend quản lý hàng chục nghìn thiết bị IoT (cảm biến nhiệt độ) kết nối liên tục qua MQTT/WebSocket tới các gateway server.

**Vai trò của flow:** Hệ thống cần biết chính xác thiết bị nào đang online, gateway nào đang giữ kết nối của thiết bị nào, để route lệnh điều khiển tới đúng nơi.

**Yêu cầu cụ thể:**
- Khi thiết bị connect tới một gateway, gateway phải đăng ký ánh xạ `device_id -> gateway_id` vào một registry chia sẻ (không lưu local), để các service khác biết gửi lệnh qua gateway nào.
- Phát hiện thiết bị offline không chỉ dựa vào việc gateway đóng kết nối tường minh, mà còn phải xử lý mất kết nối "âm thầm" (thiết bị mất điện, mất sóng) qua heartbeat/keepalive với timeout hợp lý — tránh báo sai "online" khi thiết bị đã chết từ lâu.
- Khi một gateway server bị restart (deploy mới), toàn bộ thiết bị đang giữ trên gateway đó phải được đánh dấu "cần reconnect" và registry phải dọn sạch mapping cũ, không để lại bản ghi rác gây route sai.
- Thiết kế cho khả năng mở rộng: registry phải chịu được việc hàng nghìn thiết bị connect/disconnect trong vài giây (ví dụ sau một lần mất điện khu vực rộng) mà không sập hoặc mất dữ liệu trạng thái.
- Cung cấp API cho phép các service khác truy vấn "thiết bị X đang online ở gateway nào" với latency thấp, và trả lời rõ ràng khi thiết bị không tồn tại trong registry (chưa từng connect vs vừa mất kết nối).

---

## Service discovery đa vùng (multi-region) cho hệ thống thanh toán

**Repository:** `service-discovery-payment-multi-region`

**Hệ thống:** Một hệ thống thanh toán triển khai ở 2 region (chính và phụ) để đảm bảo độ trễ thấp cho khách hàng ở các khu vực khác nhau.

**Vai trò của flow:** Service discovery phải ưu tiên route trong cùng region, nhưng có khả năng phát hiện khi toàn bộ instance của một service trong một region đều unhealthy để chuyển hướng sang region khác.

**Yêu cầu cụ thể:**
- Registry phải lưu metadata region cho mỗi instance, và client mặc định chỉ discover instance trong cùng region trước, chỉ fallback sang region khác khi region hiện tại không còn instance healthy nào.
- Health check phải phân biệt được lỗi tạm thời (network jitter giữa hai region) với lỗi thật (service đã chết) — tránh flapping (bật/tắt trạng thái liên tục) gây route qua lại giữa region không cần thiết.
- Khi cross-region failover xảy ra, phải có cảnh báo (alert) tự động cho team vận hành, và mọi request bị route sang region khác phải được đánh dấu (header/log) để phục vụ debug độ trễ tăng bất thường.
- Đảm bảo dữ liệu registry của hai region đồng bộ đủ nhanh (không cần strong consistency tuyệt đối) nhưng không để tình trạng một region không biết region kia còn sống hay không trong thời gian dài.
- Thiết kế test mô phỏng "network partition" giữa hai region (registry hai bên không thấy nhau) — hệ thống phải không tự ý coi region kia là chết hoàn toàn chỉ vì mất liên lạc giữa hai registry.

---

## Service mesh nội bộ cho doanh nghiệp với sidecar health check

**Repository:** `service-discovery-service-mesh-sidecar`

**Hệ thống:** Một nền tảng nội bộ doanh nghiệp triển khai nhiều service trên Kubernetes, mỗi pod có một sidecar proxy đảm nhiệm health check và discovery.

**Vai trò của flow:** Sidecar tự động phát hiện các service khác qua control plane, đồng thời liên tục health check các endpoint nó đang giao tiếp để định tuyến traffic tránh các pod lỗi.

**Yêu cầu cụ thể:**
- Sidecar phải hỗ trợ cả liveness check (pod có đang chạy không) và readiness check (pod đã sẵn sàng nhận traffic chưa) — phân biệt rõ hai loại và xử lý khác nhau (liveness fail thì restart pod, readiness fail thì chỉ loại khỏi routing tạm thời).
- Khi một pod mới được scheduler đưa lên (do autoscale), nó không được nhận traffic ngay cho tới khi readiness check pass ít nhất N lần liên tiếp, tránh nhận request trong lúc đang warm-up (load config, kết nối DB pool...).
- Nếu control plane (nơi lưu danh sách service) tạm thời không phản hồi được, sidecar phải dùng danh sách cache gần nhất để tiếp tục hoạt động (fail open có kiểm soát), không được để toàn bộ traffic đứng lại vì mất kết nối tới control plane.
- Đảm bảo health check không tạo ra tải đáng kể lên chính service bị check (tránh trường hợp health check dày đặc làm chậm thêm service đang yếu) — có backoff khi liên tục fail.
- Viết cơ chế báo cáo rõ nguyên nhân một pod bị đánh unhealthy (timeout, HTTP 5xx, connection refused) để team vận hành debug nhanh, không chỉ trả về "unhealthy" chung.

---

## Discovery cho marketplace có nhiều service scale động theo giờ cao điểm

**Repository:** `service-discovery-marketplace-dynamic-scaling`

**Hệ thống:** Một marketplace online (giống các sàn C2C) có service "search" và "recommendation" tự động scale số instance lên/xuống theo giờ cao điểm trong ngày.

**Vai trò của flow:** Discovery phải theo kịp việc số instance thay đổi liên tục trong ngày, đảm bảo traffic luôn được route tới instance còn sống và tránh gọi vào instance vừa bị scale-down.

**Yêu cầu cụ thể:**
- Khi autoscaler quyết định giảm số instance, instance bị chọn để tắt phải deregister trước, chờ hết các request đang xử lý (grace period) rồi mới thực sự shutdown — không deregister sau khi đã tắt.
- Client phải xử lý được trường hợp gọi tới một instance vừa bị loại khỏi registry đúng lúc request đang bay trên đường truyền (registry cập nhật chưa tới nơi) — có retry với instance khác, không throw lỗi trực tiếp cho người dùng cuối.
- Định nghĩa rõ khoảng thời gian tối đa registry được coi là "stale" trước khi client buộc phải refresh lại toàn bộ danh sách instance từ nguồn gốc.
- Thiết kế cho trường hợp cả registry chính bị quá tải trong giờ cao điểm (chính registry cũng cần discovery/health check) — không tạo ra single point of failure khi traffic tăng cao nhất.
- Viết test đo thời gian từ lúc một instance chết đột ngột tới lúc 100% traffic mới không còn route tới nó nữa, đảm bảo nằm trong SLA đã định nghĩa trước.
