# Base sequence — Read cart (cùng 1 read_quorum cho mọi use case)

Đây là **base**, flow "Đọc giỏ hàng" — dùng chung đúng 1 read_quorum cho mọi mục đích, từ hiển thị icon số lượng ở header tới đọc lúc checkout. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 4 của đề bài chính là cho phép phân biệt mức consistency theo tình huống.

```mermaid
sequenceDiagram
    actor User
    participant App as E-commerce App
    participant Config as QUORUM_CONFIG store
    participant Nodes as CART_REPLICA

    User->>App: Xem icon số lượng giỏ hàng ở header
    App->>Config: Lấy read_quorum (dùng chung, cố định)
    App->>Nodes: Đọc từ R node theo config chung
    Nodes-->>App: Trả items_json
    App-->>User: Hiển thị số lượng

    User->>App: Vào trang checkout, xem chi tiết giỏ hàng
    App->>Config: Lấy read_quorum — vẫn cùng 1 giá trị như trên, không cao hơn
    App->>Nodes: Đọc từ R node (giống hệt lúc hiển thị header)
    Nodes-->>App: Trả items_json (có thể vẫn stale nếu R thấp)
    App-->>User: Hiển thị giỏ hàng lúc checkout
    Note over App,Nodes: Đọc lúc checkout đáng lẽ cần chắc chắn hơn đọc icon header, nhưng hiện tại không có cơ chế nào phân biệt
```
