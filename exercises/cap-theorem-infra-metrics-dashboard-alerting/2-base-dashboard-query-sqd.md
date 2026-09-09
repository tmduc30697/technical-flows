# Base sequence — Dashboard query

Đây là **base**, flow "Đọc dữ liệu cho dashboard tổng hợp" — dùng cùng 1 read_quorum như mọi luồng đọc khác, không phân biệt mục đích quan sát xu hướng hay cảnh báo khẩn cấp.

```mermaid
sequenceDiagram
    actor Engineer
    participant Dashboard as Dashboard App
    participant Config as QUORUM_CONFIG store
    participant Nodes as TIME_SERIES_NODE (nhiều node)

    Engineer->>Dashboard: Xem biểu đồ xu hướng CPU/memory nhiều server
    Dashboard->>Config: Lấy read_quorum (dùng chung, không riêng cho dashboard)
    Dashboard->>Nodes: Đọc METRIC_SAMPLE từ R node theo config chung
    Nodes-->>Dashboard: Trả dữ liệu (có thể trễ vài giây, chấp nhận được cho mục đích xu hướng)
    Dashboard-->>Engineer: Hiển thị biểu đồ
```
