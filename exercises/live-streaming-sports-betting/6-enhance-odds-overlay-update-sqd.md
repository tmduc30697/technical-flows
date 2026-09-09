# Enhance sequence — Odds overlay update (đồng bộ với video)

Đây là **enhance** của flow đã có ở base "Odds overlay update". So với base, overlay không còn hiển thị ngay khi nhận dữ liệu, mà được neo vào `EVENT_TIMELINE.video_pts` và chỉ hiển thị khi player client đã decode tới đúng mốc `target_display_pts`, đáp ứng **yêu cầu 2** (tránh overlay hiển thị sự kiện trước khi hình ảnh thực tế diễn ra).

```mermaid
sequenceDiagram
    actor TraderFeed as Nguồn dữ liệu tỷ lệ cá cược
    participant OddsService as Odds Service
    participant Timeline as EVENT_TIMELINE
    participant WS as WebSocket Gateway
    actor Viewer

    Note over Viewer: Player đang decode ở video_pts hiện tại, trễ ~5s so với thời điểm thực
    TraderFeed->>OddsService: Bàn thắng vừa xảy ra (thời điểm thực tế)
    OddsService->>Timeline: Tra cứu video_pts tương ứng thời điểm sự kiện thực tế xảy ra ở nguồn ingest
    Timeline-->>OddsService: Trả về video_pts của khung hình chứa bàn thắng
    OddsService->>OddsService: Ghi ODDS_UPDATE(target_display_pts = video_pts vừa tra cứu)
    OddsService->>WS: Đẩy update kèm target_display_pts
    WS-->>Viewer: Gửi update, nhưng đánh dấu "chờ hiển thị"
    Viewer->>Viewer: Player tiếp tục decode, so sánh pts hiện tại với target_display_pts
    Note over Viewer: Khi player decode đến đúng target_display_pts
    Viewer->>Viewer: Hiển thị overlay "GOAL" đúng lúc khung hình bàn thắng xuất hiện
```
