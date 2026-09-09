# Base sequence — Đặt hàng dựa trên giá/tồn hiển thị tại máy viewer

Đây là **base**, mô tả flow đặt hàng ở trạng thái hiện tại: viewer bấm mua dựa trên giá/tồn kho đang hiển thị trên màn hình của họ, hệ thống order xác nhận theo đúng giá trị đó mà không đối chiếu lại thời điểm sự kiện chốt giá/mở bán thực tế đã diễn ra tại nguồn phát khi nào. Đây là tiền đề cho enhance yêu cầu 1 và 3 — vấn đề nảy sinh khi viewer xem chậm hơn do độ trễ CDN khác nhau.

```mermaid
sequenceDiagram
    actor SellerLive as Nguoi ban (dang live)
    participant CDN
    actor ViewerFast as Viewer do tre thap
    actor ViewerSlow as Viewer do tre cao (edge xa)
    participant OrderSvc as Order Service
    participant Stock as Product Stock (DB)

    SellerLive->>CDN: Cong bo gia moi/mo ban san pham X (tren man hinh cua nguoi ban, ngay lap tuc)
    CDN-->>ViewerFast: Nhan luong gan nhu ngay, thay gia moi
    CDN-->>ViewerSlow: Nhan luong tre vai giay, van con thay gia/ton CU

    ViewerSlow->>OrderSvc: Bam mua theo gia/ton dang thay tren man hinh (da cu)
    OrderSvc->>Stock: Doc gia/ton hien tai luc xu ly
    Stock-->>OrderSvc: Gia/ton co the da doi so voi luc ViewerSlow bam mua

    OrderSvc-->>ViewerSlow: Xac nhan don theo gia hien tai cua he thong<br/>(khong khop voi gia ViewerSlow nhin thay luc bam mua)
```
