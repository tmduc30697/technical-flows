# Sequence Diagram — Base: Write Record

Đây là **base**, flow ghi dữ liệu trực tiếp vào cấu trúc chính, không qua log tuần tự nào — tiền đề để thấy rõ rủi ro mất dữ liệu khi crash, chính là lý do đề bài yêu cầu thêm WAL.

```mermaid
sequenceDiagram
    actor Client
    participant Engine as Storage Engine
    participant DataStruct as In-memory / B-tree Structure

    Client->>Engine: Write(key, value)
    Engine->>DataStruct: Apply thay đổi trực tiếp
    DataStruct-->>Engine: Applied
    Engine-->>Client: Ack "thành công"

    Note over Engine,DataStruct: Nếu crash ngay sau ack nhưng trước khi OS flush buffer xuống đĩa, thay đổi này mất hoàn toàn
```
