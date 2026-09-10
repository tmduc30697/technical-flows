# Sequence Diagram — Base: Transfer Money

Đây là **base**, flow chuyển tiền cơ bản: trừ tiền người gửi, cộng tiền người nhận, ghi thẳng vào DB — không có đảm bảo durability đa vị trí, không checksum, không atomic rõ ràng giữa 2 vế của giao dịch.

```mermaid
sequenceDiagram
    actor Customer
    participant BankApp as Banking App
    participant Ledger as Ledger Service
    participant DB as Database

    Customer->>BankApp: Transfer money (from_account, to_account, amount)
    BankApp->>Ledger: Process transfer
    Ledger->>DB: Debit from_account.balance
    Ledger->>DB: Credit to_account.balance
    Ledger->>DB: Insert TRANSACTION record
    DB-->>Ledger: Ghi xong (chỉ 1 bản, không đảm bảo đa vị trí)
    Ledger-->>BankApp: Transfer completed
    BankApp-->>Customer: Giao dịch thành công

    Note over Ledger,DB: Nếu crash phần cứng đúng giữa lúc ghi debit và credit, dữ liệu có thể lệch không toàn-hoặc-không
```
