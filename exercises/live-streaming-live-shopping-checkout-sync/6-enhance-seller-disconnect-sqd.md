# Enhance sequence — Phân biệt gián đoạn tạm thời và kết thúc phiên live

Đây là **enhance** của flow `seller-disconnect` đã có ở base. So với base (kết thúc phiên và huỷ mọi đơn đang xử lý ngay khi mất kết nối), flow này thay đổi ở chỗ: hệ thống chuyển phiên sang trạng thái `interrupted` kèm `grace_deadline`, các `CHECKOUT_REQUEST` đang xử lý dở được giữ nguyên chờ, chỉ thực sự đóng phiên và huỷ đơn còn dang dở nếu quá hạn grace mà người bán không kết nối lại — đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    actor Seller
    participant Ingest as Ingest Server
    participant Session as Live Session Store
    participant Checkout as Checkout Service
    actor Viewer

    Viewer->>Checkout: Dang co CHECKOUT_REQUEST status=queued cho san pham X
    Seller--x Ingest: Mat ket noi dot ngot (rot mang)
    Ingest->>Session: UPDATE status = interrupted, interrupted_at = now,<br/>grace_deadline = now + N giay
    Session-->>Viewer: Hien thi "phien tam gian doan", KHONG huy CHECKOUT_REQUEST dang cho

    alt Seller ket noi lai truoc grace_deadline
        Seller->>Ingest: Ket noi lai
        Ingest->>Session: UPDATE status = live, xoa interrupted_at/grace_deadline
        Session->>Checkout: Bao phien da khoi phuc
        Checkout->>Checkout: Tiep tuc xu ly binh thuong cac CHECKOUT_REQUEST dang cho
        Checkout-->>Viewer: Don van duoc xu ly tiep, khong bi huy oan
    else Qua grace_deadline ma khong ket noi lai
        Ingest->>Session: UPDATE status = ended, ended_at = now
        Session->>Checkout: Bao phien da thuc su ket thuc
        Checkout->>Checkout: Huy cac CHECKOUT_REQUEST con dang cho (status = rejected)
        Checkout-->>Viewer: Bao don bi huy vi phien live da ket thuc that su
    end
```
