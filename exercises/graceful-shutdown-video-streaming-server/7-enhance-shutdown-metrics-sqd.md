# Sequence - Enhance - Flow "shutdown-metrics"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 5 trong đề bài: mỗi khi một stream session kết thúc, hệ thống phải ghi nhận rõ lý do kết thúc (người dùng chủ động dừng, hoàn tất tự nhiên trong lúc drain, hay bị buộc ngắt do shutdown) để tách riêng được tỷ lệ ảnh hưởng thực sự của draining, thay vì gộp chung vào số liệu drop-off thông thường.

```mermaid
sequenceDiagram
    participant Instance
    participant Metrics as Metrics Pipeline
    participant Alerting

    Note over Instance: Session kết thúc (3 trường hợp)

    alt Người dùng chủ động dừng xem
        Instance->>Metrics: ghi disconnect_reason = user_stop
    else Session tự hoàn tất trong lúc grace period
        Instance->>Metrics: ghi disconnect_reason = completed_naturally
    else Session bị buộc ngắt do hết grace period
        Instance->>Metrics: ghi disconnect_reason = shutdown_forced
    end

    Metrics-->>Metrics: tổng hợp riêng tỷ lệ shutdown_forced / tổng session,\ntách khỏi drop-off thông thường (user_stop)

    loop Theo mỗi cửa sổ thời gian (ví dụ mỗi 1 phút trong lúc deploy/scale-in)
        Metrics->>Alerting: cập nhật tỷ lệ shutdown_forced hiện tại
        alt Tỷ lệ vượt ngưỡng cho phép
            Alerting-->>Alerting: cảnh báo, nghi ngờ draining đang ảnh hưởng người xem\nvượt mức chấp nhận được, ví dụ resume thất bại hàng loạt
        else Tỷ lệ trong ngưỡng bình thường
            Alerting-->>Alerting: không cảnh báo, draining diễn ra như thiết kế
        end
    end
```
