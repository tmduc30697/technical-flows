# Service discovery & health check flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (nền tảng microservices SaaS, chuỗi bán lẻ đa chi nhánh, hạ tầng multi-region, service mesh nội bộ) để luyện việc đăng ký, phát hiện và theo dõi sức khỏe service một cách đúng đắn.

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

---

## Service discovery & health check cho hàng nghìn thiết bị POS tại chuỗi cửa hàng bán lẻ kết nối không liên tục

**Repository:** `service-discovery-retail-pos-intermittent-connectivity`

**Hệ thống:** Một nền tảng quản lý hàng nghìn thiết bị điểm bán hàng (POS) tại các chi nhánh bán lẻ trên toàn quốc, mỗi thiết bị kết nối qua đường truyền internet riêng của cửa hàng (chất lượng khác nhau tùy khu vực, có SIM 4G dự phòng khi đứt cáp) và tự động ngắt kết nối/vào chế độ nghỉ ngoài giờ mở cửa của cửa hàng.

**Vai trò của flow:** Health check ở đây khác căn bản so với service backend thông thường — một thiết bị "không phản hồi" có thể đơn giản là cửa hàng đã đóng cửa theo giờ bình thường, không phải thiết bị đã hỏng hay mất kết nối thật, nên flow phải phân biệt được hai trạng thái này để tránh báo động giả hàng loạt gây nhiễu đội vận hành.

**Yêu cầu cụ thể:**
- Mỗi thiết bị khi đăng ký vào hệ thống phải khai báo lịch hoạt động dự kiến của chi nhánh (giờ mở/đóng cửa, ngày nghỉ lễ) thay vì áp dụng một ngưỡng timeout cố định chung cho mọi thiết bị — health check phải so sánh thời gian im lặng thực tế với lịch riêng của từng chi nhánh để quyết định có bất thường hay không.
- Xử lý trường hợp một thiết bị không online đúng giờ mở cửa dự kiến của chi nhánh (quá giờ mở cửa mà chưa thấy tín hiệu) — trước khi báo động, phải có một khoảng đệm (grace window) tính đến các yếu tố như nhân viên mở cửa trễ vài phút, đường truyền internet cửa hàng khởi động chậm, thay vì báo "chết" ngay khi vừa trễ lịch một chút.
- Đảm bảo hệ thống phân biệt rõ ba trạng thái riêng biệt cho mỗi thiết bị (đang hoạt động, đang nghỉ theo giờ đóng cửa bình thường, mất kết nối bất thường/nghi ngờ hỏng) thay vì chỉ hai trạng thái healthy/unhealthy nhị phân — mỗi trạng thái cần hành vi vận hành khác nhau (không cảnh báo, không cảnh báo, cảnh báo cần xử lý).
- Xử lý tình huống hàng loạt thiết bị trong cùng một khu vực đột ngột mất kết nối cùng lúc (nhà mạng khu vực gặp sự cố, đứt cáp quang vùng) — hệ thống cần phát hiện đây là sự cố hạ tầng mạng theo khu vực (correlation theo vị trí địa lý chi nhánh) để gộp thành một cảnh báo duy nhất, thay vì tạo ra hàng trăm cảnh báo "thiết bị chết" riêng lẻ làm ngập hệ thống thông báo và khiến đội vận hành bỏ sót cảnh báo thật sự quan trọng (ví dụ 1 chi nhánh riêng lẻ bị hỏng máy POS thật giữa lúc đang có sự cố mạng diện rộng).
- Thiết kế cơ chế heartbeat tiết kiệm chi phí dữ liệu qua đường truyền dự phòng (SIM 4G có giới hạn dung lượng, không thể ping dồn dập như service check nội bộ thông thường) — cân nhắc mô hình thiết bị chủ động báo cáo trạng thái theo chu kỳ dài hơn khi đang chạy trên đường truyền dự phòng, kèm cơ chế nền tảng chủ động gửi lệnh kiểm tra khẩn qua kênh riêng khi cần xác minh gấp một thiết bị nghi ngờ gặp sự cố, thay vì chờ tới chu kỳ báo cáo tiếp theo mới biết được.
