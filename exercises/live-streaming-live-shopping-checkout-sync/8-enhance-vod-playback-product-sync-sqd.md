# Enhance sequence — Đồng bộ mốc sản phẩm khi xem lại VOD

Đây là **enhance**, flow hoàn toàn mới so với base (base không lưu mốc thời gian giới thiệu sản phẩm đồng bộ với VOD). Đáp ứng yêu cầu 5 trong đề bài: khi buổi live được ghi lại làm VOD, các mốc giới thiệu sản phẩm phải đồng bộ chính xác với luồng gốc, và viewer xem VOD không thể mua vào sản phẩm/giá đã hết hiệu lực từ buổi live gốc.

```mermaid
sequenceDiagram
    participant Recorder as VOD Recorder
    participant VODStore as VOD Store
    participant Marker as VOD Product Marker Store
    actor ViewerVOD as Viewer xem lai VOD
    participant Checkout as Checkout Service

    Note over Recorder: Trong luc live, moi BROADCAST_EVENT da phat sinh
    Recorder->>VODStore: Ghi hinh lien tuc thanh 1 file VOD duy nhat
    Recorder->>Marker: Voi moi BROADCAST_EVENT, tao VOD_PRODUCT_MARKER<br/>(vod_timestamp_offset_ms khop voi thoi diem event xay ra trong file goc, valid_until = het hieu luc live)

    ViewerVOD->>VODStore: Xem lai VOD, tua toi doan gioi thieu san pham X
    VODStore-->>ViewerVOD: Hien thi lai canh gioi thieu, kem nut mua hang gan voi marker tuong ung

    ViewerVOD->>Checkout: Bam mua san pham X tu man hinh VOD
    Checkout->>Marker: Doc VOD_PRODUCT_MARKER tuong ung, kiem tra valid_until
    alt Con hieu luc (hiem, vd flash sale keo dai)
        Checkout->>Checkout: Xu ly binh thuong nhu 1 CHECKOUT_REQUEST moi, doi chieu ton kho thuc
        Checkout-->>ViewerVOD: Xac nhan don
    else Da het hieu luc tu buoi live goc
        Checkout-->>ViewerVOD: Tu choi, "gia/san pham nay chi ap dung trong buoi live da ket thuc"
    end
```
