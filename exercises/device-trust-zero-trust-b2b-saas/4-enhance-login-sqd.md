# Enhance sequence — Login yêu cầu verify device certificate trước khi cấp session

Đây là **enhance** của flow `login` đã có ở base. So với base (chỉ kiểm tra mật khẩu), enhance bắt buộc request phải kèm device certificate, server verify certificate hợp lệ trước khi cấp session, và có nhánh từ chối rõ ràng cho thiết bị BYOD không có certificate. Đáp ứng yêu cầu 1 (verify certificate còn hạn, chưa revoke, đúng chuỗi tin cậy) và yêu cầu 2 (từ chối rõ ràng, hướng dẫn liên hệ IT đăng ký thiết bị thay vì lỗi mơ hồ).

```mermaid
sequenceDiagram
    actor User
    participant CompanyDevice as Thiết bị công ty quản lý
    participant BYOD as Thiết bị cá nhân (BYOD)
    participant App as Internal Tool
    participant MDM as MDM Service
    participant DB as Database

    rect rgb(235, 245, 235)
    Note over User,DB: Trường hợp 1 - thiết bị công ty quản lý, có certificate hợp lệ
    User->>CompanyDevice: Nhập email + mật khẩu
    CompanyDevice->>App: POST /login (email, password, device_certificate)
    App->>DB: Kiểm tra password_hash khớp
    DB-->>App: Hợp lệ

    App->>MDM: Verify certificate (chain hợp lệ, chưa revoke, còn hạn)
    MDM-->>App: Certificate hợp lệ

    App->>DB: SELECT POSTURE_POLICY theo department của user
    DB-->>App: min_os_version, require_disk_encryption...
    App->>App: So khớp posture hiện tại của DEVICE với policy, đạt chuẩn

    App->>DB: INSERT SESSION (user_id, device_id, status=active)
    App-->>CompanyDevice: Trả session token, truy cập thành công
    end

    rect rgb(250, 235, 235)
    Note over User,DB: Trường hợp 2 - thiết bị cá nhân, không có certificate hợp lệ
    User->>BYOD: Nhập email + mật khẩu
    BYOD->>App: POST /login (email, password, device_certificate=null)
    App->>DB: Kiểm tra password_hash khớp
    DB-->>App: Hợp lệ

    App->>MDM: Verify certificate
    MDM-->>App: Không tìm thấy certificate hợp lệ cho thiết bị này

    App-->>BYOD: Từ chối truy cập, "Thiết bị chưa được công ty quản lý, vui lòng liên hệ IT để đăng ký thiết bị (MDM enrollment)"
    Note over App,BYOD: Không cấp session, thông báo rõ nguyên nhân và bước tiếp theo, không phải lỗi chung chung
    end
```
