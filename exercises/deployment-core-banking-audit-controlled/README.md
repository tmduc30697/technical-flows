# Triển khai thay đổi cho hệ thống core banking cần kiểm soát chặt và có thể audit

**Hệ thống:** Một hệ thống core banking xử lý giao dịch tài khoản, chịu quy định chặt về thay đổi hệ thống (change management, audit), rất hạn chế rủi ro downtime hoặc lỗi dữ liệu.

**Vai trò của flow:** Canary/blue-green ở đây phải đi kèm với quy trình phê duyệt, khả năng audit đầy đủ, và mức độ thận trọng cao hơn nhiều so với các hệ thống web thông thường.

**Yêu cầu cụ thể:**
- Mọi giai đoạn của canary rollout (bắt đầu, tăng tỷ lệ, rollback) phải được ghi lại đầy đủ trong audit log không thể sửa đổi (ai thực hiện, khi nào, tỷ lệ traffic tại mỗi thời điểm), phục vụ yêu cầu kiểm toán và tuân thủ quy định ngành ngân hàng.
- Giai đoạn canary ban đầu chỉ nên áp dụng cho các giao dịch có giá trị thấp hoặc rủi ro thấp (ví dụ chỉ giao dịch xem số dư, chưa cho giao dịch chuyển tiền) trước khi mở rộng dần sang các loại giao dịch quan trọng hơn, thay vì chia đều theo tỷ lệ traffic ngẫu nhiên như hệ thống thông thường.
- Đảm bảo mọi thay đổi liên quan tới việc tính toán số dư/giao dịch phải có khả năng đối soát song song giữa phiên bản cũ và mới trong giai đoạn canary (chạy shadow — gửi cùng request tới cả hai phiên bản, so sánh kết quả, nhưng chỉ trả về kết quả từ phiên bản đang chính thức) để phát hiện sai lệch trước khi thực sự chuyển traffic.
- Định nghĩa quy trình rollback không chỉ về mã nguồn mà cả về dữ liệu — nếu phiên bản mới đã ghi dữ liệu theo định dạng mới trước khi phát hiện lỗi cần rollback, phải có kế hoạch xử lý dữ liệu đó khi quay lại phiên bản cũ, tránh mất tính nhất quán số liệu tài chính.
- Yêu cầu bước phê duyệt thủ công (không tự động hoàn toàn) trước khi chuyển từ mỗi giai đoạn canary sang giai đoạn tiếp theo, có ít nhất hai người xác nhận (four-eyes principle) trước khi mở rộng lên 100% traffic.
