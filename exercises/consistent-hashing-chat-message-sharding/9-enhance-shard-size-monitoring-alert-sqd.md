# Enhance sequence — Shard size monitoring & alert

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu 5 của đề bài: đo độ lệch kích thước dữ liệu giữa các shard theo thời gian và cảnh báo khi 1 shard vượt X% dung lượng trung bình để chủ động rebalance trước khi đầy.

```mermaid
sequenceDiagram
    participant AlertJob as Shard Size Monitor
    participant Shards as SHARD (toàn bộ)
    participant Metric as SHARD_SIZE_METRIC store
    actor OnCall as On-call Engineer

    loop Định kỳ (ví dụ mỗi giờ)
        AlertJob->>Shards: Lấy size_bytes hiện tại của từng shard
        Shards-->>AlertJob: Danh sách size_bytes theo shard_id
        AlertJob->>Metric: Ghi lại độ lệch kích thước giữa các shard tại thời điểm này
        AlertJob->>AlertJob: Tính trung bình size_bytes toàn cụm
        alt Có shard vượt alert_threshold_pct so với trung bình
            AlertJob-->>OnCall: Cảnh báo shard X sắp đầy, cần rebalance hoặc tách hot conversation
        else Toàn bộ shard trong ngưỡng cho phép
            AlertJob->>AlertJob: Không làm gì thêm
        end
    end

    Metric-->>OnCall: Báo cáo định kỳ độ lệch kích thước dữ liệu giữa các shard theo thời gian
```
