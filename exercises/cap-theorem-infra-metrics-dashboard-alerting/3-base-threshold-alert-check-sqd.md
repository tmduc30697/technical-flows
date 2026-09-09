# Base sequence — Threshold alert check (dùng chung quorum với dashboard)

Đây là **base**, flow "Kiểm tra ngưỡng nguy hiểm để cảnh báo" ở trạng thái hiện tại — dùng cùng read_quorum với dashboard, có thể đọc phải replica trễ. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    participant AlertEngine as Alert Engine
    participant Config as QUORUM_CONFIG store
    participant Nodes as TIME_SERIES_NODE
    actor OnCall as On-call Engineer

    loop Định kỳ kiểm tra ngưỡng
        AlertEngine->>Config: Lấy read_quorum (giống hệt dashboard)
        AlertEngine->>Nodes: Đọc METRIC_SAMPLE mới nhất từ R node theo config chung
        Nodes-->>AlertEngine: Trả giá trị (có thể là replica trễ vài giây)
        AlertEngine->>AlertEngine: So sánh với ngưỡng nguy hiểm
    end
    Note over AlertEngine,Nodes: Giá trị thực đã vượt ngưỡng từ trước đó ở node vừa nhận ghi, nhưng replica bị đọc lại chưa cập nhật kịp — alert không được kích hoạt đúng lúc
    Note over OnCall: Server sập vì hết dung lượng đĩa mà không ai được cảnh báo trước
```
