# Enhance sequence — Check following status (read quorum cao riêng cho relationship)

Đây là **enhance**, cùng flow "Check following status" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 2 của đề bài: dùng `QUORUM_POLICY(purpose=relationship_read)` riêng với quorum cao, đảm bảo đọc đúng ngay sau khi follow thành công.

```mermaid
sequenceDiagram
    actor User
    participant App as Social App
    participant Policy as QUORUM_POLICY store
    participant Rel as RELATIONSHIP_REPLICA

    User->>App: Vừa bấm Follow xong (API trả thành công)
    App->>App: Load lại trang, cần hiển thị đúng nút Follow/Following
    App->>Policy: Lấy quorum cho purpose=relationship_read (cao, khác với counter_read)
    App->>Rel: Đọc status từ đủ số node theo quorum cao
    Rel-->>App: Trả status=following (đảm bảo thấy đúng ghi gần nhất)
    App-->>User: Hiển thị đúng nút "Following"
    Note over Policy: Khác với đọc counter (chấp nhận trễ), đọc trạng thái quan hệ này luôn ưu tiên chính xác vì ảnh hưởng trực tiếp tới hành vi bấm nút của user
```
