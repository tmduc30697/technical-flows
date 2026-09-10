# Sequence Diagram — Enhance: Batch Payout Parallel Sellers

Đây là **enhance**, flow mới thể hiện yêu cầu "batch xử lý hàng nghìn seller cùng lúc, một seller lỗi không được làm dừng/rollback toàn bộ batch". Mỗi seller có `SAGA_INSTANCE` độc lập trong cùng `PAYOUT_BATCH`, chạy song song và không phụ thuộc lẫn nhau.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Batch as Payout Batch Service
    participant Orchestrator as Saga Orchestrator

    Ops->>Batch: Trigger PAYOUT_BATCH for period
    Batch->>Batch: Create PAYOUT per eligible seller

    par seller A saga
        Batch->>Orchestrator: Start SAGA_INSTANCE for seller A's payout
        Orchestrator-->>Batch: Completed successfully
    and seller B saga (invalid bank account)
        Batch->>Orchestrator: Start SAGA_INSTANCE for seller B's payout
        Orchestrator-->>Batch: Failed, transfer rejected by partner
        Orchestrator->>Orchestrator: Compensate, refund seller B's balance
        Orchestrator->>Orchestrator: Mark seller B's PAYOUT status=retry_needed
    and seller C saga
        Batch->>Orchestrator: Start SAGA_INSTANCE for seller C's payout
        Orchestrator-->>Batch: Completed successfully
    end

    Batch->>Batch: Mark PAYOUT_BATCH as completed, seller B flagged separately
    Batch-->>Ops: Batch summary, only seller B needs manual retry
    Note over Batch: Sellers A and C already paid, unaffected by seller B's failure
```
