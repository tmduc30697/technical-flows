# Enhance sequence — Ring rebalance khi scale in/out

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base dùng round-robin nên không có khái niệm "ring" hay remap. Đáp ứng **yêu cầu 3** (khi thêm/bớt instance khỏi ring, số lượng key phải bị remap là tối thiểu, verify bằng test đo tỉ lệ % key bị đổi instance trước/sau khi thêm 1 node).

```mermaid
sequenceDiagram
    actor Ops as Kỹ sư vận hành
    participant Ring as HASH_RING
    participant TestTool as Ring Rebalance Test Tool
    participant InstanceD as Instance D (mới thêm)

    Ops->>TestTool: Chạy test đo remap trước khi scale out
    TestTool->>Ring: Snapshot toàn bộ mapping transaction_id hiện tại -> instance
    Ring-->>TestTool: Trả về bảng mapping trước khi thêm node

    Ops->>Ring: Thêm Instance D vào cluster
    Ring->>Ring: Sinh các virtual node mới cho Instance D, chèn vào đúng vị trí hash trên ring

    TestTool->>Ring: Snapshot lại toàn bộ mapping sau khi thêm node
    Ring-->>TestTool: Trả về bảng mapping mới
    TestTool->>TestTool: So sánh hai bảng mapping, đếm số key đổi instance
    TestTool->>Ring: Ghi RING_REBALANCE_EVENT(triggered_by=scale_out, keys_remapped_count, keys_total_count, remap_percentage)
    Note over TestTool,Ring: Chỉ các key rơi đúng vào khoảng hash mới của Instance D bị remap, phần lớn key khác vẫn giữ nguyên instance cũ, đúng đặc tính consistent hashing
    TestTool-->>Ops: Báo cáo remap_percentage xấp xỉ 1/N (N là số instance sau khi thêm), thay vì gần như toàn bộ như round-robin
```
