# Sequence Diagram — Enhance: Replace Listing Video

Đây là **enhance**, flow mới hoàn toàn phát sinh từ yêu cầu "seller thay video mới cho tin đang bán chạy mà không tạo khoảng trống hoặc lẫn lộn video cũ/mới cho khách đang xem cùng lúc" — chưa tồn tại ở base vì base chỉ có 1 video gắn cứng theo listing.

```mermaid
sequenceDiagram
    actor Seller
    participant Marketplace as Marketplace Service
    participant Storage as Media Storage
    participant Queue as Video Processing Queue
    actor Buyer as Đang xem tin

    Seller->>Marketplace: Upload video mới thay thế cho listing đang published
    Marketplace->>Storage: Store VIDEO_RAW mới (không xóa rendition cũ)
    Marketplace->>Queue: Enqueue VIDEO_PROCESSING_JOB cho video mới

    Note over Marketplace: LISTING.active_video_rendition_id vẫn trỏ về rendition cũ
    Buyer->>Marketplace: Xem tin trong lúc video mới đang xử lý
    Marketplace-->>Buyer: Vẫn hiển thị video cũ, không có khoảng trống

    Queue-->>Marketplace: Video mới xử lý xong, VIDEO_RENDITION mới sẵn sàng
    Marketplace->>Marketplace: Atomic swap active_video_rendition_id sang rendition mới
    Marketplace->>Storage: Đánh dấu rendition cũ để dọn dẹp sau
    Marketplace-->>Buyer: Lần tải trang tiếp theo thấy video mới, không lẫn lộn cũ/mới cùng lúc
```
