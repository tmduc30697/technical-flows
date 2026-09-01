# Sequence — Logout

**Lớp: black-box** — **Nhóm: Hiện trạng**. Diagram này thể hiện action "Logout" đã bổ sung vào Use case baseline (`1-usecase-baseline-auth.md`) — action đối xứng bắt buộc với Login để kết thúc session, hoàn thiện vòng đời auth trước khi so sánh với nhóm Thay đổi.

```mermaid
sequenceDiagram
    actor U as Employee
    participant FE as Dashboard Frontend
    participant AUTH as Auth Service

    U->>FE: Bấm "Logout"
    FE->>AUTH: POST /logout (kèm session token/cookie hiện tại)
    AUTH->>AUTH: Thu hồi session (xoá cookie phía server / thêm token vào blocklist ngắn hạn)
    AUTH-->>FE: 200 OK
    FE->>FE: Xoá token phía client (localStorage/cookie)
    FE-->>U: Redirect về màn hình Login
```

**Ghi chú:** Vì baseline dùng session token dạng JWT/cookie tự phát (không lưu bảng session riêng), logout không cần entity mới trong ERD — chỉ cần vô hiệu hoá token hiện tại (hoặc đơn giản là để cookie hết hạn/bị xoá phía client nếu chấp nhận JWT còn hạn vẫn valid tới khi tự expire).
