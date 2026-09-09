# Enhance sequence — Partition heal merge

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không cho phép ghi phía minority nên chưa từng có gì để merge). Đáp ứng yêu cầu 3 và 5 của đề bài: merge union khi 2 phía cùng sửa giỏ hàng khác nhau, không tự xóa item của bên nào, và đo tần suất conflict xảy ra thực tế.

```mermaid
sequenceDiagram
    participant NodesA as CART_REPLICA (phía A, trước partition)
    participant NodesB as CART_REPLICA (phía B, minority lúc partition)
    participant Merge as Merge Resolver
    participant Log as CART_CONFLICT_LOG store

    Note over NodesA,NodesB: Partition vừa hàn lại — cả 2 phía đều có thay đổi giỏ hàng khác nhau trong lúc bị chia cắt
    NodesA->>Merge: Gửi items_json phía A (vd thêm sản phẩm X)
    NodesB->>Merge: Gửi items_json phía B (vd thêm sản phẩm Y)

    Merge->>Merge: So sánh 2 phiên bản, phát hiện khác biệt
    Merge->>Merge: Áp dụng resolution=union_no_delete — hợp nhất cả X lẫn Y vào giỏ hàng, không tự xóa item nào
    Merge->>NodesA: Ghi lại items_json đã merge (gồm cả X và Y)
    Merge->>NodesB: Ghi lại items_json đã merge (gồm cả X và Y)
    Merge->>Log: Ghi CART_CONFLICT_LOG (side_a_items, side_b_items, merged_items, resolved_at)

    Note over Log: Log này được tổng hợp định kỳ thành conflict_rate (số lần merge / tổng số ghi) để đánh giá mức độ ảnh hưởng thực tế của việc chọn AP thay vì CP cho giỏ hàng
```
