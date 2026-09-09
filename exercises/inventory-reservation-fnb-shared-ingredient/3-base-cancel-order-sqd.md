# Base sequence — Huỷ đơn và hoàn trả nguyên liệu

Đây là **base**, mô tả flow huỷ đơn ở trạng thái hiện tại: khách đổi ý, hệ thống hoàn trả lại nguyên liệu về kho theo kiểu cộng ngược đơn giản, không kiểm tra mốc thời gian bếp đã bắt đầu chế biến hay chưa. Đây là tiền đề cho enhance yêu cầu 4 trong đề bài — hoàn trả phải atomic và chỉ hợp lệ trước mốc bếp xác nhận bắt đầu chế biến.

```mermaid
sequenceDiagram
    actor Khach
    participant OrderSvc as Order Service
    participant Stock as Ingredient Stock (DB)
    participant Kitchen as Bep

    Khach->>OrderSvc: Yeu cau huy don (don da confirmed)
    OrderSvc->>OrderSvc: Cap nhat order.status = cancelled
    loop Voi tung dong don da tru truoc do
        OrderSvc->>Stock: UPDATE stock_quantity = stock_quantity + so_luong
        Stock-->>OrderSvc: OK, hoan tra thanh cong
    end
    OrderSvc-->>Khach: Xac nhan huy don thanh cong

    Note over OrderSvc,Kitchen: Khong co buoc kiem tra bep da bat dau che bien hay chua,<br/>neu bep da dung nguyen lieu de che bien thi phan hoan tra nay se sai lech ton kho thuc te
```
