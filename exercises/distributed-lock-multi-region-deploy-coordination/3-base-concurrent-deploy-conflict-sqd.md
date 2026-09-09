# Sequence Diagram - Base: concurrent-deploy-conflict

Đây là flow **base** minh hoạ chính vấn đề mà đề bài muốn giải quyết: khi 2 pipeline ở 2 region khác nhau bị trigger gần như đồng thời, cả 2 đều deploy song song vì không có cơ chế nào ngăn cản, dẫn tới trạng thái không đồng nhất giữa các region (ví dụ region A đã lên version mới, region B vẫn đang deploy version khác gây lệch dữ liệu/schema). Flow này là lý do enhance cần một distributed lock toàn cục.

```mermaid
sequenceDiagram
    actor DevA as Developer/CI (trigger A)
    actor DevB as Developer/CI (trigger B)
    participant CDA as Pipeline Region A
    participant CDB as Pipeline Region B
    participant InfraA as Hạ tầng Region A
    participant InfraB as Hạ tầng Region B

    DevA->>CDA: Trigger deploy Region A
    DevB->>CDB: Trigger deploy Region B (gần như cùng lúc)
    par Deploy Region A
        CDA->>InfraA: Deploy version mới
        InfraA-->>CDA: OK
    and Deploy Region B
        CDB->>InfraB: Deploy version khác/cùng lúc
        InfraB-->>CDB: OK
    end
    Note over InfraA,InfraB: Không ai kiểm tra region kia đang deploy, dữ liệu/schema giữa 2 region có thể lệch nhau
```

