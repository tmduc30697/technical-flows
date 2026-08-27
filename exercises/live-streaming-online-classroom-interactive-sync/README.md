# Lớp học trực tuyến kéo dài nhiều giờ với tương tác hai chiều

**Hệ thống:** Một nền tảng dạy học trực tuyến, giáo viên stream bài giảng trong các buổi học kéo dài nhiều giờ liên tục, học sinh tương tác hai chiều (giơ tay, đặt câu hỏi qua chat) trong lúc học.

**Vai trò của flow:** Nhận luồng bài giảng từ giáo viên, phân phối tới học sinh, đồng thời đảm bảo các sự kiện tương tác được đồng bộ đúng thời điểm với nội dung đang giảng và được ghi lại chính xác phục vụ xem lại sau.

**Yêu cầu cụ thể:**
- Với buổi học kéo dài nhiều giờ liên tục, pipeline ingest/ghi hình phải chịu được vận hành ổn định lâu dài (tránh rò rỉ tài nguyên worker, tích lũy lỗi nhỏ theo thời gian) mà không cần restart giữa chừng làm gián đoạn giờ học, khác hẳn các luồng live ngắn vài chục phút.
- Sự kiện học sinh giơ tay hoặc gửi câu hỏi phải được gắn đúng mốc thời gian tương ứng với nội dung giáo viên đang giảng tại thời điểm đó, kể cả khi độ trễ phân phối khiến học sinh xem chậm hơn giáo viên vài giây — tránh tình huống giáo viên trả lời một câu hỏi khi nội dung bài giảng đã trôi qua xa so với thời điểm học sinh thực sự đặt câu hỏi.
- Nếu kết nối của giáo viên bị gián đoạn giữa buổi giảng, hệ thống phải cho phép khôi phục đúng vị trí trong buổi học mà không tạo phiên mới, đồng thời không làm mất các tương tác đã phát sinh trong lúc gián đoạn.
- Bản ghi lại buổi học để xem sau phải giữ đồng bộ chính xác giữa video bài giảng và các mốc tương tác quan trọng (câu hỏi được giáo viên trả lời, thời điểm chuyển sang phần bài mới), để học sinh xem lại tra cứu đúng theo mốc thời gian mà không bị lệch do độ trễ tích lũy qua nhiều giờ ghi hình.
- Khi số lượng học sinh tham gia đồng thời một lớp lớn (hàng trăm/nghìn), lượng sự kiện tương tác gửi lên gần như đồng thời có thể tạo áp lực ghi nhận lớn — cần đảm bảo kênh tương tác này không cạnh tranh tài nguyên với luồng video chính, tránh làm giảm chất lượng/độ trễ phát sóng khi lớp học sôi động.
