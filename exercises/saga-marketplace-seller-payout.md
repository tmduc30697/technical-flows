# Chi trả tiền cho seller trên nền tảng marketplace sau khi đơn hàng hoàn tất

**Hệ thống:** Sàn marketplace có hàng nghìn seller, sau khi đơn hàng hoàn tất (khách nhận hàng, hết thời gian khiếu nại) hệ thống phải tính toán và chuyển tiền cho seller qua đối tác thanh toán bên ngoài.

**Vai trò của flow:** Saga điều phối chuỗi bước tính hoa hồng/phí, trừ vào số dư seller, và gọi đối tác thanh toán để chuyển tiền thực — đảm bảo nếu bước chuyển tiền thất bại giữa chừng, phần đã trừ được compensate đúng, không làm seller mất tiền oan hoặc bị trừ hai lần.

**Yêu cầu cụ thể:**
- Nếu bước gọi đối tác thanh toán trả về timeout (không rõ tiền đã chuyển hay chưa) sau khi số dư seller đã bị trừ trong hệ thống nội bộ, saga không được tự động compensate ngay (hoàn lại số dư) mà phải verify trạng thái thực tế qua API đối chiếu/callback của đối tác trước, tránh vừa mất tiền chuyển thật vừa hoàn số dư ảo dẫn tới double-pay.
- Compensate phần đã trừ (hoa hồng, phí) chỉ được thực hiện khi chắc chắn bước chuyển tiền chưa xảy ra hoặc đã thất bại dứt khoát được đối tác xác nhận; nếu chuyển tiền đã thành công nhưng saga crash trước khi ghi nhận, resume saga phải phát hiện qua truy vấn đối tác thay vì chạy lại từ đầu gây chuyển tiền trùng.
- Batch payout xử lý hàng nghìn seller cùng lúc — một seller lỗi (tài khoản ngân hàng không hợp lệ) không được làm dừng/rollback toàn bộ batch của các seller khác; saga cho mỗi seller phải độc lập và tiếp tục song song, chỉ đánh dấu payout của seller lỗi để retry riêng.
- Retry payout cho một seller sau lỗi tạm thời (đối tác timeout) phải idempotent ở phía đối tác thanh toán (dùng transaction reference cố định), tránh trường hợp retry vô tình tạo ra hai lệnh chuyển tiền thực cho cùng một khoản payout.
- Toàn bộ event của saga payout (tính hoa hồng, trừ số dư, gọi đối tác, xác nhận/compensate) phải lưu thành event log truy vấn được theo từng seller/đơn hàng, phục vụ đối soát tài chính và giải trình khi seller khiếu nại về số tiền nhận được.
