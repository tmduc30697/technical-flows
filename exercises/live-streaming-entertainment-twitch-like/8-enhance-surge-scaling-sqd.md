# Enhance sequence — Phân phối chịu tải viewer tăng đột biến (raid)

Đây là **enhance**, flow hoàn toàn mới so với base (base không có khái niệm edge node/mở rộng động, chỉ có 1 CDN chung). Đáp ứng yêu cầu 5 trong đề bài: khi lượng viewer đồng thời tăng đột biến (raid, streamer nổi tiếng lên live), kiến trúc phân phối phải mở rộng mà không làm tăng độ trễ cho các viewer khác đang xem stream khác.

```mermaid
sequenceDiagram
    actor Viewer as Lan song viewer moi (raid)
    participant LB as Edge Load Balancer
    participant Edge1 as Edge Node A (dang phuc vu stream khac)
    participant Edge2 as Edge Node B (moi duoc cap phat)
    participant Origin as Origin/Transcode

    Viewer->>LB: Hang loat request xem stream cung luc
    LB->>Edge1: Kiem tra current_viewer_count cua Edge1
    Edge1-->>LB: current_viewer_count vuot nguong, status = overloaded

    LB->>Edge2: Cap phat edge node moi cho luong viewer them vao
    Edge2->>Origin: Pull rendition can thiet (cache-miss lan dau)
    Origin-->>Edge2: Tra ve segment/rendition
    Edge2-->>LB: San sang phuc vu

    par Phan tai sang edge moi
        LB->>Edge2: Dieu huong toan bo viewer moi sang Edge2
    end

    Note over Edge1: Vi Edge1 khong nhan them viewer moi,<br/>viewer dang xem stream khac tren Edge1 khong bi tang do tre
    Note over Edge2,Origin: Edge2 cache lai rendition sau lan pull dau,<br/>cac request tiep theo tu edge nay khong can goi lai Origin
```
