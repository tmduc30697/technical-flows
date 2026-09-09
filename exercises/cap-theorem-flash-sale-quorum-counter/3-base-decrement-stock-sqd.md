# Base sequence — Decrement stock (chưa nhận diện partition/ngưỡng nguy hiểm)

Đây là **base**, flow "Trừ tồn kho khi mua hàng" ở trạng thái hiện tại — dùng W cố định, không kiểm tra ngưỡng tồn kho nguy hiểm, không xử lý riêng khi có network partition. Flow này liên quan mật thiết tới enhance vì yêu cầu 2 và 3 của đề bài chính là sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as Checkout Service
    participant Config as QUORUM_CONFIG store
    participant NodesA as Nhóm node A (majority)
    participant NodesB as Nhóm node B (minority, nếu có partition)

    User->>App: Xác nhận mua hàng
    App->>Config: Lấy write_quorum (W) cố định
    App->>NodesA: Ghi trừ kho vào W node bất kỳ có thể tiếp cận
    NodesA-->>App: Ghi thành công
    App-->>User: Mua hàng thành công

    Note over NodesB: Nếu đang có network partition, node ở nhóm minority không biết mình đang bị cô lập — vẫn có thể tự nhận request và trừ kho độc lập, dẫn tới đếm sai khi partition hàn lại
    Note over Config: W không tự tăng dù tồn kho đã xuống rất thấp, rủi ro oversell cao nhất lại đang dùng cùng mức đảm bảo như lúc tồn kho còn nhiều
```
