# Enhance sequence — Password reset (mức xác minh nâng cao)

Đây là **enhance**, cùng flow "Password reset" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 4 của đề bài: không còn chỉ dựa vào quyền sở hữu email, mà thêm bước `IDENTITY_VERIFICATION` (vd giấy tờ tùy thân + trả lời nhiều câu hỏi bảo mật độc lập) trước khi cho đổi mật khẩu — chấp nhận chậm hơn để giảm rủi ro lộ hồ sơ bệnh án.

```mermaid
sequenceDiagram
    actor Patient
    participant App as Healthcare Platform
    participant DB as PATIENT / IDENTITY_VERIFICATION store
    participant Email as Email Gateway

    Patient->>App: Chọn "Quên mật khẩu", nhập email
    App->>DB: Kiểm tra email có tồn tại
    DB-->>App: Khớp
    App->>Email: Gửi link xác nhận bước 1 (chỉ để tiếp tục, chưa cho đổi mật khẩu)
    Email-->>Patient: Nhận link
    Patient->>App: Mở link, tiếp tục bước xác minh danh tính nâng cao
    App->>Patient: Yêu cầu upload giấy tờ tùy thân + trả lời nhiều câu hỏi bảo mật độc lập
    Patient->>App: Nộp bằng chứng
    App->>DB: Tạo IDENTITY_VERIFICATION (method=id_document+multi_factor_kba)
    App->>App: Đối chiếu bằng chứng, tính status
    alt Xác minh hợp lệ
        DB-->>App: status=verified
        App-->>Patient: Cho phép đặt mật khẩu mới
        Patient->>App: Nhập mật khẩu mới
        App->>DB: Cập nhật password_hash
        App-->>Patient: Đổi mật khẩu thành công
    else Xác minh không đạt
        DB-->>App: status=rejected
        App-->>Patient: Từ chối, hướng dẫn liên hệ hỗ trợ trực tiếp
    end
```
