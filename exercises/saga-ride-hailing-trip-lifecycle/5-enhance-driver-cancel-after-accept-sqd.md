# Sequence Diagram — Enhance: Driver Cancel After Accept, Rematch

Đây là **enhance**, flow mới xử lý trường hợp tài xế đã "nhận cuốc" nhưng tự hủy trước khi đón khách. Flow này thể hiện yêu cầu compensate trả chuyến đi về trạng thái "tìm tài xế" và loại trừ tài xế vừa hủy khỏi vòng matching tiếp theo cho cùng chuyến, tránh matching lặp lại ngay với tài xế vừa từ chối.

```mermaid
sequenceDiagram
    actor Driver
    participant Orchestrator as Saga Orchestrator
    participant Matching as Matching Service
    actor Rider

    Driver->>Orchestrator: Cancel trip after accepting, before pickup
    Orchestrator->>Orchestrator: Create SAGA_STEP (driver_cancelled, status=failed)

    Orchestrator->>Orchestrator: Compensate, revert TRIP status to requested
    Orchestrator->>Orchestrator: Create DRIVER_EXCLUSION (trip_id, driver_id, reason=self_cancelled)

    Orchestrator->>Matching: Find new driver, excluding DRIVER_EXCLUSION list for this trip
    Matching-->>Orchestrator: New driver found (not the one who cancelled)

    Orchestrator->>Orchestrator: Create SAGA_STEP (rematched), log event
    Orchestrator-->>Rider: Notify, still finding you a driver
```
