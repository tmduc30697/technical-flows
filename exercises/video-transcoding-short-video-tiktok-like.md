# App video ngắn dạng TikTok

**Hệ thống:** Một mạng xã hội video ngắn, upload chủ yếu từ mobile, yêu cầu video xuất hiện gần như ngay lập tức sau khi đăng.

**Vai trò của flow:** Transcode nhanh video ngắn (dưới 60s) từ nhiều codec/tỉ lệ khung hình mobile khác nhau thành định dạng chuẩn hóa, tối ưu độ trễ end-to-end.

**Yêu cầu cụ thể:**
- Thời gian từ lúc upload xong đến lúc video "có thể xem được" (ít nhất 1 rendition) phải đạt SLA vài giây đến vài chục giây, khác hẳn pipeline video dài — cần ưu tiên hàng đợi riêng cho video ngắn.
- Chuẩn hóa được input từ nhiều tỉ lệ khung hình (9:16, 1:1, 16:9) và framerate khác nhau về cùng chuẩn output, không làm méo hoặc crop sai nội dung.
- Video phải đi qua bước kiểm tra sơ bộ (duration, kích thước, định dạng hợp lệ) trước khi vào hàng đợi transcode, từ chối sớm các file không hợp lệ để tránh tốn tài nguyên worker.
- Khi lượng upload tăng đột biến (viral moment, giờ cao điểm), hệ thống phải tự scale worker transcode theo hàng đợi, có cơ chế backpressure để không làm sập toàn bộ hệ thống.
- Video bị người dùng xóa trong lúc đang transcode phải hủy job giữa chừng và dọn sạch file trung gian, không tiếp tục xử lý lãng phí.
