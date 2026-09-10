# Sequence Diagram — Enhance: Cleanup Raw Recording

Đây là **enhance**, flow mới xử lý yêu cầu cuối cùng của đề bài: dọn dữ liệu ghi hình thô (segment gốc) sau khi VOD đã tạo thành công và được xác nhận phát được, tránh giữ trùng lặp dữ liệu gây tốn chi phí lưu trữ.

```mermaid
sequenceDiagram
    participant Scheduler as Cleanup Scheduler
    participant PostProc as VOD Post-Processing Service
    participant VODStore as VOD Storage
    participant Storage as Segment Storage

    Scheduler->>PostProc: Check VOD đã fully ready và đã được xác nhận phát được
    PostProc->>VODStore: Query VOD.status, available_at
    VODStore-->>PostProc: VOD ready, playback confirmed

    alt VOD confirmed playable
        PostProc->>Storage: Delete raw SEGMENT files của stream này
        Storage-->>PostProc: Segments deleted
        PostProc->>VODStore: Set raw_cleaned_up = true
    else chưa xác nhận phát được
        PostProc-->>Scheduler: Skip, giữ nguyên raw recording, thử lại lần sau
    end
```
