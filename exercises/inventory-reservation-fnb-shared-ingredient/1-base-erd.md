# Base ERD — Order F&B trước khi có atomic reservation nguyên liệu dùng chung

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống order F&B **trước khi** áp cơ chế trừ tồn nguyên tử. Đề bài nói tới nhiều kênh bán (app, tại chỗ, đối tác giao đồ ăn) cùng đặt món có công thức dùng chung nguyên liệu từ 1 kho bếp — nên base cần đủ: món ăn, công thức (recipe) liên kết món với nguyên liệu và định lượng cần dùng, kho nguyên liệu có tồn hiện tại, và đơn hàng/dòng đơn ghi nhận món được gọi theo từng kênh. Ở base, việc trừ tồn được giả định làm đơn giản theo từng dòng đơn riêng lẻ, chưa có khái niệm transaction atomic gộp nhiều nguyên liệu/nhiều món, chưa có ngưỡng cảnh báo sắp hết, chưa có cơ chế hoàn trả có điều kiện thời gian — những phần đó là enhance.

```mermaid
erDiagram
    CHANNEL ||--o{ ORDER_ITEM : "orders"
    ORDER ||--o{ ORDER_ITEM : contains
    MENU_ITEM ||--o{ ORDER_ITEM : "is ordered as"
    MENU_ITEM ||--o{ RECIPE_INGREDIENT : "requires"
    INGREDIENT ||--o{ RECIPE_INGREDIENT : "used in"

    CHANNEL {
        string id PK
        string name "app | tai-cho | doi-tac-giao-do-an"
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
        boolean is_available
    }
    RECIPE_INGREDIENT {
        string id PK
        string menu_item_id FK
        string ingredient_id FK
        decimal quantity_required "so luong nguyen lieu can cho 1 suat"
    }
    INGREDIENT {
        string id PK
        string name "thit bo | ..."
        decimal stock_quantity "ton hien tai"
        decimal updated_at
    }
```
