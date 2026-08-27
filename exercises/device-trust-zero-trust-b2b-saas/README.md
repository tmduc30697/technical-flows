# B2B SaaS Zero Trust chỉ cho phép truy cập từ thiết bị công ty quản lý

**Hệ thống:** Một công cụ nội bộ doanh nghiệp chứa dữ liệu nhạy cảm (tài chính, nhân sự), công ty áp dụng chính sách Zero Trust: chỉ thiết bị đã được MDM (Mobile Device Management) quản lý và đạt chuẩn bảo mật mới được truy cập.

**Vai trò của flow:** Session chỉ được cấp khi thiết bị chứng minh được "posture" hợp lệ (certificate do MDM cấp, trạng thái mã hóa ổ đĩa, phiên bản OS đủ mới) — không dựa vào chỉ đăng nhập đúng mật khẩu.

**Yêu cầu cụ thể:**
- Request truy cập phải kèm theo chứng chỉ thiết bị (device certificate) do hệ thống MDM cấp, và server phải verify được certificate này còn hợp lệ (chưa bị revoke, đúng chuỗi tin cậy) trước khi cấp session.
- Nếu thiết bị không có certificate hợp lệ (BYOD, thiết bị cá nhân), phải từ chối truy cập với thông báo rõ ràng hướng dẫn liên hệ IT để đăng ký thiết bị, không hiển thị lỗi mơ hồ.
- Trong lúc session đang hoạt động, nếu MDM báo cáo thiết bị chuyển sang trạng thái không đạt chuẩn (ví dụ tắt mã hóa ổ đĩa, jailbreak/root), hệ thống phải buộc kết thúc session hiện tại gần như ngay lập tức, không chờ session tự hết hạn.
- Chính sách posture yêu cầu (phiên bản OS tối thiểu, bắt buộc mã hóa...) phải cấu hình được theo nhóm người dùng/phòng ban khác nhau, không phải một chính sách cứng áp dụng chung cho mọi người.
- Có luồng dự phòng có kiểm soát (ví dụ VPN tạm thời + xác minh thủ công bởi IT) cho trường hợp nhân viên cần truy cập gấp từ thiết bị không đạt chuẩn, được ghi log đầy đủ và giới hạn thời gian.
