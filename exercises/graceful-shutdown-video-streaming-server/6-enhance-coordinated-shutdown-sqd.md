# Sequence - Enhance - Flow "coordinated-shutdown"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 4 trong đề bài: khi autoscale scale-in loạt lớn xảy ra gần như đồng thời với deploy, nhiều instance nhận tín hiệu shutdown cùng lúc. Nếu mỗi instance tự xử lý grace period độc lập như ở flow "instance-shutdown", toàn bộ client bị buộc reconnect có thể dồn vào cùng một thời điểm, gây spike tải lên các instance còn lại và CDN edge. Flow này thêm một coordinator đứng giữa để giãn cách các đợt shutdown.

```mermaid
sequenceDiagram
    participant Orchestrator as Autoscaler / Deploy tool
    participant Coordinator as Shutdown Coordinator
    participant I1 as Instance 1
    participant I2 as Instance 2
    participant I3 as Instance 3
    participant CDN as CDN Edge / Instance còn lại

    Orchestrator->>Coordinator: yêu cầu shutdown đồng thời cho instance 1, 2, 3
    Coordinator-->>Coordinator: gom thành 1 shutdown batch,\nchia instance vào các đợt (batch_index) có scheduled_at giãn cách

    Coordinator->>I1: lệnh shutdown, đợt 1, bắt đầu ngay
    activate I1
    Note over I1: I1 chạy flow instance-shutdown (phân loại, grace period, resume signal)
    deactivate I1

    Coordinator-->>Coordinator: theo dõi tải lên CDN edge / instance còn lại

    alt Tải vẫn ổn định
        Coordinator->>I2: lệnh shutdown, đợt 2, sau một khoảng trễ
    else Phát hiện tải tăng đột biến do đợt 1
        Coordinator-->>Coordinator: hoãn đợt 2, giãn thêm thời gian chờ
        Coordinator->>I2: lệnh shutdown, đợt 2, sau khi tải ổn định trở lại
    end

    Note over I2: I2 chạy flow instance-shutdown

    Coordinator->>I3: lệnh shutdown, đợt 3, tiếp tục giãn cách tương tự
    Note over I3: I3 chạy flow instance-shutdown

    I1-->>CDN: reconnect tới CDN/instance khác, rải đều theo từng đợt
    I2-->>CDN: reconnect tới CDN/instance khác, rải đều theo từng đợt
    I3-->>CDN: reconnect tới CDN/instance khác, rải đều theo từng đợt
```
