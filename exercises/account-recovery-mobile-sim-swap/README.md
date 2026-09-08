# App di động xử lý khôi phục tài khoản khi số điện thoại đăng ký bị SIM-swap

**Hệ thống:** Một app di động dùng số điện thoại làm định danh chính và kênh nhận OTP để đăng nhập/khôi phục tài khoản.

**Vai trò của flow:** Xử lý rủi ro khi kẻ tấn công đã chiếm được quyền kiểm soát số điện thoại đăng ký (SIM-swap) và dùng chính OTP hợp lệ để chiếm tài khoản, thay vì mặc định coi việc nhận đúng OTP là bằng chứng danh tính tuyệt đối.

**Yêu cầu cụ thể:**
- Không được coi việc nhận đúng OTP qua SMS là yếu tố xác minh danh tính duy nhất/đủ mạnh cho các hành động nhạy cảm (ví dụ đổi email liên kết, đổi phương thức khôi phục) — các hành động này cần thêm một factor độc lập với số điện thoại, chẳng hạn thiết bị đã được tin cậy từ trước.
- Khi số điện thoại đăng ký vừa được đổi SIM (suy luận được từ tín hiệu nhà mạng nếu có, hoặc từ việc OTP đột ngột nhận trên một thiết bị hoàn toàn khác lịch sử trước đó), hệ thống nên áp dụng một khoảng cooling-off trước khi cho phép dùng số đó để thực hiện các thay đổi quan trọng trên tài khoản.
- Nếu tài khoản còn một thiết bị đã được tin cậy trước đó đang đăng nhập, các yêu cầu khôi phục hoặc đổi thông tin quan trọng đến từ số điện thoại vừa nhận OTP mới nên được đối chiếu và cảnh báo tới thiết bị tin cậy đó trước khi thực thi, để chủ tài khoản thật có cơ hội phát hiện và chặn kịp thời.
- Cần phân biệt giữa người dùng thật đổi điện thoại hợp lệ (mua máy mới, đổi SIM chính chủ) và kẻ tấn công SIM-swap — không thể chặn cứng mọi thay đổi đến từ số điện thoại mới vì sẽ khóa nhầm người dùng hợp lệ, nên cần kết hợp thêm tín hiệu khác như lịch sử thiết bị hoặc email dự phòng để quyết định mức độ tin cậy phù hợp.
- Khi phát hiện dấu hiệu SIM-swap sau khi đã xảy ra (chủ tài khoản thật báo cáo bị chiếm quyền), phải có luồng khôi phục ưu tiên không còn phụ thuộc vào số điện thoại đó nữa (ví dụ dựa vào email dự phòng đã xác minh từ trước hoặc xác minh danh tính thủ công), đồng thời khóa ngay các thay đổi mà kẻ tấn công đã thực hiện trong thời gian chiếm quyền.
