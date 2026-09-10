# Sequence Diagram — Enhance: Write Record

Đây là **enhance**, flow ghi dữ liệu đã thay đổi so với base ([2-base-write-record-sqd.md](2-base-write-record-sqd.md)): mọi write nay phải ghi `WAL_ENTRY` và fsync xuống đĩa thành công trước khi trả ack, chỉ sau đó mới áp dụng vào cấu trúc dữ liệu chính, thay vì ghi trực tiếp như base.

```mermaid
sequenceDiagram
    actor Client
    participant Engine as Storage Engine
    participant WAL as WAL (disk)
    participant DataStruct as In-memory / B-tree Structure

    Client->>Engine: Write(key, value)
    Engine->>WAL: Append WAL_ENTRY (lsn, key, value, checksum)
    WAL->>WAL: fsync xuống đĩa

    alt fsync thành công
        WAL-->>Engine: Durable
        Engine->>DataStruct: Apply thay đổi vào cấu trúc chính
        DataStruct-->>Engine: Applied
        Engine-->>Client: Ack "thành công", durability đã đảm bảo
    else crash trước khi fsync xong
        Note over Engine,WAL: Client không nhận được ack, coi như write chưa xảy ra
    end
```
