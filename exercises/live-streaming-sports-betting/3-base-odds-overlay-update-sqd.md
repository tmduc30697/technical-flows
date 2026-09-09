# Base sequence — Odds overlay update

Đây là **base**, flow "Cập nhật overlay tỷ lệ cá cược" ở trạng thái hiện tại: dữ liệu tỷ số/tỷ lệ cá cược được đẩy tới client độc lập hoàn toàn với pipeline video, không tham chiếu tới bất kỳ mốc thời gian nào của luồng hình ảnh. Flow này là tiền đề cho yêu cầu 2 (đồng bộ video và overlay) vì đây chính là chỗ gây ra lệch pha giữa hình ảnh và overlay.

```mermaid
sequenceDiagram
    actor TraderFeed as Nguồn dữ liệu tỷ lệ cá cược
    participant OddsService as Odds Service
    participant WS as WebSocket Gateway
    actor Viewer

    Note over Viewer: Đang xem video, hình ảnh bị trễ ~5s so với thời điểm thực do encode + buffer
    TraderFeed->>OddsService: Bàn thắng vừa xảy ra (thời điểm thực tế)
    OddsService->>OddsService: Ghi ODDS_UPDATE(event_type=goal)
    OddsService->>WS: Đẩy update ngay lập tức
    WS-->>Viewer: Hiển thị overlay "GOAL" ngay khi nhận được
    Note over Viewer: Overlay báo bàn thắng hiện lên trước khi hình ảnh bàn thắng thực sự xuất hiện trên màn hình, vì overlay không chờ luồng video bắt kịp
```
