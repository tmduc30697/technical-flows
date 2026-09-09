# Enhance sequence — Load metric race condition handling

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có khái niệm cập nhật tải với kiểm soát đồng thời. Đáp ứng **yêu cầu 4** (nhiều nguồn cập nhật số liệu tải gần như đồng thời, quyết định route phải dựa trên số liệu đủ mới, tránh dồn nhiều viewer mới vào cùng một node vì đọc phải số liệu đã lỗi thời).

```mermaid
sequenceDiagram
    actor ViewerJoin as Viewer mới join
    actor ViewerLeave as Viewer khác rời đi
    participant Edge as Edge Node
    participant Metric as LOAD_METRIC_UPDATE store (atomic, có version)
    participant Router as Edge Router
    actor ViewerA as Viewer A (đang chọn node)
    actor ViewerB as Viewer B (đang chọn node gần như cùng lúc)

    ViewerJoin->>Metric: Ghi sự kiện join, tăng metric_version (atomic increment)
    ViewerLeave->>Metric: Ghi sự kiện leave gần như đồng thời, tăng metric_version (atomic increment)
    Metric->>Metric: Đảm bảo hai lần tăng version không ghi đè lẫn nhau (compare-and-swap / atomic counter)

    par Hai viewer chọn node gần như cùng lúc
        ViewerA->>Router: Yêu cầu route
        Router->>Metric: Đọc current_viewer_count kèm metric_version mới nhất tại thời điểm đọc
        Metric-->>Router: Trả về số liệu tại version N
        Router->>Router: Ghi ROUTING_DECISION(load_snapshot_version=N)
        Router-->>ViewerA: Route dựa trên số liệu version N
    and
        ViewerB->>Router: Yêu cầu route
        Router->>Metric: Đọc current_viewer_count kèm metric_version mới nhất tại thời điểm đọc
        Metric-->>Router: Trả về số liệu tại version N+1 (đã cộng thêm ViewerA vừa route)
        Router->>Router: Ghi ROUTING_DECISION(load_snapshot_version=N+1)
        Router-->>ViewerB: Route dựa trên số liệu version N+1, không dồn cùng node với ViewerA nếu node đã gần đầy
    end
    Note over Metric,Router: Vì mỗi lần đọc đều gắn kèm version, router luôn biết dữ liệu mình dùng mới tới đâu, tránh quyết định dựa trên số liệu đã lỗi thời một nhịp
```
