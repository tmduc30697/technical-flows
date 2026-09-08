# Service discovery đa vùng (multi-region) cho hệ thống thanh toán

**Hệ thống:** Một hệ thống thanh toán triển khai ở 2 region (chính và phụ) để đảm bảo độ trễ thấp cho khách hàng ở các khu vực khác nhau.

**Vai trò của flow:** Service discovery phải ưu tiên route trong cùng region, nhưng có khả năng phát hiện khi toàn bộ instance của một service trong một region đều unhealthy để chuyển hướng sang region khác.

**Yêu cầu cụ thể:**
- Registry phải lưu metadata region cho mỗi instance, và client mặc định chỉ discover instance trong cùng region trước, chỉ fallback sang region khác khi region hiện tại không còn instance healthy nào.
- Health check phải phân biệt được lỗi tạm thời (network jitter giữa hai region) với lỗi thật (service đã chết) — tránh flapping (bật/tắt trạng thái liên tục) gây route qua lại giữa region không cần thiết.
- Khi cross-region failover xảy ra, phải có cảnh báo (alert) tự động cho team vận hành, và mọi request bị route sang region khác phải được đánh dấu (header/log) để phục vụ debug độ trễ tăng bất thường.
- Đảm bảo dữ liệu registry của hai region đồng bộ đủ nhanh (không cần strong consistency tuyệt đối) nhưng không để tình trạng một region không biết region kia còn sống hay không trong thời gian dài.
- Thiết kế test mô phỏng "network partition" giữa hai region (registry hai bên không thấy nhau) — hệ thống phải không tự ý coi region kia là chết hoàn toàn chỉ vì mất liên lạc giữa hai registry.
