# Enhance sequence — Instant content-safety rollback

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: rollback tức thời khi phát hiện gợi ý bất thường/vi phạm chính sách, không cần chờ chu kỳ đánh giá business metric dài hạn như flow "Business metric evaluation".

```mermaid
sequenceDiagram
    actor User
    participant App as Recommendation App
    participant Canary as Model canary
    participant SafetyCheck as Content Safety Check
    participant Flag as CONTENT_SAFETY_FLAG store
    participant Serving as Serving Layer

    User->>App: Yêu cầu trang gợi ý (rơi vào nhóm canary)
    App->>Canary: Gọi model canary
    Canary-->>App: Trả kết quả gợi ý
    App->>SafetyCheck: Kiểm tra nhanh gợi ý có bất thường/vi phạm chính sách không
    alt Gợi ý bình thường
        SafetyCheck-->>App: OK
        App-->>User: Hiển thị gợi ý
    else Phát hiện gợi ý không liên quan hoặc vi phạm chính sách nội dung
        SafetyCheck->>Flag: Ghi CONTENT_SAFETY_FLAG(model_id=canary, flag_type)
        Flag->>Serving: Kích hoạt ROLLBACK_EVENT(triggered_by=content_safety_flag) ngay lập tức
        Serving->>Serving: Đưa traffic_percent của Canary về 0, toàn bộ traffic quay lại Stable
        Note over Serving: Không chờ đủ min_duration_hours hay phân tích business metric — an toàn nội dung được ưu tiên phản ứng tức thời
        App-->>User: Hiển thị gợi ý từ Stable thay thế (nếu request đang xử lý dở)
    end
```
