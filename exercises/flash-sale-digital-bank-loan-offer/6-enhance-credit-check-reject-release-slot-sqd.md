# Sequence - Enhance: Từ chối tín dụng, nhả suất atomic cho người kế tiếp

Đây là flow **enhance hoàn toàn mới**, minh hoạ đúng kịch bản nêu trong đề bài: khách #999 và #1000 giữ được vị trí hợp lệ, nhưng khách #999 bị từ chối tín dụng sau 5 giây — suất phải được chuyển đúng cho khách #1001 (người kế tiếp trong hàng đợi, không phải #1000 vì #1000 đã có vị trí riêng của mình và đang tự chờ kết quả credit check của chính mình), toàn bộ việc "nhả suất - cấp suất kế tiếp" atomic trong 1 transaction để không có 2 khách cùng được cấp 1 suất. Đáp ứng **yêu cầu 2** của đề bài.

```mermaid
sequenceDiagram
    participant C999 as Khách #999
    participant C1001 as Khách #1001 (hàng đợi)
    participant BE as Backend API
    participant Credit as Credit Bureau (bên ngoài)
    participant DB as Database

    Note over DB: Khách #999 và #1000 đã giữ vị trí hợp lệ, #1001 đang xếp hàng chờ vì chương trình đã đủ 1000 người vào vòng kiểm tra tín dụng
    BE->>Credit: Kiểm tra tín dụng khách #999
    Note over Credit: Chờ 5 giây
    Credit-->>BE: Từ chối, không đạt điều kiện
    BE->>DB: Transaction atomic bắt đầu
    BE->>DB: Cập nhật REGISTRATION #999 status=rejected, không giữ SLOT_ALLOCATION
    BE->>DB: Tìm registration có queue_position nhỏ nhất đang ở status=queued mà chưa vào vòng kiểm tra (đó là #1001)
    BE->>DB: Chuyển REGISTRATION #1001 sang status=checking_credit
    DB-->>BE: Transaction commit thành công
    Note over DB: Suất bị nhả ra và người kế tiếp được đưa vào vòng kiểm tra trong cùng 1 transaction, không có khoảng hở để suất bị cấp trùng
    BE->>Credit: Kiểm tra tín dụng khách #1001
    Credit-->>BE: Đạt điều kiện
    BE->>DB: Gán SLOT_ALLOCATION cho registration #1001, status=slot_granted
    BE-->>C1001: Xác nhận đã được cấp suất vay ưu đãi
    BE-->>C999: Rất tiếc, không đủ điều kiện tín dụng
```
