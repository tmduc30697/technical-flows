# Base ERD — Message broker tự xây trước khi có consensus giữa các node

Đây là **base**: mô hình dữ liệu suy luận trước khi áp consensus. Đề bài mô tả vai trò flow là "consensus giữa các broker node quyết định thứ tự chính thức của message... khi leader broker chết và broker khác lên thay" — nghĩa là trước đó mỗi partition chỉ do đúng 1 broker phụ trách, không có replica, không có cơ chế bầu lại khi leader chết. Base chỉ cần đủ Topic/Partition/Message để publish-subscribe hoạt động, và offset commit của consumer được lưu ngay trên broker đang phụ trách partition đó (chính điểm này sẽ là vấn đề enhance phải giải quyết).

```mermaid
erDiagram
    TOPIC ||--o{ PARTITION : "divided into"
    PARTITION ||--o{ MESSAGE : "stores in order"
    CONSUMER_GROUP ||--o{ CONSUMER_OFFSET : "tracks per partition"
    PARTITION ||--o{ CONSUMER_OFFSET : "read progress of"

    TOPIC {
        string id PK
        string name
    }
    PARTITION {
        string id PK
        string topic_id FK
        string broker_id "broker duy nhất phụ trách"
        int next_offset
    }
    MESSAGE {
        string id PK
        string partition_id FK
        int offset
        string payload
        datetime published_at
    }
    CONSUMER_GROUP {
        string id PK
        string name
    }
    CONSUMER_OFFSET {
        string id PK
        string consumer_group_id FK
        string partition_id FK
        int committed_offset "lưu ngay trên broker phụ trách partition"
    }
```
