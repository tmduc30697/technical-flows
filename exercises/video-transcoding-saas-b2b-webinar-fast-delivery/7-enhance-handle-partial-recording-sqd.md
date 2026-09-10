# Sequence Diagram — Enhance: Handle Partial Recording

Đây là **enhance**, flow mới xử lý trường hợp cuộc họp bị dừng đột ngột hoặc file ghi hình lỗi một phần — xử lý phần dữ liệu còn dùng được để gửi bản ghi một phần kèm thông báo rõ ràng, thay vì để khách hàng chờ vô thời hạn.

```mermaid
sequenceDiagram
    participant MeetingSvc as Meeting Service
    participant Worker as Dedicated Fast Worker
    participant Storage as Recording Storage
    actor CustomerOrg as Khách hàng (Organization)

    MeetingSvc->>MeetingSvc: Meeting bị dừng đột ngột / recording lỗi một phần
    MeetingSvc->>Storage: Store RECORDING, is_partial = true

    Worker->>Storage: Fetch RECORDING
    Worker->>Worker: Xác định phần dữ liệu còn dùng được, bỏ phần hỏng

    alt có phần dữ liệu dùng được
        Worker->>Storage: Transcode phần hợp lệ, tạo DELIVERY is_partial_delivery = true
        Worker-->>MeetingSvc: Partial recording ready
        MeetingSvc-->>CustomerOrg: Gửi bản ghi một phần kèm thông báo rõ ràng về phần bị thiếu
    else không còn phần nào dùng được
        Worker-->>MeetingSvc: Không thể tạo bản ghi
        MeetingSvc-->>CustomerOrg: Thông báo recording thất bại, không có bản ghi khả dụng
    end
```
