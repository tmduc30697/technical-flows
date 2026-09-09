# Base sequence — Deploy (chuyển thẳng 100% traffic, không phân biệt loại giao dịch, không phê duyệt)

Đây là **base**, flow "Deploy core banking service" ở trạng thái hiện tại: sau khi rolling deploy xong, router chuyển thẳng 100% traffic sang bản mới cho mọi loại giao dịch (cả xem số dư lẫn chuyển tiền) cùng lúc, không có giai đoạn tăng dần theo mức độ rủi ro, không cần ai phê duyệt, và không ghi lại quyết định deploy vào đâu cả. Flow này là nền để so sánh với yêu cầu 1, 2 và 5 của đề bài.

```mermaid
sequenceDiagram
    actor Dev as Kỹ sư triển khai
    participant CI as CI/CD
    participant Old as Deployment cũ (stable)
    participant New as Deployment mới
    participant Router as Router
    actor Customer

    Dev->>CI: Kích hoạt deploy bản mới
    CI->>New: Rolling deploy bản mới, thay thế dần instance cũ
    CI->>Router: Trỏ 100% traffic sang bản mới ngay khi rolling xong
    Note over Router: Không phân biệt giao dịch xem số dư hay chuyển tiền, cả 2 loại đều đi qua bản mới ngay lập tức
    Note over CI,Dev: Không có bước phê duyệt nào, cũng không ghi lại ai deploy, deploy lúc nào, tỷ lệ traffic từng thời điểm

    Customer->>Router: Gửi request xem số dư hoặc chuyển tiền
    Router->>New: Toàn bộ request đều xử lý bởi bản mới
    New-->>Customer: Trả kết quả
```
