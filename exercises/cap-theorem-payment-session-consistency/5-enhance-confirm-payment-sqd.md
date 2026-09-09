# Enhance sequence — Confirm payment (CP, lock quorum, từ chối rõ ràng khi thiếu quorum)

Đây là **enhance**, cùng flow "Confirm payment" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1, 2 và 4 của đề bài: chuyển sang CP với `ORDER_LOCK` cần quorum, chỉ 1 trong 2 request trùng được xác nhận, và từ chối rõ ràng thay vì âm thầm trả kết quả sai khi thiếu quorum.

```mermaid
sequenceDiagram
    actor UserRequest1 as Request xác nhận thanh toán (lần 1)
    actor UserRequest2 as Request xác nhận thanh toán (lần 2, trùng)
    participant Policy as CONSISTENCY_POLICY store
    participant Lock as ORDER_LOCK store (quorum-backed)
    participant Gateway as Payment Gateway
    participant RejectLog as PARTITION_REJECTION_LOG store

    UserRequest1->>Policy: Kiểm tra endpoint confirm-payment → mode=CP
    UserRequest2->>Policy: Kiểm tra endpoint confirm-payment → mode=CP

    par 2 request gần như đồng thời
        UserRequest1->>Lock: Thử giữ ORDER_LOCK(session_id) — cần đạt quorum
        UserRequest2->>Lock: Thử giữ ORDER_LOCK(session_id) — cần đạt quorum
    end

    alt Đủ quorum để xử lý CP
        Lock-->>UserRequest1: Giữ được lock (request đầu tiên)
        Lock-->>UserRequest2: Không giữ được lock — bị từ chối ngay
        UserRequest1->>Gateway: Charge tiền
        Gateway-->>UserRequest1: Thành công
        UserRequest1->>Lock: Cập nhật status=confirmed, release lock
        UserRequest2-->>UserRequest2: Nhận lỗi "Đơn đang được xử lý bởi request khác" — không charge lần 2
    else Không đủ quorum (network partition)
        Lock-->>UserRequest1: Không đủ quorum
        Lock->>RejectLog: Ghi PARTITION_REJECTION_LOG(endpoint=confirm-payment, reason=insufficient_quorum)
        UserRequest1-->>UserRequest1: Nhận lỗi rõ ràng "Tạm không xử lý được, vui lòng thử lại" — không charge, không trả kết quả không chắc chắn
    end
```
