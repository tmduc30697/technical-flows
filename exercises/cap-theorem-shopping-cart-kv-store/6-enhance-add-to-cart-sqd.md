# Enhance sequence — Add to cart (ưu tiên availability, chấp nhận ghi minority)

Đây là **enhance**, cùng flow "Add to cart" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 2 của đề bài: chấp nhận ghi ở phía minority lúc partition, merge để sau, thay vì từ chối như base.

```mermaid
sequenceDiagram
    actor User
    participant App as E-commerce App
    participant Policy as PARTITION_WRITE_POLICY store
    participant NodesMinority as CART_REPLICA (phía minority, đang partition)

    User->>App: Thêm sản phẩm vào giỏ hàng
    App->>Policy: Kiểm tra PARTITION_WRITE_POLICY(action=add_to_cart)
    Policy-->>App: accept_minority_writes=true
    App->>NodesMinority: Ghi vào node phía minority, không chờ đạt majority
    NodesMinority-->>App: Ghi thành công cục bộ ngay
    App-->>User: "Đã thêm vào giỏ hàng"
    Note over NodesMinority: Trải nghiệm mua hàng được ưu tiên — user không hề biết đang có partition, việc merge sẽ xử lý sau khi mạng hàn lại (xem flow "Partition heal merge")
```
