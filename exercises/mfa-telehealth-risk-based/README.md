# Nền tảng y tế từ xa với MFA thích ứng theo rủi ro (risk-based/adaptive)

**Hệ thống:** Một nền tảng khám bệnh từ xa (telemedicine) cho bác sĩ và bệnh nhân, phải tuân thủ quy định bảo mật dữ liệu y tế.

**Vai trò của flow:** MFA bắt buộc cho bác sĩ/nhân viên y tế, nhưng áp dụng adaptive MFA: chỉ yêu cầu xác thực thêm khi có tín hiệu rủi ro (thiết bị mới, vị trí bất thường, giờ truy cập lạ), giảm ma sát khi rủi ro thấp.

**Yêu cầu cụ thể:**
- Hệ thống phải tính điểm rủi ro dựa trên ít nhất: thiết bị đã biết hay chưa, vị trí địa lý (so với lịch sử đăng nhập), và thời điểm truy cập, để quyết định có yêu cầu MFA lần này hay không.
- Với bệnh nhân xem hồ sơ bệnh án cá nhân, MFA chỉ bắt buộc khi truy cập từ thiết bị/vị trí chưa từng ghi nhận; với bác sĩ truy cập hồ sơ nhiều bệnh nhân, MFA bắt buộc mỗi phiên làm việc mới không phân biệt rủi ro.
- Khi điểm rủi ro cao bất thường (ví dụ đăng nhập từ hai quốc gia cách nhau vài phút), phải chặn và yêu cầu xác minh bổ sung mạnh hơn (ví dụ liên hệ hỗ trợ) thay vì chỉ hỏi lại OTP thông thường.
- Toàn bộ quyết định yêu cầu/miễn MFA của mỗi lần đăng nhập phải được log lại kèm lý do (rủi ro thấp/cao, tín hiệu nào kích hoạt) để phục vụ audit compliance.
- Nếu dịch vụ tính điểm rủi ro gặp lỗi hoặc timeout, hệ thống phải fail-safe về yêu cầu MFA đầy đủ, không được mặc định bỏ qua MFA vì lỗi hạ tầng.
