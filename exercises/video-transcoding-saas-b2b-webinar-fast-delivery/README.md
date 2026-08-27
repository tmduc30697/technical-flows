# SaaS B2B webinar với yêu cầu gửi bản ghi ngay sau buổi họp

**Hệ thống:** Một SaaS B2B tổ chức webinar/họp video với khách hàng, cần gửi bản ghi lại cho khách hàng ngay sau khi buổi họp kết thúc, một số trường hợp cần tách riêng audio/video theo từng người nói.

**Vai trò của flow:** Nhận file ghi hình buổi họp/webinar ngay sau khi kết thúc, transcode và xử lý ưu tiên tốc độ để có thể gửi cho khách hàng trong thời gian ngắn nhất, kèm khả năng tách nguồn theo người nói nếu được yêu cầu.

**Yêu cầu cụ thể:**
- Thời gian từ lúc buổi họp kết thúc đến lúc bản ghi sẵn sàng gửi cho khách hàng phải rất ngắn để giữ được ngữ cảnh còn "nóng" của cuộc họp — pipeline này cần ưu tiên tài nguyên xử lý cao hơn hẳn so với video thông thường không có ràng buộc thời gian, kể cả phải đánh đổi chi phí compute cao hơn cho một phiên xử lý ưu tiên.
- Nếu buổi họp có nhiều người nói được ghi trên các track/nguồn audio riêng biệt, việc ghép và đồng bộ các track này với video chung phải chính xác đến từng khung hình để tránh audio người này chồng lấn/lệch với hình ảnh người khác đang nói.
- Trường hợp buổi họp bị dừng đột ngột giữa chừng hoặc file ghi hình bị lỗi một phần, hệ thống phải xử lý được phần dữ liệu còn dùng được để gửi bản ghi một phần cho khách hàng kèm thông báo rõ ràng, thay vì để khách hàng chờ vô thời hạn một bản ghi sẽ không bao giờ hoàn chỉnh.
- Nội dung webinar/họp thường chứa thông tin nhạy cảm giữa hai doanh nghiệp — bản ghi trong lúc đang xử lý và file trung gian phải được cô lập đúng theo từng khách hàng/tổ chức, không để rò rỉ chéo giữa hàng đợi xử lý của các khách hàng khác nhau kể cả khi chạy trên cùng hạ tầng xử lý dùng chung.
- Khi có nhu cầu tách riêng audio/video theo từng người nói để phục vụ nhu cầu khác nhau của khách hàng, việc tách này không được yêu cầu xử lý lại từ đầu toàn bộ file gốc mỗi lần có yêu cầu mới, mà nên tận dụng lại kết quả trung gian đã tách nguồn từ lần xử lý ban đầu.
