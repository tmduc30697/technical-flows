# Enhance sequence — Business metric evaluation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base chỉ có dashboard kỹ thuật). Đáp ứng yêu cầu 1 và 2 của đề bài: so sánh CTR/conversion rate giữa canary và stable, và coi chất lượng gợi ý kém hơn về nghiệp vụ là tiêu chí rollback dù kỹ thuật hoàn toàn ổn định.

```mermaid
sequenceDiagram
    participant Canary as Model canary
    participant Stable as Model stable
    participant Metric as BUSINESS_METRIC_WINDOW store
    participant Policy as CANARY_EVALUATION_POLICY store
    participant Evaluator as Canary Evaluator
    participant MLOps as MLOps Engineer

    Canary->>Metric: Ghi click_through_rate + conversion_rate theo window
    Stable->>Metric: Ghi click_through_rate + conversion_rate theo window (baseline)

    Note over Metric: Đã đủ min_duration_hours theo CANARY_EVALUATION_POLICY

    Evaluator->>Metric: Lấy toàn bộ business metric của cả 2 model trong suốt thời gian canary
    Evaluator->>Policy: Lấy business_metric_threshold để so sánh

    alt Canary kỹ thuật ổn định NHƯNG business metric kém hơn stable
        Evaluator->>Evaluator: Vẫn coi là fail — chất lượng gợi ý nghiệp vụ mới là tiêu chí quyết định
        Evaluator->>MLOps: Đề xuất rollback về Stable, kèm số liệu CTR/conversion so sánh
    else Canary business metric bằng hoặc tốt hơn stable
        Evaluator->>MLOps: Đề xuất promote Canary thành Stable mới
    end
```
