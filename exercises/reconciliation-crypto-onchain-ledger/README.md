# Sàn giao dịch crypto đối soát số dư giữa ví on-chain và ledger nội bộ

**Hệ thống:** Một sàn giao dịch crypto giữ tài sản khách hàng trong ví nóng/lạnh (on-chain) và duy trì một ledger nội bộ ghi số dư khả dụng của từng khách hàng để phục vụ giao dịch tức thời trên sàn.

**Vai trò của flow:** Đối soát định kỳ giữa tổng số dư thực tế trên các ví on-chain và tổng số dư khách hàng cộng lại trong ledger nội bộ, đảm bảo sàn luôn có đủ tài sản backing cho số dư đã ghi nhận.

**Yêu cầu cụ thể:**
- Đối soát phải tính đúng số dư on-chain tại một block height/thời điểm xác định và so khớp với snapshot ledger nội bộ tại đúng thời điểm tương ứng, tránh so sánh lệch thời điểm gây sai lệch giả.
- Giao dịch nạp/rút tiền on-chain đang chờ đủ số confirmation (chưa final) không được tính là "đã hoàn tất" trong ledger nội bộ cho đến khi đạt ngưỡng xác nhận an toàn theo từng loại tài sản, đối soát phải phân biệt rõ trạng thái pending và confirmed.
- Phát hiện được trường hợp tổng số dư ledger nội bộ vượt quá tổng tài sản thực có trên ví on-chain (dấu hiệu nghiêm trọng: lỗi hệ thống hoặc gian lận nội bộ) và phải cảnh báo khẩn cấp ngay lập tức, không chờ báo cáo đối soát định kỳ theo lịch thông thường.
- Xử lý đúng các giao dịch on-chain bị thay thế/hủy do phí gas thấp (replace-by-fee) hoặc bị đảo do fork mạng — không ghi nhận nhầm là hoàn tất khi giao dịch gốc thực chất không còn hiệu lực trên chuỗi.
- Toàn bộ quá trình đối soát và mọi sai lệch phát hiện phải được lưu trữ minh bạch, có khả năng phục vụ báo cáo proof-of-reserves cho khách hàng/cơ quan quản lý khi được yêu cầu.
