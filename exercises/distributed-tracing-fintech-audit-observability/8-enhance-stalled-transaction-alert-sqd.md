# Sequence Diagram - Enhance: stalled-transaction-alert

Đây là flow **enhance mới**: sau khi span `ledger-write` kết thúc, bước `notification` gặp sự cố nên không có span tiếp theo nào được ghi trong X giây; một tiến trình giám sát định kỳ phát hiện trace "dừng bất thường" giữa chừng và tự động tạo `STALL_ALERT` để người vận hành can thiệp thủ công, thay vì để giao dịch treo âm thầm không ai biết. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant Ledger as ledger-write
    participant Notify as notification (gặp sự cố)
    participant Monitor as Stall Detector (chạy định kỳ)
    participant Store as Trace Storage
    participant Alert as Alerting/Ops

    Ledger->>Store: Ghi span S3 "ledger-write" hoàn tất, trace_id=TR9
    Note over Notify: notification-service gặp sự cố (crash/treo), không nhận được message hoặc không xử lý được
    loop Mỗi chu kỳ giám sát (ví dụ mỗi 30 giây)
        Monitor->>Store: Kiểm tra các trace chưa kết thúc, span cuối cùng đã bao lâu
        Store-->>Monitor: Trace TR9, span cuối là S3 "ledger-write", đã 120 giây chưa có span tiếp theo
        alt Vượt ngưỡng X giây kỳ vọng (ví dụ 60 giây) mà chưa có span kế tiếp
            Monitor->>Store: Ghi STALL_ALERT cho trace TR9, last_span_id=S3, seconds_since_last_span=120
            Monitor->>Alert: Gửi cảnh báo giao dịch nghi bị treo cho người vận hành
            Alert-->>Alert: Người vận hành kiểm tra và can thiệp thủ công giao dịch TR9
        else Trong ngưỡng thời gian cho phép
            Monitor->>Monitor: Không cảnh báo, tiếp tục theo dõi chu kỳ sau
        end
    end
```
