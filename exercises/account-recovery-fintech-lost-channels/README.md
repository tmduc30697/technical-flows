# Fintech: khôi phục tài khoản khi mất cả email và số điện thoại đăng ký

**Hệ thống:** Một app ví điện tử/đầu tư cá nhân, tài khoản có gắn với tiền thật, quy định pháp lý yêu cầu xác minh danh tính nghiêm ngặt.

**Vai trò của flow:** Xử lý case khó nhất — người dùng mất quyền truy cập cả email và số điện thoại đã đăng ký, cần một quy trình khôi phục có kiểm soát dựa trên xác minh danh tính, không thể tự động hoàn toàn.

**Yêu cầu cụ thể:**
- Khi không còn kênh liên hệ nào khớp với thông tin đã đăng ký, hệ thống phải chuyển sang luồng xác minh danh tính thủ công/bán tự động (ví dụ upload giấy tờ tùy thân, chụp ảnh khớp khuôn mặt) trước khi cấp lại quyền truy cập.
- Trong suốt thời gian chờ xác minh, tài khoản phải bị đóng băng các hành động rút tiền/chuyển tiền, nhưng vẫn có thể xem được thông tin (nếu cần) để tránh vừa mất quyền truy cập vừa mất khả năng theo dõi tài sản.
- Phải có audit trail đầy đủ và không thể chỉnh sửa cho toàn bộ quy trình xác minh khôi phục (ai duyệt, dựa trên bằng chứng gì, khi nào) để đáp ứng yêu cầu compliance tài chính khi bị thanh tra.
- Sau khi khôi phục thành công, mọi thiết bị/session cũ phải bị đăng xuất, và phải có một khoảng "cooling-off period" (ví dụ 24-48 giờ) trước khi cho phép rút tiền lớn lần đầu sau khôi phục, để giảm rủi ro nếu quy trình xác minh bị lợi dụng.
- Thiết kế được cơ chế chống social engineering nhằm vào bộ phận hỗ trợ khách hàng (kẻ tấn công giả làm chủ tài khoản để yêu cầu khôi phục) bằng việc yêu cầu nhiều bằng chứng độc lập, không dựa vào một câu hỏi bảo mật duy nhất.
