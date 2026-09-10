# ERD — Enhance (sau khi có WAL durability cao)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — ghi WAL ra ít nhất 2 vị trí độc lập trước khi ack, checksum để phát hiện corrupt, recovery atomic toàn-hoặc-không, audit trail đầy đủ, và đo RPO/RTO. So với base, `TRANSACTION` nay chỉ được coi là hoàn tất sau khi cả hai `WAL_ENTRY` (debit + credit) đã ghi thành công ở `WAL_REPLICA_WRITE` trên tối thiểu 2 vị trí vật lý.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "debited/credited by"
    TRANSACTION ||--|{ WAL_ENTRY : "logged as"
    WAL_ENTRY ||--|{ WAL_REPLICA_WRITE : "persisted to"

    ACCOUNT {
        string account_id PK
        string owner_name
        decimal balance
        string status
    }

    TRANSACTION {
        string transaction_id PK
        string from_account_id FK
        string to_account_id FK
        decimal amount
        string status
        datetime created_at
    }

    WAL_ENTRY {
        bigint lsn PK
        string transaction_id FK
        string account_id FK
        string entry_type
        decimal amount
        string checksum
        string status
        datetime written_at
    }

    WAL_REPLICA_WRITE {
        string write_id PK
        bigint lsn FK
        string storage_location
        string status
        datetime synced_at
    }
```
