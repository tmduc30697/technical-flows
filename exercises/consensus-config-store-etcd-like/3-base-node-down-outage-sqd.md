# Base sequence — Node duy nhất chết gây outage toàn cụm

Đây là **base**, mô tả hệ quả trực tiếp của kiến trúc single-node ở flow trước — khi node duy nhất chết, toàn bộ hàng trăm service phụ thuộc config đều mất khả năng đọc/ghi. Đây chính là lý do đề bài yêu cầu dựng Raft cluster 5 node chịu được tối đa 2 node down (yêu cầu 1 của đề bài).

```mermaid
sequenceDiagram
    actor Svc as Internal Service
    participant Node as Config Node (duy nhất, đã crash)

    Svc->>Node: GET config_entry(key)
    Node--xSvc: Không phản hồi (node đã crash)
    Svc-->>Svc: Timeout, service không đọc được config/feature flag

    Note over Svc,Node: Hàng trăm service nội bộ phụ thuộc config đều bị ảnh hưởng cùng lúc, không có node dự phòng nào thay thế
```
