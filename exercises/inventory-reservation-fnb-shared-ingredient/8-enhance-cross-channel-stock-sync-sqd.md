# Enhance sequence — Đồng bộ tồn kho liên kênh và chặn nhận đơn khi phát hiện hết hàng

Đây là **enhance**, flow hoàn toàn mới so với base (base không có khái niệm SLA đồng bộ hay tự chặn nhận đơn theo kênh). Đáp ứng yêu cầu 5 trong đề bài: quy định độ trễ đồng bộ chấp nhận được (ví dụ dưới 2 giây) giữa các kênh cùng dùng chung 1 kho vật lý, và cơ chế 1 kênh phát hiện nguyên liệu vừa hết phải lập tức chặn các kênh khác nhận đơn mới dựa trên số liệu đã lỗi thời.

```mermaid
sequenceDiagram
    participant PartnerChannel as Kenh Doi tac giao do an
    participant Stock as Ingredient Stock (DB)
    participant Broadcaster as Stock Sync Broadcaster
    participant AppChannel as Kenh App
    participant InstoreChannel as Kenh Tai cho

    PartnerChannel->>Stock: Xac nhan don cuoi cung lam "thit bo" ve 0
    Stock-->>PartnerChannel: stock_quantity = 0
    PartnerChannel->>Broadcaster: Publish STOCK_SYNC_EVENT (thit_bo, stock=0, is_low_stock=true)

    Note over Broadcaster: SLA dong bo quy dinh: moi kenh phai nhan va ap dung trong duoi 2 giay

    par Phat toi tat ca kenh con lai trong SLA
        Broadcaster->>AppChannel: propagated_at = t+300ms, set accepting_new_orders=false cho mon can thit bo
    and
        Broadcaster->>InstoreChannel: propagated_at = t+450ms, set accepting_new_orders=false cho mon can thit bo
    end

    Note over AppChannel,InstoreChannel: Trong khoang tre toi da 2 giay, don moi goi mon can thit bo van co the lot qua,<br/>nhung se bi tu choi ngay o buoc reservation atomic (flow confirm-order) chu khong bao gio bi xac nhan sai

    AppChannel-->>Broadcaster: ACK da ap dung, ghi nhan propagated_at thuc te de giam sat SLA
    InstoreChannel-->>Broadcaster: ACK da ap dung, ghi nhan propagated_at thuc te de giam sat SLA
```
