# Observability cho pipeline xử lý giao dịch fintech cần audit chặt

**Hệ thống:** Một hệ thống xử lý chuyển tiền giữa các ví điện tử, mỗi giao dịch phải đi qua kiểm tra fraud, ghi ledger, và gửi thông báo — mọi bước phải có thể truy vết lại cho mục đích compliance.

**Vai trò của flow:** Tracing không chỉ dùng để debug performance mà còn đóng vai trò audit trail kỹ thuật — chứng minh được thứ tự và thời điểm chính xác của từng bước xử lý một giao dịch.

**Yêu cầu cụ thể:**
- Trace data liên quan tới giao dịch tài chính phải được lưu trữ với thời hạn (retention) dài hơn trace thông thường (theo yêu cầu compliance), và không được tự động xóa theo policy sampling/rotation mặc định của hệ thống observability.
- Đảm bảo tính toàn vẹn: span ghi lại thời điểm và kết quả của bước kiểm tra fraud không được sửa đổi được sau khi ghi (append-only), để tránh tình huống tranh chấp về việc "hệ thống đã check fraud hay chưa".
- Xử lý trường hợp một bước trong pipeline gọi ra một service bên thứ ba (ngân hàng đối tác) có độ trễ cao và không kiểm soát được — span phải ghi rõ đây là external call, tách biệt để không tính nhầm vào SLA nội bộ.
- Thiết kế cảnh báo tự động khi một giao dịch có trace "dừng bất thường" giữa chừng (ví dụ span "ledger-write" không có bước tiếp theo trong X giây) — dấu hiệu có thể là giao dịch bị treo cần can thiệp thủ công.
- Đảm bảo việc thu thập trace không làm tăng đáng kể latency của chính giao dịch (tracing overhead phải được đo và giữ dưới một ngưỡng phần trăm cụ thể so với latency gốc).
