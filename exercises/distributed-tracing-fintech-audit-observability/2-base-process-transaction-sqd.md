# Sequence Diagram - Base: process-transaction

Đây là flow **base**: giao dịch chuyển tiền đi qua `fraud-check` -> `ledger-write` -> `notification`, đã có tracing thông thường (mỗi bước tạo 1 span gắn `trace_id`), nhưng span được lưu theo policy retention/sampling mặc định như mọi trace khác, có thể bị ghi đè, không phân biệt gọi ra bên ngoài. Flow này là nền để so sánh với enhance, nơi trace của giao dịch tài chính sẽ được nâng cấp thành audit trail.

```mermaid
sequenceDiagram
    actor User
    participant Gateway as transfer-gateway
    participant Fraud as fraud-check
    participant Ledger as ledger-write
    participant Notify as notification

    User->>Gateway: Yêu cầu chuyển tiền
    Gateway->>Gateway: Sinh trace_id=TR1 (retention mặc định 7 ngày), tạo span S1
    Gateway->>Fraud: Kiểm tra fraud, header trace_id=TR1, parent_span_id=S1
    Fraud->>Fraud: Tạo span S2, ghi kết quả "PASS"
    Fraud-->>Gateway: Không phát hiện gian lận
    Gateway->>Ledger: Ghi nhận giao dịch vào ledger, header trace_id=TR1, parent_span_id=S1
    Ledger->>Ledger: Tạo span S3, ghi bút toán
    Ledger-->>Gateway: Ghi ledger thành công
    Gateway->>Notify: Gửi thông báo cho user, header trace_id=TR1, parent_span_id=S1
    Notify->>Notify: Tạo span S4, gửi notification
    Notify-->>Gateway: Đã gửi
    Gateway-->>User: Giao dịch thành công
```
