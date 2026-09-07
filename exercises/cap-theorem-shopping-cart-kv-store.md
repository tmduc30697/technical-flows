# Distributed key-value store cho giỏ hàng (shopping cart)

**Hệ thống:** E-commerce lưu giỏ hàng của user trên một key-value store phân tán (kiểu DynamoDB/Cassandra) nhiều replica.

**Vai trò của flow:** Cho phép tunable consistency — cấu hình số replica cần xác nhận khi đọc/ghi (quorum) để cân bằng giữa tính khả dụng và tính nhất quán của giỏ hàng.

**Yêu cầu cụ thể:**
- Định nghĩa cụ thể tham số N (số replica), W (write quorum), R (read quorum) cho giỏ hàng, và giải thích rõ tại sao chọn W+R > N (đảm bảo đọc thấy ghi gần nhất) hay không.
- Trong lúc network partition giữa các replica, hệ thống phải ưu tiên availability cho hành động "thêm vào giỏ" (chấp nhận ghi ở phía minority, merge sau) vì trải nghiệm mua hàng quan trọng hơn nhất quán tuyệt đối tức thời.
- Khi partition được hàn lại, phải có chiến lược merge rõ ràng nếu 2 phía cùng sửa giỏ hàng khác nhau (ví dụ union các item được thêm, không tự động xóa item của bên nào).
- Cho phép client chọn mức consistency theo tình huống (ví dụ đọc giỏ hàng lúc checkout phải dùng read quorum cao hơn đọc để hiển thị icon số lượng ở header).
- Đo lường và log rõ tần suất phải "resolve conflict" xảy ra trong thực tế để đánh giá mức độ ảnh hưởng của việc chọn AP thay vì CP.
