# Sequence - Enhance: Credit bureau timeout, retry giới hạn rồi nhả suất

Đây là flow **enhance hoàn toàn mới**, mô tả trường hợp credit bureau (hệ thống bên ngoài) không phản hồi đúng hạn. Quy định: tối đa 2 lần retry, mỗi lần chờ tối đa 5 giây (tổng thời gian chờ tối đa cho 1 khách là 15 giây), quá số lần này coi là thất bại và nhả suất cho người kế tiếp ngay, không để khách bị timeout chặn vô thời hạn cả hàng đợi phía sau. Đáp ứng **yêu cầu 4** của đề bài.

```mermaid
sequenceDiagram
    participant BE as Backend API
    participant Credit as Credit Bureau (bên ngoài)
    participant DB as Database
    participant C1002 as Khách #1002 (hàng đợi)

    BE->>DB: Tạo CREDIT_CHECK(registration=#1001, attempt_number=1)
    BE->>Credit: Gọi kiểm tra tín dụng (timeout=5s)
    Credit--xBE: Không phản hồi trong 5 giây
    BE->>DB: Cập nhật CREDIT_CHECK attempt_number=1, timeout_flag=true

    BE->>DB: Tạo CREDIT_CHECK(registration=#1001, attempt_number=2)
    BE->>Credit: Retry lần 2 (timeout=5s)
    Credit--xBE: Không phản hồi trong 5 giây
    BE->>DB: Cập nhật CREDIT_CHECK attempt_number=2, timeout_flag=true

    Note over BE: Đã đạt credit_check_max_retries=2, tổng thời gian chờ 10 giây, coi là thất bại
    BE->>DB: Transaction atomic, cập nhật REGISTRATION #1001 status=expired, không giữ SLOT_ALLOCATION
    BE->>DB: Chuyển registration #1002 (kế tiếp) sang status=checking_credit
    BE->>Credit: Kiểm tra tín dụng khách #1002
    Credit-->>BE: Đạt điều kiện
    BE->>DB: Gán SLOT_ALLOCATION cho registration #1002
    BE-->>C1002: Xác nhận đã được cấp suất vay ưu đãi
    Note over BE: Khách #1001 không chặn được hàng đợi quá 10 giây dù credit bureau timeout liên tục
```
