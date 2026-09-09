# Enhance sequence — Đặt hàng đối chiếu theo mốc phát sóng và tồn kho thực

Đây là **enhance** của flow `place-order-during-live` đã có ở base. So với base (order tin theo giá/tồn hiển thị trên máy viewer tại thời điểm bấm mua), flow này thay đổi ở chỗ: mọi `CHECKOUT_REQUEST` được gắn với `BROADCAST_EVENT` (giá đã đóng dấu thời gian tại nguồn phát) chứ không phải thời điểm viewer bấm mua (yêu cầu 1), và trước khi xác nhận đơn, hệ thống luôn đọc lại `PRODUCT.stock_quantity` thực chứ không tin `displayed_stock_at_event` trên overlay (yêu cầu 3).

```mermaid
sequenceDiagram
    actor SellerLive as Nguoi ban (dang live)
    participant Source as Broadcast Event Recorder (tai nguon phat)
    participant CDN
    actor ViewerSlow as Viewer do tre cao (edge xa)
    participant Checkout as Checkout Service
    participant Stock as Product Stock (DB)

    SellerLive->>Source: Chot gia moi/mo ban san pham X
    Source->>Source: Tao BROADCAST_EVENT, broadcast_timestamp = now (tai nguon)
    Source->>CDN: Nhung event ID + gia vao luong video de phan phoi

    CDN-->>ViewerSlow: Nhan luong tre vai giay, kem BROADCAST_EVENT ID dang hien hanh luc do

    ViewerSlow->>Checkout: Bam mua, gui kem broadcast_event_id doc duoc tu overlay
    Checkout->>Checkout: Tao CHECKOUT_REQUEST, gia lay tu BROADCAST_EVENT (khong lay gia hien tai cua he thong)
    Checkout->>Stock: Doc stock_quantity THUC tai thoi diem xu ly (khong dung displayed_stock_at_event)
    Stock-->>Checkout: stock_quantity thuc te

    alt Con hang thuc te
        Checkout->>Stock: Tru tam/giu cho
        Checkout-->>ViewerSlow: Xac nhan don theo dung gia cua BROADCAST_EVENT ma viewer da thay
    else Het hang thuc te (du overlay van con hien so duong)
        Checkout-->>ViewerSlow: Tu choi, "san pham vua het hang", khong bam vao con so overlay da tre
    end
```
