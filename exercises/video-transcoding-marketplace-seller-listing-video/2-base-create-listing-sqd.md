# Sequence Diagram — Base: Create Listing

Đây là **base**, flow seller đăng tin bán — tiền đề bắt buộc cho enhance: chính flow này tạo ra `LISTING` và `VIDEO_RAW` (file gốc chưa qua xử lý), là điểm xuất phát để pipeline chuẩn hóa video sau này gắn vào.

```mermaid
sequenceDiagram
    actor Seller
    participant Marketplace as Marketplace Service
    participant Storage as Media Storage

    Seller->>Marketplace: Submit listing (title, price, description, images)
    Marketplace->>Storage: Store LISTING_IMAGE(s)
    Seller->>Marketplace: Upload video (optional, file gốc từ thiết bị)
    Marketplace->>Storage: Store VIDEO_RAW nguyên trạng, không xử lý gì thêm

    Marketplace->>Marketplace: Create LISTING, status = published
    Marketplace-->>Seller: Listing published, video hiển thị nguyên file gốc
    Note over Marketplace,Storage: Codec/độ phân giải/tỉ lệ khung hình không đồng nhất giữa các listing
```
