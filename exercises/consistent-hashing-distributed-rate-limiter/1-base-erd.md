# Base ERD — Rate limiter cluster trước khi có consistent hashing đồng bộ

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái **trước khi** cụm dùng chung 1 hash ring nhất quán. Base đã có cụm nhiều node đếm rate limit và nhiều router instance đứng trước cụm để định tuyến request, nhưng mỗi router instance giữ 1 bản `ROUTING_TABLE` cục bộ riêng (có thể không đồng bộ với nhau, đúng như rủi ro đề bài nêu ở yêu cầu 1) thay vì tra chung 1 hash ring duy nhất. Việc route dùng hashing đơn giản kiểu modulo N node, nên khi cụm scale sẽ làm hầu hết key bị remap. Chưa có khái niệm migrate counter, chưa có xử lý hot key.

```mermaid
erDiagram
    ROUTER_INSTANCE ||--o{ ROUTING_TABLE_ENTRY : "keeps local copy"
    ROUTING_TABLE_ENTRY }o--|| NODE : "maps key range to"
    NODE ||--o{ RATE_LIMIT_COUNTER : "holds"
    API_KEY ||--o{ RATE_LIMIT_COUNTER : "counted in"

    ROUTER_INSTANCE {
        string id PK
        string routing_table_version
    }
    ROUTING_TABLE_ENTRY {
        string id PK
        string router_instance_id FK
        string key_range
        string node_id FK
    }
    NODE {
        string id PK
        string status
    }
    API_KEY {
        string id PK
        string owner
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string api_key_id FK
        string node_id FK
        datetime window_start
        int count
        int limit_per_window
    }
```
