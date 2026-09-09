# Base sequence — Live broadcast distribution

Đây là **base**, flow "Phân phối sóng trực tiếp" ở trạng thái hiện tại: một nguồn ingest duy nhất, transcode rồi fan-out tới các edge node, mỗi edge tự buffer theo điều kiện mạng riêng. Flow này là tiền đề cho yêu cầu 1 (độ trễ đồng nhất) và yêu cầu 3 (backup ingest) vì cả hai đều tác động trực tiếp lên đúng pipeline này.

```mermaid
sequenceDiagram
    actor Producer as Đơn vị sản xuất
    participant Ingest as Ingest Source (duy nhất)
    participant Transcoder
    participant EdgeA as Edge Node A (gần producer)
    participant EdgeB as Edge Node B (xa hơn)
    actor ViewerA as Viewer trên Edge A
    actor ViewerB as Viewer trên Edge B

    Producer->>Ingest: Đẩy luồng RTMP gốc
    Ingest->>Transcoder: Chuyển tiếp luồng gốc
    Transcoder->>Transcoder: Transcode sang các bitrate/segment
    Transcoder->>EdgeA: Đẩy segment mới
    Transcoder->>EdgeB: Đẩy segment mới
    EdgeA->>EdgeA: Buffer theo cấu hình riêng của edge A
    EdgeB->>EdgeB: Buffer theo cấu hình riêng của edge B (dài hơn do mạng yếu hơn)
    EdgeA-->>ViewerA: Phát segment (độ trễ ~3s so với thực tế)
    EdgeB-->>ViewerB: Phát segment (độ trễ ~9s so với thực tế)
    Note over ViewerA,ViewerB: Hai viewer xem cùng một pha bóng lệch nhau vài giây, không có cơ chế nào ép đồng nhất độ trễ giữa các edge
```
