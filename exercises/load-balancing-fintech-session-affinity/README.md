# Load balancer cho hệ thống thanh toán fintech cần session affinity

**Hệ thống:** Một hệ thống xử lý giao dịch chuyển tiền, trong đó một giao dịch có thể gồm nhiều bước gọi lại cùng service instance (ví dụ giữ state tạm trong bộ nhớ instance để xử lý multi-step OTP).

**Vai trò của flow:** Dùng consistent hashing (hash theo transaction_id hoặc user_id) để đảm bảo các request thuộc cùng một giao dịch luôn được route tới cùng một instance, tránh phải đọc lại state từ DB mỗi bước.

**Yêu cầu cụ thể:**
- Implement consistent hashing ring, hash key là `transaction_id`; đảm bảo cùng key luôn map tới cùng instance trong suốt vòng đời giao dịch (thường vài phút).
- Khi một instance chết giữa lúc đang giữ state của giao dịch đang chạy, phải có cơ chế phát hiện và fail-over request tiếp theo sang instance khác kèm khôi phục state từ nguồn bền (DB/cache) — không được để giao dịch "treo" vô thời hạn.
- Khi thêm/bớt instance khỏi ring (scale in/out), số lượng key phải bị remap là tối thiểu (đúng đặc tính consistent hashing), verify bằng test đo tỷ lệ % key bị đổi instance trước/sau khi thêm 1 node.
- Có cơ chế virtual node (nhiều điểm hash cho một instance) để tránh phân bố lệch tải khi cluster nhỏ (ví dụ chỉ 3-4 instance).
- Đảm bảo bảo mật: hash key không được lộ ra thông tin nhạy cảm qua timing attack (thời gian route không được khác biệt đáng kể giữa các user).
