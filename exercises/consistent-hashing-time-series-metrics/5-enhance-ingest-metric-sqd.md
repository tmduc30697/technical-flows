# Enhance sequence — Ingest metric (composite key hash(service) + time bucket)

Đây là **enhance** của flow `ingest-metric` đã có ở base. So với base, router không còn chỉ dựa vào `time_bucket` mà tính thêm `hash(service_id)` để chọn đúng partition trong composite key, nên các service khác nhau trong cùng khung giờ được rải đều ra nhiều partition/node khác nhau. Đáp ứng yêu cầu 1 (tránh hot partition bằng composite key) và yêu cầu 4 (số node/partition được tính theo throughput mục tiêu).

```mermaid
sequenceDiagram
    actor SvcA as Service A (bình thường)
    actor SvcB as Service B (đẩy metrics dồn dập)
    participant Router as Ingest Router
    participant Cap as CLUSTER_CAPACITY_CONFIG
    participant PA as PARTITION (hash_range=A, time_bucket=09:00-10:00)
    participant PB as PARTITION (hash_range=B, time_bucket=09:00-10:00)

    Router->>Cap: Lấy required_node_count, required_partition_count (mục tiêu 1 triệu điểm/giây toàn cụm)
    Cap-->>Router: required_node_count=N, required_partition_count=M

    SvcA->>Router: Ghi metric point (ts=09:15)
    Router->>Router: Tính composite key = hash(service_id=A) + time_bucket=09:00-10:00
    Router->>PA: Ghi vào partition A (node riêng)
    PA-->>Router: OK

    loop Hàng nghìn điểm dữ liệu mỗi giây
        SvcB->>Router: Ghi metric point (ts=09:16)
        Router->>Router: Tính composite key = hash(service_id=B) + time_bucket=09:00-10:00
        Router->>PB: Ghi vào partition B (node khác, không đụng Service A)
    end
    Note over PA,PB: Vì hash(service_id) khác nhau, dữ liệu 2 service rơi vào 2 partition/node độc lập, Service B đẩy dồn dập cũng không ảnh hưởng độ trễ ghi của Service A
    Note over Router,Cap: Với target 1 triệu điểm/giây và per_node_capacity ước lượng theo benchmark, required_node_count/required_partition_count được tính trước để đảm bảo throughput ghi tổng của cụm
```
