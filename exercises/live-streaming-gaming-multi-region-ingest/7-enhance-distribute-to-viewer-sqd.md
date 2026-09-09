# Enhance sequence — Phân phối qua nhân bản nội dung liên vùng

Đây là **enhance** của flow `distribute-to-viewer` đã có ở base. So với base (mọi viewer kéo trực tiếp từ điểm ingest gốc), flow này thay đổi ở chỗ: nội dung sau khi ingest ở 1 vùng được nhân bản sang các vùng khác có viewer, viewer ở xa xem từ bản sao gần mình thay vì kéo xuyên lục địa từ điểm gốc — đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant EndpointA as Ingest Endpoint (vung goc)
    participant Replicator as Content Replicator
    participant RegionB as Ban sao vung xa (Region B)
    actor ViewerNear as Viewer gan vung goc
    actor ViewerFar as Viewer o vung xa

    EndpointA->>Replicator: Publish noi dung moi ingest tu STREAM_SESSION
    Replicator->>RegionB: Nhan ban noi dung sang Region B
    Replicator->>Replicator: Ghi nhan CONTENT_REPLICA voi replication_latency_ms

    ViewerNear->>EndpointA: Yeu cau xem stream
    EndpointA-->>ViewerNear: Tra ve truc tiep tu vung goc, do tre thap

    ViewerFar->>RegionB: Yeu cau xem stream (dinh tuyen toi ban sao gan nhat)
    RegionB-->>ViewerFar: Tra ve tu ban sao dia phuong, do tre hop ly thay vi keo xuyen luc dia
```
