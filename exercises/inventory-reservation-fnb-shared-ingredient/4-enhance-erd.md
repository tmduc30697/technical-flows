# Enhance ERD — Atomic reservation nguyên liệu dùng chung nhiều món

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ 5 yêu cầu trong đề bài lên base. So với base, các entity/field mới:
- `INGREDIENT` thêm `version` (phục vụ update nguyên tử có điều kiện đủ số lượng — yêu cầu 1) và `low_stock_threshold` + `is_low_stock` (yêu cầu 3, đánh dấu sắp hết theo thời gian gần thực).
- `RESERVATION` (mới) đại diện cho 1 giao dịch atomic gộp toàn bộ nguyên liệu của toàn bộ món trong 1 đơn khi xác nhận (yêu cầu 2), có `status` (reserved/released/consumed) và mốc `reserved_at`.
- `RESERVATION_LINE` (mới) là từng dòng trừ tồn cụ thể thuộc 1 `RESERVATION`, gắn `ingredient_id` + `quantity_deducted`, dùng để hoàn trả chính xác khi huỷ (yêu cầu 4).
- `KITCHEN_TICKET` (mới) ghi `cooking_started_at` — mốc thời gian rõ ràng quyết định đơn còn được huỷ-hoàn trả hay không (yêu cầu 4).
- `STOCK_SYNC_EVENT` (mới) ghi lại mỗi lần trạng thái tồn/ngưỡng của 1 nguyên liệu được broadcast tới các kênh, kèm `propagated_at` từng kênh, phục vụ theo dõi độ trễ đồng bộ và chặn nhận đơn mới khi phát hiện hết hàng (yêu cầu 5).

```mermaid
erDiagram
    CHANNEL ||--o{ ORDER_ITEM : "orders"
    CHANNEL ||--o{ STOCK_SYNC_EVENT : "receives"
    ORDER ||--o{ ORDER_ITEM : contains
    ORDER ||--o| RESERVATION : "confirmed via"
    ORDER ||--o| KITCHEN_TICKET : "produces"
    MENU_ITEM ||--o{ ORDER_ITEM : "is ordered as"
    MENU_ITEM ||--o{ RECIPE_INGREDIENT : "requires"
    INGREDIENT ||--o{ RECIPE_INGREDIENT : "used in"
    INGREDIENT ||--o{ STOCK_SYNC_EVENT : "reports"
    RESERVATION ||--o{ RESERVATION_LINE : "deducts"
    INGREDIENT ||--o{ RESERVATION_LINE : "affected by"

    CHANNEL {
        string id PK
        string name "app | tai-cho | doi-tac-giao-do-an"
        boolean accepting_new_orders "false khi channel tu chan nhan don do phat hien het nguyen lieu"
    }
    ORDER {
        string id PK
        string channel_id FK
        string status "pending | confirmed | cancelled"
        datetime created_at
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string menu_item_id FK
        int quantity
        string status "pending | confirmed | rejected"
    }
    MENU_ITEM {
        string id PK
        string name "Pho bo | Bun bo | ..."
        boolean is_available "tu dong an/hien theo is_low_stock cua nguyen lieu lien quan"
    }
    RECIPE_INGREDIENT {
        string id PK
        string menu_item_id FK
        string ingredient_id FK
        decimal quantity_required
    }
    INGREDIENT {
        string id PK
        string name "thit bo | ..."
        decimal stock_quantity
        int version "optimistic lock cho conditional update"
        decimal low_stock_threshold "vd du cho duoi 3 suat"
        boolean is_low_stock
        datetime updated_at
    }
    RESERVATION {
        string id PK
        string order_id FK
        string status "reserved | released | consumed"
        datetime reserved_at
        datetime released_at
    }
    RESERVATION_LINE {
        string id PK
        string reservation_id FK
        string ingredient_id FK
        decimal quantity_deducted
    }
    KITCHEN_TICKET {
        string id PK
        string order_id FK
        datetime printed_at
        datetime cooking_started_at "moc quyet dinh con duoc huy-hoan tra hay khong"
    }
    STOCK_SYNC_EVENT {
        string id PK
        string ingredient_id FK
        string channel_id FK
        decimal stock_quantity_at_event
        boolean is_low_stock_at_event
        datetime propagated_at
    }
```
