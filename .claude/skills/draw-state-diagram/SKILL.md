---
name: draw-state-diagram
description: Dùng khi cần vẽ state diagram để minh họa các trạng thái và transition hợp lệ của một entity (ví dụ: trạng thái order, session, OAuth flow, account). Kích hoạt khi user nói "vẽ state diagram", "sơ đồ trạng thái", "các trạng thái có thể có của X", hoặc khi cần làm rõ transition nào hợp lệ/không hợp lệ.
---

# State Diagram (Mermaid)

Vẽ bằng cú pháp `stateDiagram-v2` của Mermaid, đặt trong code block ```mermaid.

## Cú pháp cơ bản

```mermaid
stateDiagram-v2
    [*] --> Initiated: user click "Sign in with Google"
    Initiated --> PendingConsent: redirected to Google
    PendingConsent --> Allowed: user click Allow
    PendingConsent --> Denied: user click Deny
    Allowed --> LinkedAccount: email matches existing account
    Allowed --> NewAccount: no matching account
    Denied --> [*]: show error, no session
    LinkedAccount --> [*]: session created
    NewAccount --> [*]: session created
```

## Quy tắc

- `[*]` là start/end state — dùng ở đầu (start) và cuối (end) của diagram, ý nghĩa khác nhau theo hướng mũi tên.
- Nhãn trên transition (`: user click Allow`) mô tả **sự kiện/điều kiện** gây ra transition, không phải mô tả trạng thái đích.
- Nếu một entity có state lồng nhau (composite state), dùng `state X { ... }`.
- Chỉ vẽ transition có thật trong logic — không vẽ transition "tưởng tượng" để lấp khoảng trống; nếu chưa rõ transition nào hợp lệ, đó chính là điểm cần hỏi lại thay vì đoán.
- Dùng diagram này để bắt case thiếu: một state không có đường thoát (dead end) hoặc không có đường vào thường là dấu hiệu bug/thiếu xử lý trong thiết kế.

## Hiển thị cho user xem

Code block ```mermaid``` không tự render thành hình trong môi trường chat này — user chỉ thấy text thô, phải copy đi nơi khác mới xem được. Để user xem được sơ đồ thật ngay tại chỗ, publish bằng tool Artifact:

- Nhúng đúng đoạn `stateDiagram-v2` đã vẽ vào `<pre class="mermaid">...</pre>` trong một file HTML, publish bằng tool Artifact.
- Bắt buộc load skill `artifact-design` trước khi publish (yêu cầu cứng của tool Artifact) — nhưng vì đây chỉ là một sơ đồ kỹ thuật để xem nhanh/kiểm tra, chọn treatment tối giản, utilitarian: tiêu đề ngắn + khung chứa diagram (đảm bảo `overflow-x: auto` vì state diagram nhiều transition dễ tràn ngang), không cần hero, minh họa, hay hệ thống type phức tạp — tránh tốn công thiết kế không cần thiết cho một tác vụ vốn chỉ cần "xem cho đúng".
- Vẫn phải tôn trọng phần cứng bắt buộc của Artifact: theme sáng/tối đều đọc được, `body` có `background` tường minh.
- Khi user chỉ nói "test"/"thử" mà không cần bản đẹp để chia sẻ, ưu tiên tốc độ và tối giản hơn là đầu tư thẩm mỹ.
- KHÔNG tự `list`/dò title để tìm artifact cũ — việc này do bên gọi skill (ví dụ agent `interviewer`) quản lý, không phải trách nhiệm của skill này. Skill chỉ quan tâm 2 việc:
  - Nếu prompt gọi skill có kèm sẵn **1 artifact URL đã có từ trước** (do caller truyền vào để yêu cầu cập nhật) → publish lại đúng file với `url` đó để ghi đè lên artifact đó, không tạo mới.
  - Nếu prompt không kèm URL nào → publish mới bình thường (không truyền `url`). Sau khi publish xong, **luôn nói rõ URL vừa tạo trong câu trả lời** để caller (agent gọi skill) có thể tự lưu lại và tái sử dụng cho lần sau nếu cần — không được im lặng bỏ qua bước báo URL này.
- Đặt title mô tả đúng nội dung diagram lần này (không bắt buộc cố định "State Diagram Preview" nữa vì việc chọn artifact nào để update giờ do caller quyết định qua URL, không còn dựa vào title).
