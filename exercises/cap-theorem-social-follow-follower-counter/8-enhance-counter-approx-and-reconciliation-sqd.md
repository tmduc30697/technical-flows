# Enhance sequence — Approximate counter update & reconciliation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base cập nhật counter đồng bộ trực tiếp). Đáp ứng yêu cầu 1 và 5 của đề bài: counter hiển thị cập nhật bất đồng bộ theo batch, chấp nhận sai lệch tạm thời, và có đối soát định kỳ với ground truth để tự điều chỉnh khi lệch vượt ngưỡng.

```mermaid
sequenceDiagram
    participant Rel as FOLLOW_RELATIONSHIP store
    participant Job as COUNTER_UPDATE_JOB store
    participant Counter as FOLLOWER_COUNT store
    actor User
    participant Reconcile as Reconciliation Job
    participant Report as COUNTER_RECONCILIATION_REPORT store

    Rel->>Job: Mỗi lần follow/unfollow thành công, enqueue batch_delta (+1/-1)
    loop Định kỳ mỗi vài giây tới vài phút
        Job->>Counter: Áp dụng gộp batch_delta vào approximate_count
        Counter->>Counter: last_batch_applied_at = now
    end
    User->>Counter: Xem follower count trên profile
    Counter-->>User: Trả approximate_count (chấp nhận sai lệch tạm thời vài giây tới vài phút)

    Note over Reconcile: Chạy định kỳ (vd mỗi giờ)
    Reconcile->>Rel: Đếm trực tiếp số bản ghi status=following (ground truth)
    Reconcile->>Counter: Lấy approximate_count hiện tại
    Reconcile->>Reconcile: Tính drift = |approximate_count - ground_truth_count|
    alt drift trong ngưỡng cho phép
        Reconcile->>Report: Ghi COUNTER_RECONCILIATION_REPORT(auto_corrected=false)
    else drift vượt ngưỡng cho phép
        Reconcile->>Counter: Ghi đè approximate_count = ground_truth_count
        Reconcile->>Report: Ghi COUNTER_RECONCILIATION_REPORT(auto_corrected=true)
    end
```
