# WAL cho message queue broker tự xây đảm bảo message không mất

**Hệ thống:** Nhóm tự xây một message queue broker đơn giản để producer gửi message, consumer đọc và ack, cần đảm bảo message đã ack cho producer (nhận thành công) không bị mất kể cả khi broker crash trước khi consumer kịp đọc.

**Vai trò của flow:** WAL ghi message xuống đĩa ngay khi broker nhận được, trước khi trả ack "đã nhận" cho producer, và dùng WAL để phục hồi hàng đợi (thứ tự, message chưa được consumer đọc/ack) sau khi broker crash và restart.

**Yêu cầu cụ thể:**
- Broker chỉ được trả ack "đã nhận" cho producer sau khi message được ghi (fsync) vào WAL thành công; nếu broker crash giữa lúc ghi WAL (trước fsync) thì phải không trả ack — producer coi như gửi thất bại và tự retry, tránh tình huống producer tưởng đã gửi thành công nhưng message chưa từng được ghi bền vững.
- Sau crash, khi broker khởi động lại phải replay WAL để tái tạo đúng thứ tự hàng đợi và biết chính xác message nào consumer đã ack trước lúc crash (không đưa lại message đã ack cho consumer đọc lần nữa) và message nào chưa ack (phải đưa lại để consumer xử lý tiếp) — cần ghi rõ trạng thái ack vào WAL, không chỉ ghi lúc nhận message.
- Xử lý được trường hợp consumer đã đọc message, đang xử lý dở thì broker crash trước khi consumer kịp gửi ack — sau khi broker phục hồi từ WAL, message đó phải coi là "chưa ack" và được giao lại cho consumer (có thể là consumer khác), đòi hỏi việc xử lý message ở phía consumer phải idempotent để chịu được đọc trùng.
- Nhiều producer ghi đồng thời vào broker tạo áp lực ghi WAL liên tục — cần checkpoint định kỳ để cắt bớt phần WAL chứa toàn message đã được mọi consumer liên quan ack xong, tránh WAL phình vô hạn khiến thời gian replay khi crash ngày càng dài.
- Đo lường thời gian từ lúc broker crash tới lúc sẵn sàng nhận/giao message lại bình thường (recovery time), tương quan với kích thước WAL và số lượng message chưa checkpoint, để xác định ngưỡng cảnh báo vận hành trước khi recovery time vượt SLA cho phép.
