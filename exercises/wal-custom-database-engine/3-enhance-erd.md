# ERD — Enhance (sau khi có WAL)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — ghi WAL fsync trước khi ack, replay đúng thứ tự sau crash, phát hiện torn write bằng checksum, checkpoint định kỳ an toàn, và đo recovery time theo kích thước WAL. So với base, `RECORD` nay chỉ được cập nhật thông qua `WAL_ENTRY` đã fsync thành công, không còn ghi trực tiếp.

```mermaid
erDiagram
    WAL_ENTRY ||--o| RECORD : "applies to"
    CHECKPOINT ||--o{ WAL_ENTRY : "covers up to"

    RECORD {
        string key PK
        string value
        datetime updated_at
    }

    WAL_ENTRY {
        bigint lsn PK
        string key
        string value
        string operation
        string checksum
        string status
        datetime written_at
    }

    CHECKPOINT {
        bigint checkpoint_id PK
        bigint up_to_lsn
        string status
        datetime started_at
        datetime completed_at
    }
```
