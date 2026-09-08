# Cách ly nghiêm ngặt cho nền tảng hồ sơ sức khỏe đa phòng khám

**Hệ thống:** Một nền tảng SaaS cho nhiều phòng khám tư nhân (mỗi phòng khám là một tenant) quản lý hồ sơ bệnh nhân, chịu yêu cầu bảo mật/compliance dữ liệu y tế cao hơn nhiều so với SaaS thông thường.

**Vai trò của flow:** Do mức độ nhạy cảm và hậu quả pháp lý nếu lộ dữ liệu chéo tenant (dữ liệu bệnh nhân của phòng khám A lộ sang phòng khám B), flow cách ly ở đây cần nhiều lớp bảo vệ hơn shared schema thông thường.

**Yêu cầu cụ thể:**
- Đánh giá và lựa chọn mô hình cách ly phù hợp với mức độ nhạy cảm (ví dụ database riêng theo tenant hoặc schema riêng theo tenant, thay vì chỉ cột `tenant_id` trong shared schema) — giải thích rõ trade-off giữa mức độ cách ly cao hơn và độ phức tạp vận hành/chi phí tăng thêm.
- Đảm bảo khóa mã hóa dữ liệu (encryption key) tại rest là riêng biệt theo từng tenant, để nếu một khóa bị lộ (hoặc một tenant yêu cầu xóa toàn bộ dữ liệu và hủy khóa — crypto shredding) không ảnh hưởng tới dữ liệu của các tenant khác.
- Xử lý yêu cầu xóa dữ liệu hoàn toàn của một tenant (khi phòng khám ngừng hợp tác, theo quy định pháp lý về quyền xóa dữ liệu) — phải xóa/hủy được toàn bộ dữ liệu liên quan trên mọi hệ thống lưu trữ (database chính, cache, backup, log) trong một khung thời gian xác định, có xác nhận hoàn tất.
- Thiết kế audit log chi tiết cho mọi lần truy cập vào hồ sơ bệnh nhân (ai xem, khi nào, dữ liệu nào), tách biệt audit log theo tenant và không cho tenant khác (dù là admin hệ thống ở mức thông thường) xem được audit log của tenant khác.
- Đảm bảo các tính năng dùng AI/phân tích dữ liệu tổng hợp (nếu có, ví dụ thống kê xu hướng bệnh) chỉ được huấn luyện/tính toán trên dữ liệu đã ẩn danh hóa đúng cách và không bao giờ để mô hình học được cách "nhớ" và lộ lại thông tin cụ thể của một bệnh nhân cho tenant khác truy vấn.
