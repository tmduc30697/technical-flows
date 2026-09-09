# Base sequence — Xử lý người bán mất kết nối (naive)

Đây là **base**, mô tả cách hệ thống hiện tại xử lý khi người bán rớt kết nối: hệ thống kết thúc phiên live ngay lập tức khi phát hiện mất luồng, không phân biệt mất mạng tạm thời hay live đã thực sự kết thúc, các đơn đang xử lý dở bị huỷ theo. Đây là tiền đề cho enhance yêu cầu 2 — phân biệt "gián đoạn tạm thời" và "kết thúc phiên".

```mermaid
sequenceDiagram
    actor Seller
    participant Ingest as Ingest Server
    participant Session as Live Session Store
    participant OrderSvc as Order Service
    actor Viewer

    Viewer->>OrderSvc: Dang chuan bi chot don san pham X
    Seller--x Ingest: Mat ket noi dot ngot (rot mang)
    Ingest->>Session: UPDATE LIVE_SESSION SET status = ended, ended_at = now
    Session->>OrderSvc: Bao phien live da ket thuc
    OrderSvc->>OrderSvc: Huy toan bo don dang xu ly do thuoc live_session nay
    OrderSvc-->>Viewer: Don bi huy vi "phien live da ket thuc"

    Note over Seller,OrderSvc: Neu day chi la rot mang tam thoi va Seller ket noi lai sau vai giay,<br/>don cua Viewer van da bi huy oan truoc do
```
