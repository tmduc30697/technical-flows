# Sequence Diagram - Enhance: trigger-deploy

Đây là flow **enhance** của `trigger-deploy` (so với base ở file `2-base-trigger-deploy-sqd.md`): trước khi deploy thật, pipeline phải xin lock toàn cục từ coordinator ensemble (ZooKeeper/etcd, chịu lỗi cao, độc lập với mọi region), chỉ khi acquire thành công mới được deploy, và toàn bộ vòng đời lock/deploy đều được audit log lại. Đáp ứng yêu cầu 1 (coordinator chịu lỗi cao, độc lập region) và yêu cầu 3 (audit đầy đủ region/thời điểm/người trigger).

```mermaid
sequenceDiagram
    actor Dev as Developer / CI trigger
    participant CD as Pipeline CI/CD (Region A)
    participant Coord as Lock Coordinator ensemble (ZK/etcd)
    participant Audit as Audit Log
    participant Infra as Hạ tầng Region A

    Dev->>CD: Push code, trigger deploy
    CD->>CD: Build, test, đóng gói artifact
    CD->>Coord: Request acquire lock "global-deploy-lock"
    Coord->>Coord: Đồng thuận quorum giữa các node trong ensemble
    Coord-->>CD: Cấp lock, kèm fencing_token, lease_ttl
    Coord->>Audit: Ghi log LOCK_ACQUIRED (region=A, actor=Dev, thời điểm)
    CD->>Infra: Deploy artifact (kèm fencing_token)
    Infra-->>CD: Deploy thành công
    CD->>Coord: Release lock
    Coord->>Audit: Ghi log LOCK_RELEASED, DEPLOY_COMPLETED (region=A, thời điểm)
    CD-->>Dev: Thông báo deploy Region A hoàn tất
```
