---
name: draw-sequence-diagram
description: Dùng khi cần vẽ sequence diagram để minh họa luồng tương tác giữa nhiều actor/service theo thời gian (request/response, message passing). Kích hoạt khi user nói "vẽ sequence diagram", "luồng request thế nào", "vẽ flow tương tác giữa A và B", hoặc khi giải thích một flow API/OAuth/network nhiều bước.
---

# Sequence Diagram (Mermaid)

Vẽ bằng cú pháp `sequenceDiagram` của Mermaid, đặt trong code block ```mermaid.

## Cú pháp cơ bản

```mermaid
sequenceDiagram
    actor U as User
    participant B as Browser
    participant S as Backend
    participant G as Google OAuth

    U->>B: Click "Sign in with Google"
    B->>S: GET /auth/google
    S-->>B: 302 redirect (state, redirect_uri)
    B->>G: Authorization request
    G-->>U: Show consent screen
    U->>G: Allow
    G-->>B: 302 redirect callback (code, state)
    B->>S: GET /auth/google/callback?code&state
    S->>G: POST /token (exchange code)
    G-->>S: access_token, id_token
    S-->>B: Set session cookie
```

## Rẽ nhánh trong sequence

```mermaid
sequenceDiagram
    participant B as Browser
    participant G as Google
    participant S as Backend

    B->>G: Authorization request
    alt User Allow
        G-->>B: redirect with code
        B->>S: exchange code
    else User Deny
        G-->>B: redirect with error=access_denied
        B->>S: show error, no session created
    end
```

## Quy tắc

- `->>` : gọi đồng bộ (solid arrow); `-->>` : phản hồi (dashed arrow); `-x` : gọi lỗi/mất kết nối.
- `actor` cho con người, `participant` cho hệ thống/service.
- Dùng `alt/else/end`, `opt/end`, `loop/end` để biểu diễn rẽ nhánh/lặp (ví dụ nhánh Deny vs Allow).
- Dùng `Note over X,Y: ...` để chú thích trạng thái quan trọng (ví dụ: "state param lưu trong session để validate khi callback").
- Đặt alias (`as`) cho participant có tên dài để diagram không bị tràn ngang.
- Khác với flowchart: sequence diagram thể hiện **thứ tự message qua lại giữa nhiều actor theo thời gian**, không phải logic rẽ nhánh nội bộ của một actor — nếu chỉ cần biểu diễn if/else của một xử lý đơn lẻ, dùng skill flowchart thay vì cái này.

## Hiển thị cho user xem

Code block ```mermaid``` không tự render thành hình trong môi trường chat này — user chỉ thấy text thô, phải copy đi nơi khác mới xem được. Để user xem được sơ đồ thật ngay tại chỗ, publish bằng tool Artifact:

- Nhúng đúng đoạn `sequenceDiagram` đã vẽ vào `<pre class="mermaid">...</pre>` trong một file HTML, publish bằng tool Artifact.
- Bắt buộc load skill `artifact-design` trước khi publish (yêu cầu cứng của tool Artifact) — nhưng vì đây chỉ là một sơ đồ kỹ thuật để xem nhanh/kiểm tra, chọn treatment tối giản, utilitarian: tiêu đề ngắn + khung chứa diagram (đảm bảo `overflow-x: auto` vì sequence diagram nhiều participant dễ tràn ngang), không cần hero, minh họa, hay hệ thống type phức tạp — tránh tốn công thiết kế không cần thiết cho một tác vụ vốn chỉ cần "xem cho đúng".
- Vẫn phải tôn trọng phần cứng bắt buộc của Artifact: theme sáng/tối đều đọc được, `body` có `background` tường minh.
- Khi user chỉ nói "test"/"thử" mà không cần bản đẹp để chia sẻ, ưu tiên tốc độ và tối giản hơn là đầu tư thẩm mỹ.
- KHÔNG tự `list`/dò title để tìm artifact cũ — việc này do bên gọi skill (ví dụ agent `interviewer`) quản lý, không phải trách nhiệm của skill này. Skill chỉ quan tâm 2 việc:
  - Nếu prompt gọi skill có kèm sẵn **1 artifact URL đã có từ trước** (do caller truyền vào để yêu cầu cập nhật) → publish lại đúng file với `url` đó để ghi đè lên artifact đó, không tạo mới.
  - Nếu prompt không kèm URL nào → publish mới bình thường (không truyền `url`). Sau khi publish xong, **luôn nói rõ URL vừa tạo trong câu trả lời** để caller (agent gọi skill) có thể tự lưu lại và tái sử dụng cho lần sau nếu cần — không được im lặng bỏ qua bước báo URL này.
- Đặt title mô tả đúng nội dung diagram lần này (không bắt buộc cố định "Sequence Diagram Preview" nữa vì việc chọn artifact nào để update giờ do caller quyết định qua URL, không còn dựa vào title).
