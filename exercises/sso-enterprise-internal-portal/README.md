# Cổng đăng nhập chung cho các app nội bộ doanh nghiệp

**Hệ thống:** Một công ty có nhiều app nội bộ tách biệt (HR portal, hệ thống chấm công, wiki nội bộ, công cụ báo cáo chi phí), mỗi app hiện có login riêng.

**Vai trò của flow:** Xây một Identity Provider nội bộ tập trung — nhân viên đăng nhập một lần, sau đó truy cập mọi app con mà không cần đăng nhập lại (SSO nội bộ giữa các subdomain/app riêng biệt).

**Yêu cầu cụ thể:**
- Session đăng nhập trung tâm phải chia sẻ được giữa các app nằm trên domain/subdomain khác nhau, kèm cơ chế chống session fixation khi chuyển giữa các app.
- Khi nhân viên nghỉ việc, revoke session ở IdP trung tâm phải làm mất quyền truy cập ở tất cả app con gần như ngay lập tức (không chờ từng app tự hết session).
- Hỗ trợ Single Logout (SLO): đăng xuất ở một app phải đăng xuất khỏi toàn bộ app con đã SSO vào, kể cả khi user chỉ đóng tab mà không bấm logout rõ ràng ở app đó.
- Phân quyền theo app phải dựa trên role/group đồng bộ từ hệ thống HR (nguồn dữ liệu nhân sự), không quản lý quyền riêng lẻ ở từng app.
- Có cơ chế audit log tập trung: ghi lại mọi lần đăng nhập SSO, app nào được truy cập, để phục vụ điều tra khi có sự cố an ninh.
