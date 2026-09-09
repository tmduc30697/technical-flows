# Sequence Diagram - Enhance: process-transaction

Đây là flow **enhance** của `process-transaction` (so với base ở file `2-base-process-transaction-sqd.md`): ngay khi tạo trace, `transfer-gateway` gắn `is_compliance_trace=true` và tra `TRACE_RETENTION_POLICY` loại COMPLIANCE để áp retention dài hơn (ví dụ 7 năm) thay vì dùng retention mặc định 7 ngày, đảm bảo trace không bị job rotation/sampling mặc định xoá mất. Đáp ứng yêu cầu 1.

```mermaid
sequenceDiagram
    actor User
    participant Gateway as transfer-gateway
    participant Policy as Trace Retention Policy
    participant Fraud as fraud-check
    participant Ledger as ledger-write
    participant Notify as notification
    participant Store as Trace Storage

    User->>Gateway: Yêu cầu chuyển tiền
    Gateway->>Policy: Tra policy retention cho giao dịch tài chính
    Policy-->>Gateway: Trả về policy COMPLIANCE, retention_days=2555, deletion_exempt=true
    Gateway->>Gateway: Sinh trace_id=TR9, is_compliance_trace=true, tạo span S1
    Gateway->>Fraud: Kiểm tra fraud, header trace_id=TR9, parent_span_id=S1
    Fraud->>Fraud: Tạo span S2, ghi kết quả "PASS"
    Fraud-->>Gateway: Không phát hiện gian lận
    Gateway->>Ledger: Ghi nhận giao dịch vào ledger, header trace_id=TR9, parent_span_id=S1
    Ledger->>Ledger: Tạo span S3, ghi bút toán
    Ledger-->>Gateway: Ghi ledger thành công
    Gateway->>Notify: Gửi thông báo cho user, header trace_id=TR9, parent_span_id=S1
    Notify->>Notify: Tạo span S4, gửi notification
    Notify-->>Gateway: Đã gửi
    Gateway->>Store: Lưu trace TR9 kèm policy COMPLIANCE
    Store->>Store: Đánh dấu deletion_exempt=true, không đưa vào job rotation mặc định
    Gateway-->>User: Giao dịch thành công
```
