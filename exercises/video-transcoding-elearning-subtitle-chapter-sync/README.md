# E-learning với phụ đề và chương mục đồng bộ

**Hệ thống:** Một nền tảng e-learning, video bài giảng cần đi kèm phụ đề và chương mục (chapter markers) để học viên tra cứu nhanh nội dung.

**Vai trò của flow:** Transcode video bài giảng ra nhiều độ phân giải đồng thời xử lý và gắn phụ đề/mốc chương mục, đảm bảo các mốc thời gian này luôn khớp chính xác với video sau khi qua các bước xử lý.

**Yêu cầu cụ thể:**
- Khi video được transcode qua nhiều bước (cắt intro thừa, chuẩn hóa framerate, đổi tốc độ phát nếu giảng viên upload bản đã chỉnh sửa), các mốc timestamp của phụ đề và chapter marker phải được tính lại đồng bộ theo từng bước biến đổi, tránh tình trạng phụ đề bị lệch dần so với hình sau khi qua pipeline xử lý.
- Nếu giảng viên upload file phụ đề riêng sau khi video đã transcode xong và đã publish cho học viên xem, việc gắn thêm phụ đề mới không được yêu cầu transcode lại toàn bộ video, mà chỉ cần xử lý và đồng bộ lại phần phụ đề độc lập với các rendition video đã có sẵn.
- Chapter marker do giảng viên đánh dấu thủ công tại thời điểm xem bản gốc trước khi transcode có thể lệch nếu quá trình transcode làm thay đổi độ dài tổng thể video (ví dụ loại bỏ đoạn câm/đen ở đầu) — cần cơ chế ánh xạ lại chính xác mốc thời gian gốc sang mốc thời gian của video đã transcode, không đơn giản giữ nguyên số giây gốc.
- Với khóa học có video dài, việc xử lý phụ đề/chapter không được làm chậm thời gian video sẵn sàng để học viên xem — cần tách pipeline xử lý phụ đề/chapter chạy song song độc lập với pipeline transcode video chính, cho phép video hiển thị trước và phụ đề/chapter cập nhật bổ sung sau nếu xử lý xong muộn hơn.
- Khi giảng viên chỉnh sửa lại nội dung phụ đề sau khi khóa học đã có học viên đang học, bản phụ đề cũ đang được học viên xem dở không được thay đổi đột ngột giữa chừng gây gián đoạn trải nghiệm, cần có cơ chế áp dụng bản cập nhật hợp lý (ví dụ áp dụng từ lần xem tiếp theo).
