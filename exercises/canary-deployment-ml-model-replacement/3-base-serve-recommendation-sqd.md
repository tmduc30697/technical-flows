# Base sequence — Serve recommendation

Đây là **base**, flow "Serving gợi ý cho user" — nền tảng cho enhance: dữ liệu tương tác (click/mua) được gộp thẳng vào tập huấn luyện mà không đánh dấu model nào đã ảnh hưởng tới hành vi đó, tạo rủi ro feedback loop khi có nhiều model chạy song song sau này.

```mermaid
sequenceDiagram
    actor User
    participant App as Recommendation App
    participant Model as Model đang active
    participant DB as RECOMMENDATION_REQUEST store
    participant Training as TRAINING_DATASET store

    User->>App: Yêu cầu trang gợi ý sản phẩm
    App->>Model: Gọi model lấy danh sách gợi ý
    Model-->>App: Trả kết quả gợi ý
    App->>DB: Ghi RECOMMENDATION_REQUEST (model_id, latency_ms)
    App-->>User: Hiển thị gợi ý
    User->>App: Click / mua sản phẩm được gợi ý (nếu có)
    App->>DB: Cập nhật clicked/purchased trên RECOMMENDATION_REQUEST
    DB->>Training: Toàn bộ request (bất kể model nào serving) đều được đưa vào TRAINING_DATASET như nhau
    Note over Training: Không phân biệt dữ liệu này sinh ra từ model nào — nếu sau này có canary, hành vi bị ảnh hưởng bởi model mới sẽ lẫn vào dữ liệu huấn luyện tiếp theo mà không ai biết
```
