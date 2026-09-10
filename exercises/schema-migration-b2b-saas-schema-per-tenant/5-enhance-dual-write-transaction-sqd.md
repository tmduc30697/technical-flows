# Sequence Diagram — Enhance: Dual-Write Transaction

Đây là **enhance**, flow mới cho giai đoạn dual-write trước cutover: mọi write cho tenant đang migrate phải ghi cả 2 schema, có xử lý rõ khi một bên thất bại (retry hoặc rollback toàn bộ, không để một bên có một bên không). Flow này cũng thể hiện yêu cầu đọc luôn từ old schema (nguồn sự thật) trong giai đoạn này.

```mermaid
sequenceDiagram
    actor HRUser as HR User
    participant App as HR App Service
    participant Outbox as Outbox/Saga Coordinator
    participant OldDB as Old Schema (source of truth)
    participant NewDB as New Schema (backfill target)

    HRUser->>App: Update employee salary (tenant in migrating state)
    App->>Outbox: Begin dual-write transaction

    Outbox->>OldDB: Write update (source of truth)
    OldDB-->>Outbox: Write succeeded

    Outbox->>NewDB: Write same update
    alt new schema write succeeds
        NewDB-->>Outbox: Write succeeded
        Outbox->>Outbox: Mark DUAL_WRITE_LOG (old=success, new=success, resolution=committed)
        Outbox-->>App: Dual-write complete
    else new schema write fails
        NewDB-->>Outbox: Write failed
        Outbox->>Outbox: Create DUAL_WRITE_LOG (old=success, new=failed)
        Outbox->>Outbox: Retry write to new schema with backoff
        alt retry succeeds within limit
            Outbox->>NewDB: Retry write
            NewDB-->>Outbox: Write succeeded
            Outbox->>Outbox: Mark DUAL_WRITE_LOG resolution=recovered_by_retry
        else retry exhausted
            Outbox->>Outbox: Mark DUAL_WRITE_LOG resolution=needs_backfill_reconcile
            Note over Outbox: Old schema remains correct and authoritative
            Note over Outbox: new schema gap queued for the backfill job to fix later
        end
    end

    App-->>HRUser: Salary updated
    Note over App,OldDB: All reads during migrating state still go to Old Schema only
```
