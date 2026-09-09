# Enhance sequence — Read cart (chọn mức consistency theo use case)

Đây là **enhance**, cùng flow "Read cart" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 4 của đề bài: client chọn read quorum khác nhau theo tình huống — R thấp cho icon header, R cao hơn cho checkout.

```mermaid
sequenceDiagram
    actor User
    participant App as E-commerce App
    participant Level as READ_CONSISTENCY_LEVEL store
    participant Nodes as CART_REPLICA

    User->>App: Xem icon số lượng giỏ hàng ở header
    App->>Level: Lấy READ_CONSISTENCY_LEVEL(use_case=header_icon) → read_quorum thấp
    App->>Nodes: Đọc nhanh từ R thấp
    Nodes-->>App: Trả items_json (chấp nhận trễ nhẹ)
    App-->>User: Hiển thị số lượng

    User->>App: Vào trang checkout
    App->>Level: Lấy READ_CONSISTENCY_LEVEL(use_case=checkout) → read_quorum cao hơn
    App->>Nodes: Đọc từ R cao, đảm bảo thấy đúng ghi gần nhất hơn
    Nodes-->>App: Trả items_json chính xác hơn
    App-->>User: Hiển thị giỏ hàng đáng tin cậy để tiến hành thanh toán
    Note over Level: Cùng 1 dữ liệu giỏ hàng, nhưng mức đảm bảo đọc được chọn khác nhau tuỳ mức độ quan trọng của tình huống
```
