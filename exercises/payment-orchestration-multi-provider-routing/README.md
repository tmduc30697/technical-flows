# Cổng thanh toán tổng hợp (payment orchestration) định tuyến qua nhiều nhà cung cấp

**Hệ thống:** Một hệ thống trung gian định tuyến giao dịch thanh toán qua nhiều nhà cung cấp khác nhau (theo tỷ lệ, theo phí, theo tình trạng sẵn sàng của từng nhà cung cấp) để tối ưu tỷ lệ thành công.

**Vai trò của flow:** Flow phải chọn đúng 1 nhà cung cấp cho mỗi giao dịch, xử lý callback trả về từ nhiều nhà cung cấp có format khác nhau, và không để 1 giao dịch bị xử lý bởi 2 nhà cung cấp cùng lúc do lỗi retry.

**Yêu cầu cụ thể:**
- Khi request thanh toán đầu tiên gửi tới nhà cung cấp A bị timeout (không rõ đã xử lý hay chưa ở phía A), hệ thống không được tự động gửi luôn request y hệt tới nhà cung cấp B mà không kiểm tra trước — phải có bước xác minh trạng thái với A (query status) hoặc đợi đủ thời gian timeout xác định trước khi coi là thất bại và chuyển hướng qua B, tránh giao dịch bị xử lý (charge tiền khách) ở cả A và B.
- Mỗi giao dịch chỉ được gán đúng 1 nhà cung cấp xử lý tại một thời điểm — dùng cơ chế lock/trạng thái atomic (ví dụ `status = routing` → `status = sent_to_provider_A`) để tránh 2 worker xử lý hàng đợi routing cùng lúc gán nhầm 1 giao dịch cho 2 nhà cung cấp khác nhau.
- Callback từ các nhà cung cấp khác nhau có format khác nhau nhưng phải được chuẩn hóa về cùng 1 mô hình trạng thái nội bộ trước khi ghi nhận, và mọi callback phải map lại đúng giao dịch gốc qua mã tham chiếu đã gửi kèm lúc khởi tạo (không dựa vào số tiền + thời gian để đoán, dễ nhầm khi trùng giá trị).
- Mô tả cụ thể race giữa callback thật từ nhà cung cấp A báo thất bại và cơ chế failover tự động của hệ thống đã quyết định chuyển sang B do quá thời gian chờ — nếu callback A đến ngay sau khi B đã bắt đầu xử lý, phải đảm bảo giao dịch không bị double-charge (chỉ 1 trong 2 được coi là hợp lệ, phải hủy/hoàn tiền phía còn lại nếu cả 2 vô tình đều thành công).
- Có dashboard theo dõi tỷ lệ thành công/lỗi theo từng nhà cung cấp real-time để tự động điều chỉnh tỷ lệ routing, và log đầy đủ lịch sử routing của mỗi giao dịch (đã thử nhà cung cấp nào, theo thứ tự nào, kết quả gì) phục vụ điều tra khi có khiếu nại double-charge.
