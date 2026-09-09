# Enhance sequence — Get/set key (đúng trong lúc resharding, dual-write)

Đây là **enhance**, cùng flow "get-set-key" đã có ở base nhưng nay thay đổi: node phụ trách được tra qua `HASH_RING`, và khi key đang nằm trong `MIGRATION_JOB` đang chạy, hệ thống dual-write (ghi cả node cũ lẫn node mới) và route đọc theo đúng nguồn còn xác thực, để không mất update và không đọc dữ liệu cũ. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor Service as Service gọi cache
    participant Ring as HASH_RING
    participant Job as MIGRATION_JOB (key K1 đang di chuyển)
    participant NodeOld as NODE cũ
    participant NodeNew as NODE mới

    Service->>Ring: SET key=K1, value=V2
    Ring->>Job: Kiểm tra K1 có đang trong migration đang chạy không
    Job-->>Ring: Có, dual_write_active=true, from=NodeOld, to=NodeNew
    par Dual-write trong lúc resharding
        Ring->>NodeOld: Ghi CACHE_ENTRY(K1, V2)
        Ring->>NodeNew: Ghi CACHE_ENTRY(K1, V2)
    end
    NodeOld-->>Service: OK
    Note over NodeOld,NodeNew: Ghi cả 2 phía đảm bảo không mất update dù copy nền của migration đang diễn ra song song

    Service->>Ring: GET key=K1
    Ring->>Job: Kiểm tra trạng thái migration của K1
    Job-->>Ring: Chưa hoàn tất, nguồn xác thực hiện tại vẫn là NodeOld
    Ring->>NodeOld: Đọc CACHE_ENTRY(K1)
    NodeOld-->>Service: V2 (giá trị mới nhất, không phải dữ liệu cũ)
```
