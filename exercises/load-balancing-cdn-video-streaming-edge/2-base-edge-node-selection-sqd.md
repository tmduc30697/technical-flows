# Base sequence — Edge node selection

Đây là **base**, flow "Chọn edge node cho viewer mới" ở trạng thái hiện tại: chỉ dựa vào khoảng cách địa lý đơn thuần, không xét tới tải hiện tại của node. Flow này là tiền đề cho **yêu cầu 1** (kết hợp cả địa lý lẫn tải hiện tại) vì đây chính là logic sẽ được thay đổi.

```mermaid
sequenceDiagram
    actor Viewer
    participant Router as Edge Router
    participant EdgeNear as Edge Node gần nhất (đã gần đầy tải)
    participant EdgeFar as Edge Node xa hơn (còn nhiều tải trống)

    Viewer->>Router: Yêu cầu xem video (kèm vị trí địa lý)
    Router->>Router: Tính khoảng cách địa lý tới từng edge node
    Router->>Router: Chọn EdgeNear vì gần nhất theo khoảng cách, không kiểm tra tải hiện tại
    Router-->>Viewer: Route tới EdgeNear
    EdgeNear-->>Viewer: Bắt đầu phát, nhưng node đã gần đạt giới hạn băng thông
    Note over EdgeNear,Viewer: Viewer có thể gặp giật/lag ngay từ đầu vì bị dồn vào node gần nhất dù node đó đã tải cao, trong khi EdgeFar còn dư tải
```
