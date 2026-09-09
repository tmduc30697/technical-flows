# Sequence Diagram - Enhance: coordinator-partition-safety

Đây là flow **enhance mới**: khi coordinator mất kết nối tạm thời với 1 region, pipeline ở region đó không được tự cho là "an toàn để deploy" chỉ vì không nhận được lỗi rõ ràng - nó phải chủ động xác nhận lock hợp lệ (qua fencing_token còn hiệu lực) trước khi tiến hành, và huỷ/tạm dừng nếu không xác nhận được. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    actor Dev as Developer / CI trigger
    participant CD as Pipeline CI/CD (Region B)
    participant Coord as Lock Coordinator ensemble (ZK/etcd)
    participant Audit as Audit Log
    participant Infra as Hạ tầng Region B

    Dev->>CD: Trigger deploy Region B
    CD->>Coord: Request acquire lock "global-deploy-lock"
    Note over CD,Coord: Kết nối giữa Region B và coordinator bị gián đoạn tạm thời
    CD--xCoord: Timeout, không nhận được phản hồi acquire
    CD->>CD: Không giả định lock đã được cấp, chuyển sang trạng thái chờ xác nhận
    CD->>Coord: Retry kiểm tra trạng thái lock hiện tại
    Coord-->>CD: Kết nối phục hồi, xác nhận lock chưa được cấp cho Region B
    alt Không xác nhận được lock hợp lệ trong thời gian cho phép
        CD->>Audit: Ghi log DEPLOY_ABORTED (region=B, lý do=không xác nhận được lock)
        CD-->>Dev: Huỷ deploy, báo lỗi không an toàn để tiếp tục
    else Coordinator xác nhận cấp lock hợp lệ kèm fencing_token
        CD->>Infra: Deploy artifact (kèm fencing_token)
        Infra-->>CD: Deploy thành công
        CD->>Audit: Ghi log DEPLOY_COMPLETED (region=B)
    end
```
