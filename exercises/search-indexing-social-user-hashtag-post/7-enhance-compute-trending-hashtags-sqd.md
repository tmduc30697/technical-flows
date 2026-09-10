# Sequence Diagram — Enhance: Compute Trending Hashtags

Đây là **enhance**, flow hoàn toàn mới: hashtag trending được tính gần thời gian thực dựa trên tốc độ tăng trưởng sử dụng trong cửa sổ thời gian ngắn, thay vì tổng lượt dùng lịch sử, để tránh hashtag cũ dùng nhiều nhưng đã hết hot vẫn đứng đầu.

```mermaid
sequenceDiagram
    participant Queue as Index Sync Event Queue
    participant TrendWorker as Trending Compute Worker
    participant DB as Hashtag Trend Metric Store
    actor Searcher as Người tìm kiếm
    participant WebApp as Social App

    loop Liên tục nhận sự kiện dùng hashtag
        Queue->>TrendWorker: Deliver post event kèm hashtag_ids
        TrendWorker->>DB: Tăng usage_in_window cho cửa sổ thời gian hiện tại
    end

    loop Định kỳ ngắn, ví dụ mỗi phút
        TrendWorker->>DB: Đọc usage_in_window hiện tại và cửa sổ trước đó cho từng hashtag
        TrendWorker->>TrendWorker: Tính growth_rate = tăng trưởng giữa 2 cửa sổ liên tiếp
        TrendWorker->>DB: Cập nhật growth_rate, xếp hạng lại danh sách trending
    end

    Searcher->>WebApp: Xem danh sách hashtag trending
    WebApp->>DB: Lấy top hashtag theo growth_rate
    DB-->>WebApp: Danh sách trending phản ánh đúng xu hướng đang lên, không phải hashtag cũ dùng nhiều nhưng đã hết hot
    WebApp-->>Searcher: Hiển thị trending gần thời gian thực
```
