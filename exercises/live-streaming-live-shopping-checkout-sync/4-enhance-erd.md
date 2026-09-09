# Enhance ERD — Đồng bộ mốc phát sóng, phân biệt gián đoạn, cách ly checkout, đồng bộ VOD

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ 5 yêu cầu trong đề bài. So với base, các entity/field mới:
- `BROADCAST_EVENT` (mới): mỗi sự kiện chốt giá/mở bán được đóng dấu thời gian **ngay tại nguồn phát** (`broadcast_timestamp`), kèm `price_at_event` và `displayed_stock_at_event` — là "nguồn sự thật" để hệ thống order đối chiếu, không phụ thuộc thời điểm viewer bấm mua (yêu cầu 1).
- `LIVE_SESSION.status` mở rộng thêm `interrupted`, cùng `interrupted_at`/`grace_deadline` — phân biệt gián đoạn tạm thời với kết thúc hẳn (yêu cầu 2).
- `CHECKOUT_REQUEST` (mới) là hàng đợi trung gian tách biệt khỏi pipeline ingest/CDN: mỗi lượt bấm mua tạo 1 request tham chiếu `broadcast_event_id` (giá theo nguồn phát) và được đối chiếu lại `PRODUCT.stock_quantity` thực tại thời điểm xử lý, không tin tuyệt đối `displayed_stock_at_event` (yêu cầu 1, 3, 4).
- `VOD_PRODUCT_MARKER` (mới) gắn từng `BROADCAST_EVENT` với mốc thời gian trong file VOD và `valid_until` — đảm bảo viewer xem lại không mua được sản phẩm/giá đã hết hiệu lực (yêu cầu 5).

```mermaid
erDiagram
    SELLER ||--o{ LIVE_SESSION : hosts
    LIVE_SESSION ||--o{ BROADCAST_EVENT : emits
    PRODUCT ||--o{ BROADCAST_EVENT : "referenced in"
    BROADCAST_EVENT ||--o{ CHECKOUT_REQUEST : "priced by"
    PRODUCT ||--o{ CHECKOUT_REQUEST : "ordered as"
    CHECKOUT_REQUEST ||--o| ORDER : confirms
    LIVE_SESSION ||--o| VOD : "recorded as"
    VOD ||--o{ VOD_PRODUCT_MARKER : contains
    BROADCAST_EVENT ||--o{ VOD_PRODUCT_MARKER : "anchors"

    SELLER {
        string id PK
        string name
    }
    LIVE_SESSION {
        string id PK
        string seller_id FK
        string status "live | interrupted | ended"
        datetime started_at
        datetime interrupted_at
        datetime grace_deadline
        datetime ended_at
    }
    PRODUCT {
        string id PK
        string name
        decimal price
        int stock_quantity
    }
    BROADCAST_EVENT {
        string id PK
        string live_session_id FK
        string product_id FK
        string event_type "price_lock | flash_sale_open | product_intro"
        decimal price_at_event
        int displayed_stock_at_event
        datetime broadcast_timestamp "dong dau tai nguon phat, khong phai luc viewer nhan duoc"
    }
    CHECKOUT_REQUEST {
        string id PK
        string broadcast_event_id FK
        string product_id FK
        string viewer_id
        int quantity
        string status "queued | validated | rejected | confirmed"
        datetime submitted_at
        datetime processed_at
    }
    ORDER {
        string id PK
        string checkout_request_id FK
        string product_id FK
        decimal price_charged
        int quantity
        datetime confirmed_at
    }
    VOD {
        string id PK
        string live_session_id FK
        datetime recorded_at
    }
    VOD_PRODUCT_MARKER {
        string id PK
        string vod_id FK
        string broadcast_event_id FK
        int vod_timestamp_offset_ms
        datetime valid_until
    }
```
