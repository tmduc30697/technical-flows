# Tính route/ETA qua API bản đồ bên thứ ba cho ride-hailing

**Hệ thống:** App ride-hailing gọi API bản đồ/định tuyến của bên thứ ba để tính route và ETA cho tài xế trong lúc đang chở khách (chuyến đi đang chạy thời gian thực).

**Vai trò của flow:** Circuit breaker và retry/fallback giúp app tài xế không bị treo hoặc mất định hướng giữa chuyến đi khi API bản đồ bên ngoài chậm hoặc lỗi, vì đây là luồng ảnh hưởng trực tiếp tới trải nghiệm và an toàn vận hành trong lúc chuyến đi đang diễn ra.

**Yêu cầu cụ thể:**
- Khi API bản đồ chậm hoặc timeout giữa lúc chuyến đi đang chạy, app tài xế không được để màn hình chỉ đường bị treo hoặc trắng — phải fallback ngay về route/hướng dẫn gần nhất đã tính thành công trước đó (cached route) trong lúc chờ, hoặc chuyển sang provider bản đồ dự phòng, để tài xế luôn có chỉ dẫn khả dụng.
- Circuit breaker cho API bản đồ phải cấu hình ngưỡng nhạy với latency (không chỉ tỉ lệ lỗi cứng) vì với luồng real-time, một API trả đúng nhưng quá chậm (ví dụ vượt vài giây) cũng coi như vô dụng cho việc chỉ đường đang di chuyển — cần treat timeout mềm này như một dạng lỗi để tính vào ngưỡng mở breaker.
- Retry phải giới hạn số lần rất chặt (không quá 1-2 lần) và backoff phải cực ngắn cho luồng ETA/route trong chuyến đi đang chạy, vì tài xế đang di chuyển thực tế — retry kiểu chờ lâu (backoff dài như luồng batch khác) làm dữ liệu route trả về bị lỗi thời so với vị trí hiện tại của xe.
- Khi breaker mở kéo dài (provider bản đồ chính down), phải tự động chuyển toàn bộ app tài xế đang hoạt động sang provider dự phòng theo cơ chế graceful switch (không làm gián đoạn phiên đang chạy, không bắt tài xế phải khởi động lại app), đồng thời thông báo rõ tình trạng "đang dùng nguồn bản đồ dự phòng" nếu chất lượng route có thể khác biệt.
- Đo lường: latency thực tế của API bản đồ theo từng khu vực địa lý (một số vùng có thể có route/dữ liệu kém hơn ở provider chính), tỉ lệ chuyến đi phải fallback sang provider dự phòng, và tương quan giữa thời gian breaker mở với các phản hồi tiêu cực từ tài xế (báo cáo bị lạc đường, chỉ dẫn sai) để đánh giá tác động thực tế.
