# Sequence Diagram - Enhance: partner-bank-external-call

Đây là flow **enhance mới**: bước `ledger-write` cần xác nhận số dư qua một ngân hàng đối tác bên ngoài trước khi ghi bút toán, span của lời gọi này được đánh dấu `is_external_call=true` kèm `external_partner_name`, để khi tổng hợp SLA nội bộ, độ trễ cao và không kiểm soát được của ngân hàng đối tác không bị tính nhầm vào SLA của chính hệ thống. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant Ledger as ledger-write
    participant Bank as Ngân hàng đối tác (external)
    participant Tracer as Tracing SDK

    Ledger->>Tracer: Bắt đầu span S3b, gắn tag is_external_call=true, external_partner_name=Partner-Bank-A
    Ledger->>Bank: Gọi xác nhận số dư trước khi ghi bút toán
    Note over Bank: Ngân hàng đối tác xử lý mất 3000ms, độ trễ không kiểm soát được
    Bank-->>Ledger: Xác nhận số dư hợp lệ
    Ledger->>Tracer: Kết thúc span S3b, duration=3000ms
    Ledger->>Ledger: Tiếp tục ghi bút toán vào ledger nội bộ (span S3, duration=15ms)
    Note over Tracer: Khi tính SLA nội bộ ledger-write, span S3b (external) được loại riêng, chỉ span S3 (nội bộ) được tính vào SLA
```
