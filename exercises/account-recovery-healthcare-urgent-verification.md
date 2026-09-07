# Nền tảng y tế cân bằng giữa khôi phục tài khoản nhanh cho tình huống cấp cứu và xác minh danh tính chặt

**Hệ thống:** Một nền tảng y tế cho bệnh nhân xem hồ sơ bệnh án, kết quả xét nghiệm, và đặt lịch khám; dữ liệu thuộc loại nhạy cảm cao và chịu quy định bảo vệ dữ liệu sức khỏe nghiêm ngặt.

**Vai trò của flow:** Xử lý mâu thuẫn giữa việc bệnh nhân có thể cần truy cập gấp (ví dụ đang ở phòng cấp cứu cần xem lịch sử dị ứng thuốc/bệnh nền) và yêu cầu xác minh danh tính đủ chặt để không cho người không liên quan xem hồ sơ sức khỏe của người khác.

**Yêu cầu cụ thể:**
- Cần phân tầng mức độ khẩn cấp của yêu cầu khôi phục: khôi phục thông thường (quên mật khẩu) đi qua luồng xác minh tiêu chuẩn, nhưng có một luồng khẩn cấp y tế riêng cho phép nhân viên y tế đang cấp cứu yêu cầu quyền xem tạm thời có kiểm soát mà không phải chờ hoàn tất toàn bộ bước xác minh danh tính đầy đủ của bệnh nhân.
- Luồng khẩn cấp phải giới hạn phạm vi truy cập được cấp, ví dụ chỉ xem dị ứng và thuốc đang dùng chứ không cấp toàn quyền chỉnh sửa hồ sơ hoặc xem thông tin tài chính, và tự động hết hạn sau một khoảng thời gian ngắn thay vì tồn tại như một quyền truy cập lâu dài.
- Mọi lần sử dụng luồng khẩn cấp phải được ghi log chi tiết và tự động kích hoạt review sau sự việc — ai đã cấp quyền, dựa trên căn cứ gì, đã xem gì — vì đây là lối tắt bỏ qua xác minh chuẩn nên cần cơ chế giám sát để phát hiện lạm dụng, không thể chỉ tin tưởng vào lý do "trường hợp khẩn cấp" mà không kiểm tra lại.
- Với khôi phục tài khoản thông thường không khẩn cấp, vì dữ liệu sức khỏe nhạy cảm hơn dữ liệu thông thường, mức xác minh danh tính phải cao hơn chuẩn phổ biến (không chỉ dựa vào email), chấp nhận đánh đổi tốc độ chậm hơn để giảm rủi ro lộ hồ sơ bệnh án cho người không phải chủ tài khoản.
- Phải xử lý được trường hợp người thân hoặc người giám hộ hợp pháp cần khôi phục quyền truy cập thay cho bệnh nhân không tự thao tác được, ví dụ bệnh nhân bất tỉnh, trẻ em, hoặc người cao tuổi — quy trình ủy quyền này cần xác minh mối quan hệ hợp pháp một cách có kiểm soát và tách biệt hẳn khỏi luồng tự khôi phục thông thường của chính chủ tài khoản.
