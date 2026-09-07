# Thanh toán thương mại xuyên biên giới với chuyển đổi tiền tệ và callback đa múi giờ

**Hệ thống:** Một sàn thương mại xuyên biên giới cho phép khách thanh toán bằng tiền tệ nội địa, hệ thống tự chuyển đổi sang tiền tệ của seller, callback báo kết quả từ đối tác thanh toán quốc tế có thể đến sau nhiều giờ.

**Vai trò của flow:** Flow phải khóa đúng tỷ giá tại thời điểm giao dịch, xử lý callback đến rất trễ (đôi khi hơn 24 giờ) mà không làm sai lệch số tiền do tỷ giá đã biến động, và đối soát được số tiền 2 loại tiền tệ.

**Yêu cầu cụ thể:**
- Tại thời điểm khách bấm thanh toán, tỷ giá quy đổi phải được "khóa" (snapshot) và lưu lại cùng giao dịch — callback xử lý kết quả sau đó (dù đến trễ bao lâu) phải dùng đúng tỷ giá đã khóa này để tính số tiền cuối cùng, không lấy tỷ giá hiện tại tại thời điểm callback đến.
- Mô tả cụ thể: callback báo giao dịch thành công đến sau 20 giờ, trong khoảng thời gian đó tỷ giá đã biến động 2%; hệ thống ghi nhận doanh thu cho seller phải dùng đúng tỷ giá đã khóa lúc khách thanh toán, không phải tỷ giá tại lúc nhận callback — có test cụ thể xác nhận số tiền quy đổi khớp với tỷ giá khóa ban đầu.
- Nếu callback báo thất bại đến sau khi hệ thống đã tạm ghi nhận đơn hàng ở trạng thái "đã thanh toán" (do quá lâu chưa nhận được callback nên có luồng dự phòng tự chuyển trạng thái theo polling), phải có cơ chế đảo ngược đúng: hủy đơn, hoàn tiền theo đúng tỷ giá đã khóa ban đầu (không phải tỷ giá hiện tại lúc hoàn tiền).
- Do đối tác thanh toán quốc tế ở múi giờ khác, các mốc "theo ngày" trong đối soát (ví dụ báo cáo cuối ngày) phải được chuẩn hóa về 1 múi giờ tham chiếu duy nhất (ví dụ UTC) khi so khớp dữ liệu giữa 2 hệ thống, tránh lệch báo cáo do 1 bên tính theo giờ địa phương.
- Có luồng đối soát riêng theo từng loại tiền tệ (không gộp chung), so khớp cả số tiền gốc (tiền khách trả) và số tiền quy đổi (tiền seller nhận), phát hiện các giao dịch có sai lệch tỷ giá bất thường để điều tra riêng.
