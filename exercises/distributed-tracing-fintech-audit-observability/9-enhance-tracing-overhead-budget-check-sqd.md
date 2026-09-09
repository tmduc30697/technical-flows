# Sequence Diagram - Enhance: tracing-overhead-budget-check

Đây là flow **enhance mới**: tracing SDK tự đo thời gian nó tiêu tốn thêm (`tracing_overhead_ms`) so với thời gian xử lý nghiệp vụ thật (`business_latency_ms`), tính tỉ lệ overhead so với latency gốc, và so sánh với ngưỡng cho phép (ví dụ 5%) - nếu vượt ngưỡng, hệ thống tự động giảm mức chi tiết tracing (ví dụ bớt số attribute ghi thêm) và cảnh báo đội nền tảng thay vì để tracing âm thầm làm chậm giao dịch thật. Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant Gateway as transfer-gateway
    participant Tracer as Tracing SDK
    participant Store as Trace Storage
    participant Alert as Alerting/Ops

    Gateway->>Tracer: Bắt đầu đo tracing_overhead_ms song song với business_latency_ms
    Gateway->>Gateway: Xử lý nghiệp vụ giao dịch (fraud-check, ledger-write, notification)
    Gateway->>Tracer: Kết thúc đo, business_latency_ms=200, tracing_overhead_ms=8
    Tracer->>Tracer: Tính overhead_ratio_pct = 8/200 = 4%
    Tracer->>Store: Ghi trace kèm overhead_ratio_pct=4%
    alt overhead_ratio_pct vượt ngưỡng cho phép (ví dụ 5%)
        Tracer->>Tracer: Tự động giảm mức chi tiết tracing (bớt attribute không thiết yếu)
        Tracer->>Alert: Cảnh báo đội nền tảng, tracing đang ảnh hưởng đáng kể tới latency giao dịch
    else overhead_ratio_pct trong ngưỡng cho phép
        Tracer->>Tracer: Giữ nguyên mức chi tiết tracing hiện tại
    end
```
