# Sequence Diagram — Enhance: Periodic Checkpoint

Đây là **enhance**, flow mới xử lý yêu cầu checkpoint định kỳ để giới hạn độ dài WAL cần replay, và phải an toàn nếu crash xảy ra giữa lúc đang checkpoint.

```mermaid
sequenceDiagram
    participant Scheduler as Checkpoint Scheduler
    participant Engine as Storage Engine
    participant DataStruct as In-memory / B-tree Structure
    participant WAL as WAL (disk)

    Scheduler->>Engine: Trigger checkpoint định kỳ
    Engine->>Engine: Tạo CHECKPOINT mới, status = in_progress
    Engine->>DataStruct: Flush toàn bộ trạng thái hiện tại xuống snapshot bền vững
    DataStruct-->>Engine: Snapshot ghi xong

    alt crash giữa lúc checkpoint đang ghi snapshot
        Note over Engine,WAL: Checkpoint dở dang không được coi là hợp lệ
        Note over Engine,WAL: Khi recovery, engine bỏ qua checkpoint chưa completed và dùng checkpoint completed gần nhất trước đó
    else checkpoint hoàn tất
        Engine->>Engine: Set CHECKPOINT.status = completed, up_to_lsn = lsn hiện tại
        Engine->>WAL: Có thể cắt bớt/archive WAL_ENTRY trước up_to_lsn
    end
```
