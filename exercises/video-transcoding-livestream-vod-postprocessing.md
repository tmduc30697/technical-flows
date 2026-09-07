# Hậu xử lý bản ghi sau khi kết thúc livestream

**Hệ thống:** Một nền tảng livestream cho phép người xem xem lại (VOD) sau khi buổi live kết thúc.

**Vai trò của flow:** Nhận file ghi hình thô (đã ghép từ các segment trong lúc live) và chạy transcode thành VOD chất lượng cao, khác với luồng xử lý real-time lúc đang live.

**Yêu cầu cụ thể:**
- Khi stream kết thúc, hệ thống phải ghép các segment ghi hình (thường ở dạng .ts nhỏ) thành file liên tục chính xác theo thứ tự thời gian, xử lý được trường hợp segment bị thiếu/lỗi do gián đoạn mạng lúc live.
- Chạy transcode lại toàn bộ VOD ở chất lượng cao hơn bản live gốc (vì không còn ràng buộc độ trễ thấp), tận dụng được thời gian off-peak để giảm chi phí worker.
- VOD phải giữ được đúng các mốc thời gian quan trọng trong lúc live (ví dụ thời điểm donate, highlight được đánh dấu) sau khi ghép và transcode.
- Nếu buổi live kéo dài nhiều giờ, pipeline phải xử lý theo từng đoạn (chunked processing) thay vì chờ xử lý toàn bộ một lần, để VOD có thể sẵn sàng dần (phần đầu xem được trước khi phần cuối xử lý xong).
- Có cơ chế dọn dữ liệu ghi hình thô sau khi VOD đã tạo thành công và được xác nhận phát được, tránh giữ trùng lặp dữ liệu gây tốn chi phí lưu trữ lâu dài.
