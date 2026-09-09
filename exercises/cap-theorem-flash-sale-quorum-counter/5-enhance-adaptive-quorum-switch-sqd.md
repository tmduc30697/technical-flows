# Enhance sequence — Adaptive quorum switch (R thấp → W cao)

Đây là **enhance**, hợp nhất 2 flow "Read stock" và "Decrement stock" đã có ở base thành 1 luồng thích ứng, theo yêu cầu 1 và 2 của đề bài: dùng R thấp trước sale, tự chuyển sang W cao khi tồn kho xuống dưới ngưỡng nguy hiểm.

```mermaid
sequenceDiagram
    actor User
    participant App as E-commerce App
    participant Policy as QUORUM_POLICY store
    participant Monitor as Stock Threshold Monitor
    participant Nodes as INVENTORY_NODE

    Note over Monitor: Trước giờ flash sale
    User->>App: Xem tồn kho
    App->>Policy: Lấy policy mode=pre_sale (read_quorum thấp)
    App->>Nodes: Đọc từ R thấp node
    Nodes-->>App: Trả stock_value nhanh, chấp nhận sai lệch nhỏ
    App-->>User: Hiển thị tồn kho

    Monitor->>Nodes: Theo dõi liên tục stock_value / initial_stock
    alt Tồn kho còn trên ngưỡng nguy hiểm (vd >5%)
        Monitor->>App: Vẫn dùng policy mode=pre_sale cho cả đọc lẫn trừ kho
        User->>App: Mua hàng
        App->>Nodes: Trừ kho với write_quorum mặc định
        Nodes-->>App: Ghi thành công
    else Tồn kho xuống dưới ngưỡng nguy hiểm (vd <5%)
        Monitor->>Policy: Kích hoạt chuyển sang mode=low_stock_danger (write_quorum cao hơn)
        Policy-->>App: Áp dụng policy mới ngay cho API trừ kho
        User->>App: Mua hàng
        App->>Nodes: Trừ kho với write_quorum cao — chờ nhiều node xác nhận hơn
        Nodes-->>App: Ghi thành công (chấp nhận latency tăng để giảm rủi ro oversell)
        App-->>User: Mua hàng thành công
    end
```
