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
