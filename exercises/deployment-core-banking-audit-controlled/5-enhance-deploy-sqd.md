# Enhance sequence — Deploy (canary theo rủi ro giao dịch, four-eyes approval, audit log mỗi bước)

Đây là **enhance** của flow `deploy` đã có ở base. So với base (chuyển thẳng 100% traffic cho mọi loại giao dịch, không phê duyệt, không ghi log), nay canary bắt đầu chỉ phục vụ giao dịch rủi ro thấp (`balance_inquiry`), mỗi lần mở rộng loại giao dịch hoặc tăng tỷ lệ traffic đều cần tối thiểu 2 người phê duyệt (four-eyes), và mọi bước đều ghi vào `AUDIT_LOG` bất biến. Đáp ứng yêu cầu 1, 2 và 5 của đề bài.

```mermaid
sequenceDiagram
    actor Dev as Kỹ sư triển khai
    participant CI as CI/CD
    participant Canary as Deployment canary
    participant Stable as Deployment stable
    participant Stage as CANARY_STAGE store
    actor Approver1 as Người phê duyệt 1
    actor Approver2 as Người phê duyệt 2
    participant Audit as AUDIT_LOG (append-only)
    participant Router as Router
    actor Customer

    Dev->>CI: Kích hoạt deploy canary
    CI->>Canary: Deploy bản mới thành canary, song song với Stable
    CI->>Stage: Tạo CANARY_STAGE 1 (traffic_percent=1%, allowed_transaction_types=[balance_inquiry])
    CI->>Audit: Ghi AUDIT_LOG(action=stage_started, performed_by=Dev, new_traffic_percent=1%, scope=balance_inquiry)

    Customer->>Router: Gửi request xem số dư hoặc chuyển tiền
    alt Loại giao dịch nằm trong allowed_transaction_types của stage hiện tại
        Router->>Canary: Route request sang canary (chỉ balance_inquiry ở stage 1)
    else Loại giao dịch chưa được phép ở stage hiện tại (vd transfer)
        Router->>Stable: Route request sang stable, dù rơi vào nhóm traffic canary
    end

    Note over Stage,Audit: Sau khi stage 1 ổn định qua thời gian giám sát quy định

    Dev->>Approver1: Yêu cầu phê duyệt mở rộng sang stage 2 (thêm transfer, tăng traffic_percent=10%)
    Approver1->>Stage: Ghi APPROVAL(decision=approved)
    Dev->>Approver2: Yêu cầu phê duyệt thứ 2 (bắt buộc người khác với Approver1)
    Approver2->>Stage: Ghi APPROVAL(decision=approved)

    alt Đã đủ 2 approval hợp lệ
        Stage->>Stage: Chuyển CANARY_STAGE sang stage 2 (allowed_transaction_types=[balance_inquiry,transfer])
        Stage->>Audit: Ghi AUDIT_LOG(action=traffic_increased, performed_by=[Approver1,Approver2], previous=1%, new=10%, scope=balance_inquiry+transfer)
    else Chưa đủ 2 approval
        Stage->>Stage: Giữ nguyên stage hiện tại, không tự động mở rộng
    end

    Note over Stage,Audit: Lặp lại quy trình phê duyệt tương tự cho từng bậc tiếp theo, và bắt buộc tối thiểu 2 người xác nhận riêng trước khi mở lên 100% traffic
```
