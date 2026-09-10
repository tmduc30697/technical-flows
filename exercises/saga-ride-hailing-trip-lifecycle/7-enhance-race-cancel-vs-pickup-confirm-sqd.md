# Sequence Diagram — Enhance: Race, Cancel vs Pickup Confirm

Đây là **enhance**, flow mới xử lý race condition khi khách hủy chuyến đúng lúc tài xế bấm "đã đón khách" gần như đồng thời từ hai thiết bị độc lập. Flow này thể hiện yêu cầu có luật ưu tiên rõ ràng (sự kiện nào server nhận trước theo thời điểm xử lý được coi là hợp lệ) và bên thua phải nhận đúng thông báo trạng thái thực tế.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Orchestrator as Saga Orchestrator

    par near-simultaneous events
        Rider->>Orchestrator: Cancel trip (arrives at t1)
    and
        Driver->>Orchestrator: Confirm picked up (arrives at t2)
    end

    Orchestrator->>Orchestrator: Compare server-received timestamps, t1 arrived first
    Orchestrator->>Orchestrator: Rider's cancel event is treated as authoritative

    Orchestrator->>Orchestrator: Create SAGA_STEP (trip_cancelled, status=completed)
    Orchestrator->>Orchestrator: Reject the picked_up event as stale, log SAGA_EVENT (conflict_resolved, winner=cancel)

    Orchestrator-->>Rider: Trip cancelled as requested
    Orchestrator-->>Driver: Trip was cancelled by rider just before your pickup confirmation, no charge applies
    Note over Orchestrator: Both apps now show the same resolved state, no contradiction
```
