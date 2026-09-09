# Enhance sequence — Scale cluster rebalance (bảng "đang di chuyển", giữ nguyên TTL)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base không có cơ chế rebalance an toàn khi thêm/bớt node. Đáp ứng yêu cầu 1 (chỉ session thuộc dải hash bị dịch chuyển mới di chuyển), yêu cầu 2 (router biết hỏi node nào trong lúc chuyển tiếp, tránh session-not-found tạm thời) và yêu cầu 4 (giữ nguyên TTL gốc khi di chuyển) của đề bài.

```mermaid
sequenceDiagram
    participant Ring as Hash Ring
    participant Router as Session Router
    participant NodeOld as NODE cũ
    participant NodeNew as NODE mới thêm vào
    participant Migration as SESSION_MIGRATION

    Ring->>Ring: Thêm NODE mới, chỉ các virtual node trong 1 dải hash nhỏ được gán lại
    Ring->>Migration: Session S rơi vào dải bị dịch chuyển, tạo SESSION_MIGRATION(from=NodeOld, to=NodeNew, moving=true)

    Migration->>NodeOld: Đọc SESSION_REPLICA(S) hiện tại, gồm expires_at gốc
    NodeOld-->>Migration: data_json, expires_at=T (không đổi)
    Migration->>NodeNew: Ghi SESSION_REPLICA(S) với đúng expires_at=T (giữ nguyên TTL còn lại, không reset đầy)

    actor User
    User->>Router: Request kèm session token của S, ngay trong lúc đang moving=true
    Router->>Migration: Kiểm tra trạng thái migration của S
    Migration-->>Router: moving=true, gợi ý double-read cả NodeOld và NodeNew
    par Double-read trong giai đoạn chuyển tiếp
        Router->>NodeOld: Thử đọc session S
        Router->>NodeNew: Thử đọc session S
    end
    NodeOld-->>Router: Vẫn còn dữ liệu (chưa xóa)
    Router-->>User: Phục vụ request bình thường, không trả session-not-found

    Migration->>Migration: Xác nhận NodeNew đã có dữ liệu đầy đủ, đặt moving=false, status=completed
    Migration->>NodeOld: Xóa SESSION_REPLICA(S) khỏi node cũ
    Note over Router: Sau khi completed, router chỉ còn hỏi NodeNew, các session không thuộc dải bị dịch chuyển hoàn toàn không bị ảnh hưởng
```
