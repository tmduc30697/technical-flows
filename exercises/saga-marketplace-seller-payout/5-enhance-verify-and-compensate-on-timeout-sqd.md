# Sequence Diagram — Enhance: Verify And Compensate On Timeout

Đây là **enhance**, flow mới xử lý trường hợp bước gọi đối tác timeout sau khi số dư seller đã bị trừ. Flow này thể hiện yêu cầu quan trọng nhất của đề bài: không tự động compensate ngay, mà phải verify trạng thái thực qua API đối chiếu/callback trước, tránh vừa mất tiền chuyển thật vừa hoàn số dư ảo (double-pay).

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Balance as Seller Balance Service
    participant Partner as Payment Partner

    Orchestrator->>Partner: Initiate transfer (idempotency_key=payout_id)
    Partner-->>Orchestrator: Request timed out, unknown result

    Orchestrator->>Orchestrator: Create SAGA_STEP (initiate_transfer, status=uncertain)
    Note over Orchestrator: Do NOT compensate balance yet, actual transfer state is unknown

    Orchestrator->>Partner: Query transfer status by idempotency_key
    Partner-->>Orchestrator: Transfer status found

    alt transfer actually succeeded
        Orchestrator->>Orchestrator: Update PARTNER_TRANSFER verified_status=succeeded
        Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed, no compensation
    else transfer confirmed failed by partner
        Orchestrator->>Orchestrator: Update PARTNER_TRANSFER verified_status=failed
        Orchestrator->>Balance: Compensate, refund deducted amount
        Balance-->>Orchestrator: Balance restored
        Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=compensated
    else still no definitive answer
        Orchestrator->>Orchestrator: Schedule re-check with backoff, keep saga in uncertain state
    end
```
