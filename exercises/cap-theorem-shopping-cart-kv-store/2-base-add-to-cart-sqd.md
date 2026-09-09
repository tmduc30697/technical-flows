# Base sequence — Add to cart (W yêu cầu majority, mất availability lúc partition)

Đây là **base**, flow "Thêm sản phẩm vào giỏ hàng" ở trạng thái hiện tại — write_quorum yêu cầu đạt majority, nên phía minority bị từ chối hoàn toàn khi có partition. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài chính là đổi lại ưu tiên ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as E-commerce App
    participant Config as QUORUM_CONFIG store
    participant NodesMinority as CART_REPLICA (phía minority, đang partition)

    User->>App: Thêm sản phẩm vào giỏ hàng
    App->>Config: Lấy write_quorum (yêu cầu majority trên N node)
    App->>NodesMinority: Gửi ghi tới node phía minority
    NodesMinority-->>App: Không đạt write_quorum (thiếu node do partition)
    App-->>User: "Không thể thêm vào giỏ hàng lúc này"
    Note over App,NodesMinority: Chỉ vì đang có sự cố mạng tạm thời mà một hành động ít rủi ro như "thêm vào giỏ" cũng bị chặn hoàn toàn, ảnh hưởng trải nghiệm mua hàng
```
