# Enhance sequence — Dashboard query (R thấp riêng, đo staleness)

Đây là **enhance**, cùng flow "Dashboard query" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 5 của đề bài: dùng `QUORUM_POLICY(purpose=dashboard)` với R thấp riêng, và đo `DASHBOARD_STALENESS_METRIC` để biết tỉ lệ đọc bị trễ.

```mermaid
sequenceDiagram
    actor Engineer
    participant Dashboard as Dashboard App
    participant Policy as QUORUM_POLICY store
    participant Nodes as TIME_SERIES_NODE
    participant Metric as DASHBOARD_STALENESS_METRIC store

    Engineer->>Dashboard: Xem biểu đồ xu hướng nhiều server
    Dashboard->>Policy: Lấy QUORUM_POLICY(purpose=dashboard) — read_quorum thấp
    Dashboard->>Nodes: Đọc METRIC_SAMPLE từ R thấp
    Nodes-->>Dashboard: Trả dữ liệu, kèm data_as_of (thời điểm dữ liệu thực sự đại diện)
    Dashboard->>Metric: Ghi DASHBOARD_STALENESS_METRIC (requested_at, data_as_of, staleness_ms)
    Dashboard-->>Engineer: Hiển thị biểu đồ, chấp nhận trễ vài giây có chủ đích
    Note over Metric: Số liệu này dùng để đánh giá tỉ lệ đọc dashboard bị stale vượt ngưỡng chấp nhận được
```
