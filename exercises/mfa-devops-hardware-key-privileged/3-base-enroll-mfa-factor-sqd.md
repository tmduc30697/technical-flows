# Base sequence — Enroll MFA factor (chỉ cần 1 factor duy nhất, không phân biệt loại)

Đây là **base**, flow enroll MFA hiện tại: kỹ sư chỉ cần đăng ký 1 phương thức bất kỳ là được cấp quyền production ngay, không yêu cầu số lượng tối thiểu hay loại phương thức cụ thể. Đây là tiền đề cho yêu cầu 2 của đề bài (chưa có ràng buộc tối thiểu 2 key) và gián tiếp liên quan yêu cầu 5 (chưa có giám sát tần suất enroll).

```mermaid
sequenceDiagram
    actor SRE as Kỹ sư SRE
    participant Server
    participant DB as Database

    SRE->>Server: Enroll MFA factor (chọn SMS OTP, nhập số điện thoại)
    Server->>DB: INSERT MFA_FACTOR (user=SRE, type=sms_otp, phone_number=...)
    DB-->>Server: OK

    Server->>DB: UPDATE USER SET production_access=granted WHERE id=SRE
    DB-->>Server: OK
    Server-->>SRE: Enroll thành công, được cấp quyền production ngay với đúng 1 factor

    Note over Server,DB: Không kiểm tra số lượng factor tối thiểu, không phân biệt factor có phải hardware key hay không, cũng không theo dõi tần suất enroll factor mới
```
