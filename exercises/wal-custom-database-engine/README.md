# Database engine tự xây dựng cho ứng dụng nội bộ

**Hệ thống:** Một nhóm xây dựng storage engine đơn giản (kiểu key-value hoặc bảng) cho một ứng dụng nội bộ, cần đảm bảo dữ liệu không mất khi crash.

**Vai trò của flow:** WAL ghi mọi thay đổi ra log tuần tự trước khi áp dụng vào cấu trúc dữ liệu chính (in-memory hoặc B-tree trên đĩa), đảm bảo có thể phục hồi đúng trạng thái sau crash.

**Yêu cầu cụ thể:**
- Mọi write phải được fsync xuống đĩa vào WAL trước khi trả response "thành công" cho client (durability trước khi ack), không được ack sớm khi dữ liệu còn nằm trong buffer OS chưa chắc đã ghi đĩa.
- Sau khi crash (kill -9 giữa lúc ghi), khi khởi động lại phải replay WAL đúng thứ tự để tái tạo đúng trạng thái tại thời điểm crash, không thiếu không thừa transaction đã commit.
- Ghi log phải phát hiện được entry bị ghi dở (torn write, ví dụ crash giữa lúc ghi 1 record) và bỏ qua phần dở đó khi replay, không làm hỏng toàn bộ log.
- Có checkpoint định kỳ để giới hạn độ dài WAL cần replay (không phải replay từ đầu thời gian sử dụng), và checkpoint phải an toàn nếu crash xảy ra giữa lúc đang checkpoint.
- Đo lường: thời gian recovery (từ lúc start tới lúc sẵn sàng nhận request) tương ứng với kích thước WAL cụ thể, để biết giới hạn WAL tối đa cho phép theo SLA khởi động lại.
