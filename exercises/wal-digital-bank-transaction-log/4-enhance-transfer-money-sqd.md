# Sequence Diagram — Enhance: Transfer Money

Đây là **enhance**, flow chuyển tiền đã thay đổi so với base ([2-base-transfer-money-sqd.md](2-base-transfer-money-sqd.md)): trước khi trả kết quả "giao dịch thành công", cả hai `WAL_ENTRY` (debit và credit) phải được ghi thành công ra ít nhất 2 vị trí lưu trữ vật lý độc lập, thay vì ghi thẳng 1 bản vào DB như base.

```mermaid
sequenceDiagram
    actor Customer
    participant BankApp as Banking App
    participant Ledger as Ledger Service
    participant WAL_A as WAL Disk/Node A
    participant WAL_B as WAL Disk/Node B
    participant DB as Database

    Customer->>BankApp: Transfer money (from_account, to_account, amount)
    BankApp->>Ledger: Process transfer

    Ledger->>Ledger: Tạo WAL_ENTRY debit + WAL_ENTRY credit (cùng transaction_id, kèm checksum)
    par ghi song song 2 vị trí độc lập
        Ledger->>WAL_A: Ghi + fsync WAL_ENTRY
        WAL_A-->>Ledger: Ghi thành công
    and
        Ledger->>WAL_B: Ghi + fsync WAL_ENTRY
        WAL_B-->>Ledger: Ghi thành công
    end

    alt cả 2 vị trí ghi thành công
        Ledger->>DB: Apply debit + credit vào balance
        Ledger-->>BankApp: Transfer completed
        BankApp-->>Customer: Giao dịch thành công
    else một trong hai vị trí ghi thất bại
        Ledger-->>BankApp: Transfer failed, không ack thành công
        BankApp-->>Customer: Giao dịch thất bại, yêu cầu thử lại
    end
```
