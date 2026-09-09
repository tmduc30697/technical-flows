# Sequence Diagram - Base: trace-auto-deleted-by-retention-policy

Đây là flow **base** minh hoạ chính vấn đề mà đề bài cần giải quyết: vì trace của giao dịch tài chính dùng chung policy retention/sampling mặc định như mọi trace khác trong hệ thống, dữ liệu trace của 1 giao dịch bị rotation tự động xoá sau vài ngày; khi đội compliance cần tra soát lại một giao dịch cũ để phục vụ audit, dữ liệu đã không còn. Đây là lý do enhance cần retention riêng và đảm bảo toàn vẹn cho trace tài chính.

```mermaid
sequenceDiagram
    participant Tracer as Tracing Pipeline (policy mặc định)
    participant Store as Trace Storage
    actor Compliance as Đội compliance

    Tracer->>Store: Ghi trace TR1 của giao dịch chuyển tiền
    Store->>Store: Áp dụng retention mặc định 7 ngày như mọi trace khác
    Note over Store: Sau 7 ngày, job rotation tự động xoá trace TR1 theo policy chung
    Compliance->>Store: 30 ngày sau, yêu cầu tra soát trace TR1 để phục vụ audit tranh chấp
    Store-->>Compliance: Không tìm thấy, trace đã bị xoá theo policy mặc định
    Note over Compliance: Không còn bằng chứng kỹ thuật thời điểm/kết quả từng bước xử lý giao dịch
```
