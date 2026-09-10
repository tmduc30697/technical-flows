# Sequence Diagram — Base: Search Basic

Đây là **base**, flow tìm kiếm cơ bản trên index đồng bộ theo lô — chưa có cá nhân hóa theo follow, chưa có trending theo tốc độ tăng trưởng, chỉ xếp hạng theo độ phổ biến tổng/khớp từ khóa đơn thuần.

```mermaid
sequenceDiagram
    actor Searcher as Người tìm kiếm
    participant WebApp as Social App
    participant Index as Search Index

    Searcher->>WebApp: Tìm "tên người dùng" hoặc "#hashtag"
    WebApp->>Index: Query khớp từ khóa, xếp theo total_usage_count/độ phổ biến chung
    Index-->>WebApp: Kết quả không phân biệt người tìm là ai, không ưu tiên bạn bè/follow

    WebApp-->>Searcher: Hiển thị kết quả, có thể đã lỗi thời do đồng bộ theo lô
```
