# Sequence Diagram — Base: Register Device

Đây là **base**, flow thiết bị POS đăng ký vào hệ thống khi lắp đặt tại chi nhánh — tiền đề bắt buộc để sau này bổ sung khai báo lịch hoạt động của chi nhánh ngay tại bước đăng ký này.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên lắp đặt
    participant Device as Thiết bị POS
    participant Platform as Nền tảng quản lý POS

    Staff->>Device: Kích hoạt thiết bị tại chi nhánh
    Device->>Platform: Register (store_id, device info)
    Platform->>Platform: Create DEVICE record (status=active)
    Platform-->>Device: Registration confirmed
```
