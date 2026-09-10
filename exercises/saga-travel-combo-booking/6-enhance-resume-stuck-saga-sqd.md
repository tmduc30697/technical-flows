# Sequence Diagram — Enhance: Detect And Resume Stuck Saga

Đây là **enhance**, flow mới phát hiện saga "kẹt" quá lâu ở một bước (ví dụ worker xử lý bước đó chết) và tự động resume hoặc alert vận hành. Flow này thể hiện yêu cầu giám sát chủ động thay vì để khách chờ vô thời hạn không rõ trạng thái combo booking của mình.

```mermaid
sequenceDiagram
    participant Monitor as Saga Health Monitor
    participant Orchestrator as Saga Orchestrator
    participant Car as Car Rental Provider (Partner)
    participant Alert as Alert Service
    actor Ops as Ops Team

    Monitor->>Monitor: Scan SAGA_INSTANCE where last_progress_at exceeds step's timeout_seconds
    Monitor->>Monitor: Found a saga stuck at step car_held for too long

    Monitor->>Orchestrator: Attempt automatic resume, re-check step status
    Orchestrator->>Car: Query actual reservation status by combo_id
    Car-->>Orchestrator: Reservation status found (confirmed)

    alt step actually completed, just not acknowledged
        Orchestrator->>Orchestrator: Mark SAGA_STEP (car_held, status=completed), continue saga
        Orchestrator-->>Monitor: Saga resumed automatically
    else step truly failed or worker unresponsive
        Monitor->>Alert: Escalate, saga stuck beyond auto-resume capability
        Alert-->>Ops: Alert, combo_id needs manual intervention
        Ops->>Orchestrator: Manually inspect and trigger compensate or retry
    end
```
