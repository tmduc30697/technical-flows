# Enhance sequence — Read session (AP, cảnh báo staleness bất thường)

Đây là **enhance**, cùng flow "Read session" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 3 của đề bài: vẫn giữ AP cho tốc độ, nhưng thêm phát hiện và cảnh báo khi staleness vượt ngưỡng bất thường (vd 1 node bị treo lâu).

```mermaid
sequenceDiagram
    actor User
    participant App as Checkout App
    participant Policy as CONSISTENCY_POLICY store
    participant Nodes as SESSION_REPLICA
    participant Monitor as Staleness Monitor
    participant Alert as STALENESS_ALERT store
    actor Ops as Vận hành

    User->>App: Xem trạng thái giỏ hàng/checkout
    App->>Policy: Kiểm tra endpoint read-session → mode=AP
    App->>Nodes: Đọc từ node gần nhất bất kỳ
    Nodes-->>App: Trả status kèm updated_at
    App-->>User: Hiển thị trạng thái

    Monitor->>Nodes: Định kỳ so sánh updated_at của từng node với node mới nhất
    alt Staleness trong ngưỡng bình thường
        Monitor->>Monitor: Không làm gì thêm
    else Staleness vượt ngưỡng bất thường (vd node bị treo lâu)
        Monitor->>Alert: Ghi STALENESS_ALERT(node_id, staleness_ms, threshold_exceeded_at)
        Alert-->>Ops: Cảnh báo để can thiệp (kiểm tra node bị treo)
    end
```
