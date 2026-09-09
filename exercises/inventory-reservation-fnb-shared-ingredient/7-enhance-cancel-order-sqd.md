# Enhance sequence — Huỷ đơn với hoàn trả atomic có điều kiện thời gian

Đây là **enhance** của flow `cancel-order` đã có ở base. So với base (hoàn trả cộng ngược đơn giản, không kiểm tra bếp đã chế biến hay chưa), flow này thay đổi ở chỗ: trước khi hoàn trả, hệ thống kiểm tra `KITCHEN_TICKET.cooking_started_at` — chỉ hoàn trả atomic (dựa trên đúng các dòng `RESERVATION_LINE` đã trừ) nếu huỷ trước mốc bếp xác nhận bắt đầu chế biến, đáp ứng yêu cầu 4 trong đề bài.

```mermaid
sequenceDiagram
    actor Khach
    participant OrderSvc as Order Service
    participant Kitchen as Kitchen Ticket
    participant Stock as Ingredient Stock (DB, transactional)

    Khach->>OrderSvc: Yeu cau huy don (don da confirmed, co RESERVATION)

    OrderSvc->>Kitchen: Kiem tra KITCHEN_TICKET.cooking_started_at cua don

    alt cooking_started_at chua duoc set (bep chua bat dau che bien)
        OrderSvc->>Stock: BEGIN TRANSACTION
        OrderSvc->>Stock: Doc toan bo RESERVATION_LINE cua RESERVATION nay
        loop Voi tung RESERVATION_LINE
            OrderSvc->>Stock: UPDATE stock_quantity += quantity_deducted, version += 1
        end
        OrderSvc->>Stock: COMMIT, RESERVATION.status = released, order.status = cancelled
        Stock-->>OrderSvc: Hoan tra thanh cong toan bo nguyen lieu da tru
        OrderSvc-->>Khach: Xac nhan huy don thanh cong
    else cooking_started_at da duoc set (bep da bat dau che bien)
        OrderSvc-->>Khach: Tu choi huy, "mon da bat dau che bien, khong the hoan tra nguyen lieu"
        Note over OrderSvc,Stock: Khong thuc hien bat ky UPDATE nao len Stock,<br/>tranh hoan nham nguyen lieu da thuc su duoc dung
    end
```
