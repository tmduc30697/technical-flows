# Enhance sequence — Deploy model (canary song song, đủ thời gian đánh giá)

Đây là **enhance**, cùng flow "Deploy model" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 4 của đề bài: model mới chạy song song với model cũ trên 1 phần traffic thật, và canary phải chạy đủ lâu (qua cả ngày thường lẫn cuối tuần) trước khi kết luận.

```mermaid
sequenceDiagram
    participant MLOps as MLOps Engineer
    participant Serving as Serving Layer
    participant Stable as Model stable (đang chạy)
    participant Canary as Model canary (mới train)
    participant Policy as CANARY_EVALUATION_POLICY store

    MLOps->>Serving: Deploy Model mới là Canary, giữ nguyên Stable song song
    Serving->>Policy: Lấy min_duration_hours (vd 168 giờ, đủ 1 tuần)
    Serving->>Serving: Chia 1 phần nhỏ traffic thật cho Canary, phần còn lại vẫn ở Stable
    loop Suốt min_duration_hours
        Serving->>Stable: Phục vụ phần lớn traffic
        Serving->>Canary: Phục vụ phần traffic nhỏ dành cho canary
    end
    Note over Serving,Policy: Không kết luận model nào tốt hơn cho tới khi đủ thời gian tối thiểu — tránh quyết định dựa trên mẫu ngắn/thiên lệch theo ngày thường-cuối tuần
    Note over MLOps: Kết quả đánh giá cuối cùng nằm ở flow "Business metric evaluation" sau khi đủ thời gian, hoặc rollback sớm nếu có ROLLBACK_EVENT
```
