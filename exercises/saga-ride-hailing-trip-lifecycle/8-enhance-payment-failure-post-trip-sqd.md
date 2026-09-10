# Sequence Diagram — Enhance: Payment Failure After Trip Completion

Đây là **enhance**, flow mới xử lý trường hợp bước thanh toán cuối saga thất bại (thẻ bị từ chối) sau khi chuyến đã hoàn thành thực tế. Flow này thể hiện yêu cầu quan trọng: không compensate bằng cách "hủy chuyến" vì chuyến không thể hủy ngược, mà tách thành trạng thái riêng với retry/thu nợ và vẫn ghi nhận thu nhập tạm cho tài xế.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Payment as Payment Service
    actor Driver
    actor Rider

    Orchestrator->>Orchestrator: TRIP already status=completed (trip physically happened)
    Orchestrator->>Payment: Charge rider for trip
    Payment-->>Orchestrator: Payment failed, card declined

    Orchestrator->>Orchestrator: Create SAGA_STEP (payment_settled, status=failed)
    Note over Orchestrator: Do NOT compensate by cancelling the trip, it already happened

    Orchestrator->>Orchestrator: Mark TRIP payment_status=completed_pending_payment
    Orchestrator->>Orchestrator: Create PROVISIONAL_EARNING for driver (status=provisional)
    Orchestrator-->>Driver: Trip earning credited provisionally, will finalize once payment collected

    Orchestrator->>Orchestrator: Create PAYMENT_RETRY_ATTEMPT (attempt_number=1), schedule retry with backoff
    Orchestrator->>Payment: Retry charge later
    Payment-->>Orchestrator: Payment succeeded on retry

    Orchestrator->>Orchestrator: Mark PROVISIONAL_EARNING status=finalized
    Orchestrator-->>Rider: Charge finally collected, receipt sent
```
