# Enhance sequence — Capacity benchmark per mode

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: benchmark rõ request/giây hệ thống chịu được ở từng mode quorum, phục vụ lập kế hoạch capacity trước khi flash sale diễn ra thật.

```mermaid
sequenceDiagram
    actor SRE as SRE/Capacity Planner
    participant LoadTest as Load Test Tool
    participant Policy as QUORUM_POLICY store
    participant Nodes as INVENTORY_NODE (môi trường staging/tương đương prod)
    participant Bench as CAPACITY_BENCHMARK store

    SRE->>Policy: Lấy từng mode_name (pre_sale, low_stock_danger)
    loop Với mỗi mode
        SRE->>Nodes: Áp policy tương ứng (R/W của mode đó) lên cụm test
        SRE->>LoadTest: Chạy tải tăng dần cho tới khi hệ thống bắt đầu suy giảm/lỗi
        LoadTest-->>SRE: Xác định max_requests_per_second hệ thống chịu được ở mode này
        SRE->>Bench: Ghi CAPACITY_BENCHMARK(mode_name, max_requests_per_second)
    end
    Bench-->>SRE: Bảng benchmark đầy đủ cho cả 2 mode
    Note over SRE: Dùng số liệu này để lập kế hoạch capacity thật — vd biết trước mode low_stock_danger chịu tải thấp hơn nhiều so với pre_sale do write_quorum cao hơn
```
