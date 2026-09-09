# Enhance sequence — Live broadcast distribution (độ trễ đồng nhất)

Đây là **enhance** của flow đã có ở base "Live broadcast distribution". So với base, mỗi segment nay mang `video_pts` chung và có một Latency Coordinator ép mọi edge node cùng phát tại một mốc pts mục tiêu thay vì tự buffer theo ý mình, đáp ứng **yêu cầu 1** (độ trễ đồng nhất giữa các viewer để đảm bảo công bằng cá cược).

```mermaid
sequenceDiagram
    actor Producer as Đơn vị sản xuất
    participant Ingest as Ingest Source (primary)
    participant Transcoder
    participant Coordinator as Latency Coordinator
    participant EdgeA as Edge Node A (gần producer)
    participant EdgeB as Edge Node B (xa hơn)
    actor ViewerA as Viewer trên Edge A
    actor ViewerB as Viewer trên Edge B

    Producer->>Ingest: Đẩy luồng RTMP gốc
    Ingest->>Transcoder: Chuyển tiếp luồng gốc
    Transcoder->>Transcoder: Gắn video_pts cho từng segment
    Transcoder->>Coordinator: Đăng ký video_pts mới nhất
    Coordinator->>Coordinator: Tính target_latency_ms chung (đủ lớn để edge chậm nhất vẫn theo kịp)
    Coordinator->>EdgeA: Áp target_latency_ms, cập nhật DELIVERY_LATENCY_PROFILE
    Coordinator->>EdgeB: Áp target_latency_ms, cập nhật DELIVERY_LATENCY_PROFILE
    EdgeA->>EdgeA: Buffer thêm cho khớp target (dù mạng vốn nhanh hơn)
    EdgeB->>EdgeB: Buffer đúng target (mạng vốn đã chậm)
    EdgeA-->>ViewerA: Phát segment tại đúng video_pts mục tiêu
    EdgeB-->>ViewerB: Phát segment tại đúng video_pts mục tiêu
    Note over ViewerA,ViewerB: Cả hai viewer xem cùng một pha bóng lệch nhau không đáng kể, vì độ trễ bị ép về cùng một mốc pts thay vì để mỗi edge tự do
    Coordinator->>Coordinator: Giám sát measured_latency_ms liên tục, điều chỉnh target nếu edge nào bắt đầu tụt lại
```
