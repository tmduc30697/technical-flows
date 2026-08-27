# App messaging với phiên đăng nhập liên kết đa thiết bị (linked devices)

**Hệ thống:** Một app tin nhắn (giống WhatsApp/Telegram) cho phép dùng đồng thời trên điện thoại (thiết bị chính) và mở thêm trên web/desktop (thiết bị liên kết).

**Vai trò của flow:** Quản lý session dạng "thiết bị chính + nhiều thiết bị liên kết", đồng bộ trạng thái và cho phép revoke từng thiết bị liên kết mà không ảnh hưởng thiết bị chính.

**Yêu cầu cụ thể:**
- Việc liên kết thiết bị mới (web/desktop) phải được xác nhận trực tiếp từ thiết bị chính (ví dụ quét QR code hoặc approve trên điện thoại), không cho liên kết chỉ bằng thông tin đăng nhập từ xa.
- Mỗi thiết bị liên kết phải có khóa mã hóa session riêng, revoke một thiết bị liên kết không được ảnh hưởng tới session của thiết bị chính hoặc các thiết bị liên kết khác.
- Người dùng phải xem được danh sách đầy đủ thiết bị liên kết đang hoạt động (kèm thời điểm liên kết, lần hoạt động cuối) ngay trên thiết bị chính, và revoke được từng thiết bị riêng lẻ ngay lập tức.
- Nếu thiết bị chính bị mất/đăng xuất, phải có chính sách rõ ràng cho các thiết bị liên kết đang hoạt động (ví dụ tự động hết hạn sau một khoảng thời gian, hoặc yêu cầu liên kết lại với thiết bị chính mới).
- Đồng bộ tin nhắn/trạng thái đọc giữa các thiết bị liên kết phải xử lý được race condition khi hai thiết bị cùng thao tác gần như đồng thời (ví dụ đánh dấu đã đọc trên cả hai thiết bị cùng lúc) mà không gây trạng thái không nhất quán.
