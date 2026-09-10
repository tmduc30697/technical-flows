# Sequence Diagram — Enhance: Crash Recovery Replay

Đây là **enhance**, flow mới hoàn toàn: khi engine khởi động lại sau crash, replay WAL từ checkpoint gần nhất theo đúng thứ tự, phát hiện và bỏ qua entry bị torn write nhờ checksum.

```mermaid
sequenceDiagram
    participant Engine as Storage Engine
    participant WAL as WAL (disk)
    participant DataStruct as In-memory / B-tree Structure

    Engine->>Engine: Start up sau crash
    Engine->>WAL: Đọc CHECKPOINT gần nhất, lấy up_to_lsn
    Engine->>WAL: Đọc WAL_ENTRY tuần tự từ up_to_lsn trở đi

    loop mỗi WAL_ENTRY theo đúng thứ tự lsn
        Engine->>Engine: Verify checksum của entry
        alt checksum hợp lệ
            Engine->>DataStruct: Apply entry (key, value, operation)
        else checksum sai (torn write giữa lúc ghi dở)
            Engine->>Engine: Bỏ qua entry này và toàn bộ entry sau đó (log coi như kết thúc tại đây)
        end
    end

    Engine-->>Engine: Trạng thái tái tạo đúng thời điểm crash
    Engine-->>Engine: Sẵn sàng nhận request mới
```
