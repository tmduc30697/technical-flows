# Sequence Diagram — Enhance: Disconnect During Pickup, Grace Window

Đây là **enhance**, flow mới xử lý trường hợp app khách mất kết nối đúng lúc tài xế đang trên đường đón khách. Flow này thể hiện yêu cầu "không tự hủy chuyến ngay lập tức, cần cửa sổ chờ hợp lý để khách reconnect" — tránh hủy sai khi tài xế đã di chuyển gần tới nơi.

```mermaid
sequenceDiagram
    actor Rider
    participant Orchestrator as Saga Orchestrator
    actor Driver

    Note over Rider,Orchestrator: Rider's app loses connection between matched and picked_up

    Orchestrator->>Orchestrator: Detect missed heartbeat from rider app
    Orchestrator->>Orchestrator: Create DISCONNECT_EVENT (party=rider, disconnected_at)
    Orchestrator->>Orchestrator: Start grace window timer, do not cancel trip yet
    Orchestrator-->>Driver: Continue to pickup location, no change

    alt rider reconnects within grace window
        Rider->>Orchestrator: App reconnects
        Orchestrator->>Orchestrator: Update DISCONNECT_EVENT (reconnected_at, resolution=resumed)
        Orchestrator-->>Rider: Trip continues normally
    else grace window expires with no reconnect
        Orchestrator->>Orchestrator: Update DISCONNECT_EVENT (resolution=cancelled)
        Orchestrator->>Orchestrator: Compensate, cancel trip, create DRIVER_EXCLUSION not applied here
        Orchestrator-->>Driver: Trip cancelled due to rider unreachable, compensation for time driven
    end
```
