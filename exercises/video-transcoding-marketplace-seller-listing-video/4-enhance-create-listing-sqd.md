# Sequence Diagram — Enhance: Create Listing

Đây là **enhance**, flow đăng tin bán đã thay đổi so với base ([2-base-create-listing-sqd.md](2-base-create-listing-sqd.md)): listing publish ngay với ảnh/thông tin mà không chờ video xử lý xong; video được đẩy vào hàng đợi xử lý nền và tự động cập nhật hiển thị khi hoàn tất, thay vì hiển thị file gốc chưa chuẩn hóa như ở base.

```mermaid
sequenceDiagram
    actor Seller
    participant Marketplace as Marketplace Service
    participant Storage as Media Storage
    participant Queue as Video Processing Queue

    Seller->>Marketplace: Submit listing (title, price, description, images, video)
    Marketplace->>Storage: Store LISTING_IMAGE(s)
    Marketplace->>Storage: Store VIDEO_RAW
    Marketplace->>Marketplace: Create LISTING, status = published (chưa có active_video)
    Marketplace-->>Seller: Listing published ngay, hiển thị ảnh + thông tin, video đang xử lý

    Marketplace->>Queue: Enqueue VIDEO_PROCESSING_JOB (không block request)
    Note over Marketplace,Queue: Seller có thể rời trang, không cần chờ

    Queue-->>Marketplace: Job completed, VIDEO_RENDITION sẵn sàng (xem 5-enhance-validate-and-transcode-video-sqd.md)
    Marketplace->>Marketplace: Set LISTING.active_video_rendition_id
    Marketplace-->>Seller: Video hiển thị tự động trên trang sản phẩm, không cần thao tác lại
```
