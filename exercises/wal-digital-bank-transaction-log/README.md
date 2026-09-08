# WAL cho transaction log tài khoản ngân hàng số

**Hệ thống:** App ngân hàng số ghi transaction chuyển tiền, đây là dữ liệu tuyệt đối không được mất hoặc ghi sai dù có crash phần cứng.

**Vai trò của flow:** Mọi thay đổi số dư phải được ghi WAL với đảm bảo durability cao nhất (có thể ghi đồng thời ra nhiều thiết bị/đĩa) trước khi coi giao dịch là hoàn tất.

**Yêu cầu cụ thể:**
- WAL phải được ghi ra ít nhất 2 vị trí lưu trữ vật lý độc lập (ví dụ 2 đĩa hoặc 2 node) trước khi trả kết quả "giao dịch thành công", chống mất dữ liệu khi 1 đĩa hỏng ngay lúc crash.
- Mỗi entry log phải có checksum để phát hiện log bị corrupt (do lỗi đĩa) và từ chối replay entry đó, thay vì âm thầm apply dữ liệu sai vào số dư tài khoản.
- Recovery sau crash phải là toàn-hoặc-không cho mỗi transaction (atomic) — không có trường hợp một giao dịch chuyển tiền chỉ trừ tiền người gửi mà chưa cộng tiền người nhận sau khi phục hồi.
- Phải log đủ thông tin để có thể replay ra một audit trail đầy đủ, phục vụ đối soát với ngân hàng đối tác/cơ quan quản lý sau này.
- Đo lường: RPO (recovery point objective, lượng dữ liệu tối đa có thể mất) phải bằng 0 cho transaction đã ack thành công, và RTO (thời gian phục hồi) phải được benchmark cụ thể theo kích thước log thực tế.
