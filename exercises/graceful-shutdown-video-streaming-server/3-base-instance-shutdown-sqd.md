# Sequence - Base - Flow "instance-shutdown"

Đây là **base**: cách một instance xử lý tín hiệu shutdown theo kiểu request ngắn thông thường (fixed grace period ngắn, hết giờ là cắt hết). Flow này được chọn vì chính nó là điểm bất hợp lý mà đề bài yêu cầu sửa - áp dụng lên các kết nối streaming dài sẽ cắt ngang video đang phát và khiến player báo lỗi im lặng.

```mermaid
sequenceDiagram
    actor Client
    participant Instance
    participant Orchestrator as Autoscaler / Deploy tool

    Orchestrator->>Instance: SIGTERM (yêu cầu shutdown)
    Instance-->>Instance: ngừng nhận connection mới
    Note over Instance: chờ grace period cố định, ví dụ 30 giây,\nkhông xét tới thời lượng video còn lại

    par Các stream session vẫn đang mở
        Instance-->>Client: tiếp tục gửi chunk trong lúc chờ
    end

    Note over Instance: hết 30 giây, bất kể session còn dài bao lâu

    Instance-->>Client: đóng đột ngột toàn bộ kết nối còn mở (TCP reset)
    Client-->>Client: player không nhận được tín hiệu rõ ràng,\nhiển thị lỗi phát chung chung
    Orchestrator->>Instance: force kill process
```
