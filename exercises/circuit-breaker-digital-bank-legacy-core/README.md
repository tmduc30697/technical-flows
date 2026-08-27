# Ngân hàng số gọi hệ thống core banking nội bộ (legacy)

**Hệ thống:** App ngân hàng số hiện đại gọi vào hệ thống core banking cũ (legacy, hiệu năng hạn chế) để lấy số dư, thực hiện giao dịch.

**Vai trò của flow:** Circuit breaker bảo vệ core banking (rất khó scale, không chịu được tải đột biến) khỏi bị app hiện đại gọi dồn dập lúc cao điểm hoặc lúc core banking đang chậm.

**Yêu cầu cụ thể:**
- Circuit breaker phải cấu hình chặt hơn bình thường (ngưỡng mở thấp hơn) để bảo vệ core banking vốn dễ bị sập khi quá tải, ưu tiên bảo vệ hệ thống lõi hơn là giữ app luôn phản hồi.
- Retry cho giao dịch ghi (chuyển tiền, thanh toán) phải cực kỳ cẩn trọng: chỉ retry khi chắc chắn core banking chưa xử lý request trước đó (dựa vào transaction reference được core banking hỗ trợ tra cứu trạng thái), tuyệt đối không retry mù.
- Với giao dịch đọc (xem số dư), có thể fallback về số dư cache gần nhất khi breaker mở, nhưng phải hiển thị rõ cho user đây là "số dư tại thời điểm gần nhất" không phải real-time.
- Khi breaker mở kéo dài (core banking down lâu), phải có quy trình escalate tới team vận hành ngay (alert khẩn) vì đây ảnh hưởng trực tiếp tới khả năng giao dịch của khách hàng.
- Ghi log đầy đủ mọi lần retry/circuit breaker chuyển trạng thái liên quan tới giao dịch tiền, phục vụ audit và đối soát khi có khiếu nại từ khách hàng.
