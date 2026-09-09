# Enhance sequence — Training data feedback-loop guard

Đây là **enhance**, cùng flow "Serve recommendation" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 3 của đề bài: gắn nhãn rõ dữ liệu tương tác theo model nào đã serving, tránh dữ liệu bị ảnh hưởng bởi canary lẫn vào train model tương lai một cách không kiểm soát.

```mermaid
sequenceDiagram
    actor User
    participant App as Recommendation App
    participant Serving as Serving Layer (Stable hoặc Canary)
    participant DB as RECOMMENDATION_REQUEST store
    participant Label as TRAINING_DATA_LABEL store
    participant Training as Training Pipeline

    User->>App: Yêu cầu trang gợi ý
    App->>Serving: Gọi model đang được route tới (stable hoặc canary)
    Serving-->>App: Trả kết quả gợi ý, kèm role của model (stable/canary)
    App->>DB: Ghi RECOMMENDATION_REQUEST(model_id)
    App-->>User: Hiển thị gợi ý
    User->>App: Click/mua sản phẩm (nếu có)
    App->>DB: Cập nhật clicked/purchased

    App->>Label: Tạo TRAINING_DATA_LABEL(served_by_model_role)
    alt Dữ liệu sinh ra từ Canary trong giai đoạn đang đánh giá
        Label->>Label: excluded_from_training=true (tạm loại khỏi tập train tiếp theo)
    else Dữ liệu sinh ra từ Stable
        Label->>Label: excluded_from_training=false
    end

    Training->>Label: Chỉ lấy dữ liệu excluded_from_training=false để train model tương lai
    Note over Training: Nhờ vậy hành vi user bị ảnh hưởng bởi canary không âm thầm làm thiên lệch dữ liệu huấn luyện tiếp theo
```
