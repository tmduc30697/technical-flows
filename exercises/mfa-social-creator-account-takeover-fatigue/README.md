# Mạng xã hội bảo vệ tài khoản creator/influencer lớn khỏi chiếm quyền và tấn công MFA fatigue

**Hệ thống:** Một nền tảng mạng xã hội cho phép đăng nội dung công khai, tài khoản có lượng follower lớn (creator/influencer) là mục tiêu thường xuyên bị nhắm tới để chiếm quyền, thường nhằm lừa đảo follower hoặc đòi tiền chuộc.

**Vai trò của flow:** Tự động áp dụng chính sách MFA mạnh hơn cho tài khoản có ảnh hưởng lớn, đồng thời phát hiện và ngăn chặn tấn công "MFA fatigue" — kẻ tấn công spam liên tục yêu cầu push notification approve nhằm khiến chủ tài khoản bấm nhầm chấp nhận.

**Yêu cầu cụ thể:**
- Hệ thống phải tự động nâng cấp yêu cầu MFA (ví dụ buộc chuyển từ SMS hoặc không có MFA sang WebAuthn/TOTP) khi tài khoản vượt một ngưỡng follower hoặc mức độ ảnh hưởng nhất định, kèm luồng thông báo và hỗ trợ enroll để không đột ngột khóa quyền truy cập của creator đang hoạt động bình thường.
- Phải giới hạn số lượng push notification MFA gửi liên tiếp trong một khoảng thời gian ngắn, tự động khóa các yêu cầu mới sau một số lần nhất định, để chặn kịch bản kẻ tấn công spam hàng chục request đăng nhập liên tục nhằm khiến chủ tài khoản mệt mỏi và bấm approve nhầm.
- Giao diện push notification approve phải hiển thị đủ ngữ cảnh (vị trí, thiết bị, thời gian request) và yêu cầu một hành động xác nhận có chủ đích, ví dụ nhập số hiển thị trên màn hình đăng nhập vào app push (number matching), thay vì chỉ một nút "Approve" đơn giản rất dễ bấm nhầm khi đang bị dồn dập.
- Khi phát hiện một chuỗi request MFA bị từ chối liên tiếp trong thời gian ngắn — dấu hiệu rõ ràng của tấn công thay vì nhầm lẫn ngẫu nhiên — hệ thống phải tự động tạm khóa đăng nhập từ nguồn phát sinh request và cảnh báo chủ tài khoản qua một kênh độc lập, không chờ họ tự nhận ra đang bị tấn công.
- Với tài khoản creator lớn, cần một phương thức phục hồi khẩn cấp riêng (đường dây hỗ trợ ưu tiên hoặc xác minh danh tính tăng cường) khi họ thực sự mất yếu tố MFA, cân bằng giữa tốc độ khôi phục — vì mất quyền truy cập lâu gây thiệt hại uy tín/doanh thu — và rủi ro kẻ tấn công lợi dụng chính kênh phục hồi ưu tiên này để chiếm tài khoản.
