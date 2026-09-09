# Enhance sequence — Threshold alert check (R cao/đọc node vừa ghi, đo alert latency)

Đây là **enhance**, cùng flow "Threshold alert check" đã có ở base nhưng nay thay đổi theo yêu cầu 2 và 5 của đề bài: dùng `QUORUM_POLICY(purpose=alert)` với R cao hoặc đọc trực tiếp node vừa nhận ghi, và đo `ALERT_LATENCY_METRIC` end-to-end.

```mermaid
sequenceDiagram
    participant AlertEngine as Alert Engine
    participant Policy as QUORUM_POLICY store
    participant Nodes as TIME_SERIES_NODE
    participant Metric as ALERT_LATENCY_METRIC store
    actor OnCall as On-call Engineer

    loop Định kỳ kiểm tra ngưỡng
        AlertEngine->>Policy: Lấy QUORUM_POLICY(purpose=alert) — read_quorum cao hoặc read_from_latest_writer_node=true
        AlertEngine->>Nodes: Đọc METRIC_SAMPLE mới nhất theo policy alert
        Nodes-->>AlertEngine: Trả giá trị chính xác, không bị trễ như base
        AlertEngine->>AlertEngine: So sánh với ngưỡng nguy hiểm
        alt Vượt ngưỡng
            AlertEngine->>Metric: Ghi ALERT_LATENCY_METRIC (breach_at=sample_timestamp, alert_triggered_at=now)
            AlertEngine->>OnCall: Kích hoạt cảnh báo ngay
        else Chưa vượt ngưỡng
            AlertEngine->>AlertEngine: Tiếp tục theo dõi
        end
    end
    Note over Metric: end-to-end alert latency được đo và tách riêng khỏi staleness của dashboard để đánh giá đúng trade-off từng luồng
```
