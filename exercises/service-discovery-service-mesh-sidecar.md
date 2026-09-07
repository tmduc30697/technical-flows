# Service mesh nội bộ cho doanh nghiệp với sidecar health check

**Hệ thống:** Một nền tảng nội bộ doanh nghiệp triển khai nhiều service trên Kubernetes, mỗi pod có một sidecar proxy đảm nhiệm health check và discovery.

**Vai trò của flow:** Sidecar tự động phát hiện các service khác qua control plane, đồng thời liên tục health check các endpoint nó đang giao tiếp để định tuyến traffic tránh các pod lỗi.

**Yêu cầu cụ thể:**
- Sidecar phải hỗ trợ cả liveness check (pod có đang chạy không) và readiness check (pod đã sẵn sàng nhận traffic chưa) — phân biệt rõ hai loại và xử lý khác nhau (liveness fail thì restart pod, readiness fail thì chỉ loại khỏi routing tạm thời).
- Khi một pod mới được scheduler đưa lên (do autoscale), nó không được nhận traffic ngay cho tới khi readiness check pass ít nhất N lần liên tiếp, tránh nhận request trong lúc đang warm-up (load config, kết nối DB pool...).
- Nếu control plane (nơi lưu danh sách service) tạm thời không phản hồi được, sidecar phải dùng danh sách cache gần nhất để tiếp tục hoạt động (fail open có kiểm soát), không được để toàn bộ traffic đứng lại vì mất kết nối tới control plane.
- Đảm bảo health check không tạo ra tải đáng kể lên chính service bị check (tránh trường hợp health check dày đặc làm chậm thêm service đang yếu) — có backoff khi liên tục fail.
- Viết cơ chế báo cáo rõ nguyên nhân một pod bị đánh unhealthy (timeout, HTTP 5xx, connection refused) để team vận hành debug nhanh, không chỉ trả về "unhealthy" chung.
