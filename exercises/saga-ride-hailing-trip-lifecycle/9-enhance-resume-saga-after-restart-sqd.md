# Sequence Diagram — Enhance: Resume Saga After Service Restart

Đây là **enhance**, flow mới xử lý trường hợp Trip Service điều phối bị restart (ví dụ deploy) giữa lúc hàng nghìn chuyến đang diễn ra. Flow này thể hiện yêu cầu trạng thái từng chuyến phải persist đủ để không gửi lại thông báo cho bước đã qua từ trước, tránh gây nhiễu loạn app khách/tài xế.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator (new instance)
    actor Driver
    actor Rider

    Note over Orchestrator: Trip Service process restarted mid-deploy, thousands of trips in flight

    Orchestrator->>Orchestrator: Query all SAGA_INSTANCE with status=running
    Orchestrator->>Orchestrator: Load SAGA_STEP history per saga_id, find last completed step

    loop for each in-flight trip
        alt last completed step = matched
            Note over Orchestrator: Driver already notified before restart, do not resend "driver found"
            Orchestrator->>Orchestrator: Just resubscribe to pickup confirmation, no duplicate notification
        else last completed step = picked_up
            Note over Orchestrator: Pickup already confirmed before restart
            Orchestrator->>Orchestrator: Resume waiting for trip completion event only
        else step was in progress at crash time (uncertain)
            Orchestrator->>Orchestrator: Re-verify actual state with Driver/Rider apps before resuming
        end
    end

    Orchestrator-->>Driver: No redundant "trip found" or "picked up" notifications
    Orchestrator-->>Rider: Trip status remains consistent through the restart
```
