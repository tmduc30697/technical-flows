# Sequence Diagram — Enhance: Crash Recovery Atomic Replay

Đây là **enhance**, flow mới hoàn toàn: recovery sau crash phải toàn-hoặc-không cho mỗi transaction, và mỗi entry phải qua kiểm tra checksum trước khi apply, không âm thầm ghi sai số dư.

```mermaid
sequenceDiagram
    participant Ledger as Ledger Service
    participant WAL_A as WAL Disk/Node A
    participant WAL_B as WAL Disk/Node B
    participant DB as Database

    Ledger->>Ledger: Khởi động lại sau crash phần cứng
    Ledger->>WAL_A: Đọc toàn bộ WAL_ENTRY
    Ledger->>WAL_B: Đọc toàn bộ WAL_ENTRY (đối chiếu 2 bản)

    loop mỗi TRANSACTION theo transaction_id
        Ledger->>Ledger: Kiểm tra checksum của cả entry debit và credit

        alt cả 2 entry hợp lệ và đầy đủ
            Ledger->>DB: Apply cả debit và credit (atomic, toàn bộ hoặc không gì cả)
        else 1 trong 2 entry corrupt hoặc thiếu (ví dụ chỉ có debit, chưa có credit)
            Ledger->>Ledger: Từ chối replay transaction này, không apply phần nào
            Ledger->>Ledger: Đánh dấu transaction cần điều tra thủ công
        end
    end

    Ledger-->>Ledger: Số dư tất cả account nhất quán, không có giao dịch dở dang
```
