# Enhance sequence — Theo dõi rate limit theo IP/device fingerprint (log, không chặn cứng)

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: ghi log số request theo IP/device fingerprint để phát hiện nguồn bất thường trong sự kiện, nhưng không chặn cứng ngay lập tức để tránh ảnh hưởng người dùng thật dùng chung IP (mạng công ty/wifi công cộng).

```mermaid
sequenceDiagram
    actor Customer as Nhiều Customer (chung 1 IP công ty)
    participant App as Checkout Service
    participant DB as REQUEST_LOG store
    participant Analyst as Job phân tích bất thường (chạy nền)

    loop Mỗi request mua trong sự kiện
        Customer->>App: Gửi request mua (kèm IP, device_fingerprint)
        App->>DB: UPDATE REQUEST_LOG SET request_count=request_count+1 WHERE ip=... AND window_start=cửa_sổ_hiện_tại (upsert nếu chưa có)
        App->>App: Xử lý request bình thường theo flow purchase-sneaker, không chặn
    end

    Analyst->>DB: Định kỳ quét REQUEST_LOG tìm ip/fingerprint có request_count vượt ngưỡng bất thường
    DB-->>Analyst: Danh sách nguồn nghi vấn
    Analyst->>DB: UPDATE REQUEST_LOG SET flagged=true cho các nguồn vượt ngưỡng
    Note over Analyst,DB: flagged=true chỉ phục vụ phân tích/cảnh báo sau sự kiện, không tự động chặn request đang chạy, tránh chặn nhầm nhiều người dùng thật chia sẻ 1 IP công ty/wifi công cộng
    Analyst-->>App: (tuỳ chọn) Gửi cảnh báo cho đội vận hành để xem xét thủ công nếu cần chặn IP cụ thể
```
