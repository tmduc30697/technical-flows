# Sharding lưu trữ tin nhắn theo conversation cho app chat

**Hệ thống:** App chat với hàng triệu cuộc hội thoại, mỗi cuộc hội thoại có thể có rất nhiều tin nhắn, cần chia dữ liệu ra nhiều shard DB.

**Vai trò của flow:** Consistent hashing/partition theo conversation ID để đảm bảo tất cả tin nhắn của một cuộc hội thoại nằm trên cùng shard (giữ tính cục bộ để đọc lịch sử nhanh, không cần join xuyên shard).

**Yêu cầu cụ thể:**
- Toàn bộ message của một conversation_id phải luôn thuộc đúng một shard xác định để truy vấn lịch sử chat không phải scatter-gather nhiều shard.
- Xử lý được "hot conversation" (group chat cực lớn, hàng trăm nghìn tin nhắn/ngày) không làm quá tải shard chứa nó — nêu chiến lược tách riêng (ví dụ shard riêng cho conversation vượt ngưỡng kích thước).
- Khi thêm shard mới, chỉ những conversation thuộc range bị dịch chuyển mới cần migrate, các conversation khác không bị ảnh hưởng gì (không downtime).
- Đảm bảo trong lúc migrate một conversation sang shard mới, tin nhắn gửi tới trong lúc migrate không bị mất hoặc ghi nhầm shard cũ.
- Đo lường: độ lệch kích thước dữ liệu giữa các shard theo thời gian, và cảnh báo khi một shard vượt X% dung lượng trung bình để chủ động rebalance trước khi đầy.
