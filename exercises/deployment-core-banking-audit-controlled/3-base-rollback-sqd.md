# Base sequence — Rollback (chỉ rollback mã nguồn, chưa tính tới dữ liệu)

Đây là **base**, flow rollback ở trạng thái hiện tại: khi phát hiện lỗi, kỹ sư chỉ rollback mã nguồn về bản cũ, không có kế hoạch xử lý dữ liệu mà bản mới đã ghi ở định dạng mới trong lúc đang chạy. Đây chính là khoảng trống mà yêu cầu 4 của đề bài nhắm tới xử lý.

```mermaid
sequenceDiagram
    actor Dev as Kỹ sư triển khai
    participant CI as CI/CD
    participant New as Deployment mới (đang chạy)
    participant Old as Deployment cũ
    participant Router as Router
    participant DB as Database giao dịch

    Note over New,DB: Deployment mới đã ghi một số TRANSACTION theo định dạng dữ liệu mới (vd thêm trường mới, đổi cấu trúc) trước khi lỗi được phát hiện

    Dev->>CI: Phát hiện lỗi, ra lệnh rollback
    CI->>Old: Deploy lại bản cũ, thay thế bản mới
    CI->>Router: Trỏ 100% traffic về bản cũ

    Note over Old,DB: Bản cũ không biết đọc/ghi định dạng dữ liệu mới, có thể đọc sai hoặc lỗi khi gặp các TRANSACTION đã được bản mới ghi theo định dạng mới
    Old-->>Dev: Một số giao dịch cũ hiển thị sai hoặc lỗi do không tương thích định dạng dữ liệu, không có kế hoạch xử lý từ trước
```
