# Enhance sequence — Edge node selection (kết hợp địa lý và tải)

Đây là **enhance** của flow đã có ở base "Edge node selection". So với base, router không chỉ xét khoảng cách địa lý mà còn đọc `current_viewer_count`/`current_bandwidth_mbps` kèm `metric_version` mới nhất của từng ứng viên trước khi quyết định, đáp ứng **yêu cầu 1** (kết hợp cả yếu tố địa lý lẫn tải hiện tại, không chỉ chọn node gần nhất nếu node đó đã gần đạt giới hạn băng thông).

```mermaid
sequenceDiagram
    actor Viewer
    participant Router as Edge Router
    participant EdgeNear as Edge Node gần nhất (tải cao)
    participant EdgeFar as Edge Node xa hơn (tải thấp)

    Viewer->>Router: Yêu cầu xem video (kèm vị trí địa lý)
    Router->>EdgeNear: Đọc current_viewer_count, current_bandwidth_mbps, metric_version
    Router->>EdgeFar: Đọc current_viewer_count, current_bandwidth_mbps, metric_version
    Router->>Router: Tính điểm kết hợp = trọng số(độ trễ ước tính) + trọng số(mức tải hiện tại)
    alt EdgeNear đã gần giới hạn băng thông
        Router->>Router: Chọn EdgeFar dù xa hơn, vì điểm kết hợp tốt hơn
        Router->>Router: Ghi ROUTING_DECISION(candidate=EdgeFar, load_snapshot_version=version vừa đọc)
        Router-->>Viewer: Route tới EdgeFar
        EdgeFar-->>Viewer: Phát mượt, không giật ngay từ đầu
    else EdgeNear còn đủ tải
        Router->>Router: Chọn EdgeNear vì vừa gần vừa còn tải
        Router-->>Viewer: Route tới EdgeNear
    end
```
