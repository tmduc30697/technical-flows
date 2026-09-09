# Enhance sequence — Route latency theo vùng & tương quan phản hồi tài xế

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: đo latency theo từng khu vực địa lý, tỉ lệ fallback, và tương quan thời gian breaker mở với phản hồi tiêu cực từ tài xế.

```mermaid
sequenceDiagram
    participant RouteReq as ROUTE_REQUEST store
    participant Metric as ROUTE_LATENCY_METRIC store
    participant Breaker as CIRCUIT_BREAKER_STATE
    actor Driver
    participant Feedback as DRIVER_FEEDBACK store
    participant Correlation as BREAKER_FEEDBACK_CORRELATION store
    actor Analyst as Data/Ops Analyst

    loop Mỗi request route hoàn tất
        RouteReq->>Metric: Ghi response_time_ms theo region của chuyến đi
    end
    Metric->>Metric: Tổng hợp p95_latency_ms theo từng region theo window

    Driver->>Feedback: Báo cáo bị lạc đường/chỉ dẫn sai (nếu có)
    Feedback->>Feedback: Ghi kèm trip_id, thời điểm report

    Analyst->>Breaker: Lấy toàn bộ khoảng thời gian breaker đã mở (opened_at → closed_at)
    Analyst->>Feedback: Lấy toàn bộ DRIVER_FEEDBACK trong và quanh các khoảng đó
    Analyst->>Correlation: Ghi BREAKER_FEEDBACK_CORRELATION (breaker_open_window, feedback_count)
    Correlation-->>Analyst: Cho biết vùng/thời điểm nào breaker mở tương quan rõ với tăng phản hồi tiêu cực

    Note over Metric,Correlation: Kết quả dùng để quyết định có cần đổi provider chính cho riêng 1 số vùng có latency/chất lượng kém hơn hay không
```
