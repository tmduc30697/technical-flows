---
name: draw-er-diagram
description: Dùng khi cần vẽ ER diagram (entity-relationship diagram) để minh họa data model / quan hệ giữa các entity, bảng, hoặc object trong hệ thống. Kích hoạt khi user nói "vẽ ER diagram", "vẽ sơ đồ quan hệ thực thể", "data model trông như thế nào", hoặc khi cần làm rõ quan hệ 1-1/1-n/n-n giữa các entity.
---

# ER Diagram (Mermaid)

Vẽ bằng cú pháp `erDiagram` của Mermaid, đặt trong code block ```mermaid.

## Cú pháp cơ bản

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : has
    USER {
        uuid id PK
        string email UK
        string name
    }
    ACCOUNT {
        uuid id PK
        uuid user_id FK
        string provider
        string provider_account_id
    }
```

## Ký hiệu quan hệ (cardinality)

- `||--||` : one-to-one (bắt buộc cả hai bên)
- `||--o{` : one-to-many (một bên bắt buộc, bên kia optional/nhiều)
- `}o--o{` : many-to-many
- `|o--||` : zero-or-one to one

Đọc thứ tự trái→phải: ký hiệu sát entity nào mô tả cardinality (min/max) ứng với phía entity đó.

## Quy tắc

- Luôn đánh dấu PK (primary key), FK (foreign key), UK (unique key) trong khối attribute nếu biết.
- Tên entity viết HOA, không dùng khoảng trắng (dùng snake_case hoặc CamelCase).
- Nhãn quan hệ (`: has`, `: belongs to`) ngắn gọn, dùng động từ.
- Nếu chưa chắc cardinality, hỏi lại thay vì đoán — vẽ sai cardinality dễ gây hiểu nhầm về ràng buộc dữ liệu (ví dụ: 1-1 thay vì 1-n giữa User và Account khi hệ thống cho phép link nhiều provider).

## Hiển thị cho user xem

Code block ```mermaid``` không tự render thành hình trong môi trường chat này — user chỉ thấy text thô, phải copy đi nơi khác mới xem được. Để user xem được sơ đồ thật ngay tại chỗ, publish bằng tool Artifact:

- Nhúng đúng đoạn `erDiagram` đã vẽ vào `<pre class="mermaid">...</pre>` trong một file HTML, publish bằng tool Artifact.
- Bắt buộc load skill `artifact-design` trước khi publish (yêu cầu cứng của tool Artifact) — nhưng vì đây chỉ là một sơ đồ kỹ thuật để xem nhanh/kiểm tra, chọn treatment tối giản, utilitarian: tiêu đề ngắn + khung chứa diagram (đảm bảo `overflow-x: auto` vì ER diagram dễ tràn ngang), không cần hero, minh họa, hay hệ thống type phức tạp — tránh tốn công thiết kế không cần thiết cho một tác vụ vốn chỉ cần "xem cho đúng".
- Vẫn phải tôn trọng phần cứng bắt buộc của Artifact: theme sáng/tối đều đọc được, `body` có `background` tường minh.
- Khi user chỉ nói "test"/"thử" mà không cần bản đẹp để chia sẻ, ưu tiên tốc độ và tối giản hơn là đầu tư thẩm mỹ.
- KHÔNG tự `list`/dò title để tìm artifact cũ — việc này do bên gọi skill (ví dụ agent `interviewer`) quản lý, không phải trách nhiệm của skill này. Skill chỉ quan tâm 2 việc:
  - Nếu prompt gọi skill có kèm sẵn **1 artifact URL đã có từ trước** (do caller truyền vào để yêu cầu cập nhật) → publish lại đúng file với `url` đó để ghi đè lên artifact đó, không tạo mới.
  - Nếu prompt không kèm URL nào → publish mới bình thường (không truyền `url`). Sau khi publish xong, **luôn nói rõ URL vừa tạo trong câu trả lời** để caller (agent gọi skill) có thể tự lưu lại và tái sử dụng cho lần sau nếu cần — không được im lặng bỏ qua bước báo URL này.
- Đặt title mô tả đúng nội dung diagram lần này (không bắt buộc cố định "ER Diagram Preview" nữa vì việc chọn artifact nào để update giờ do caller quyết định qua URL, không còn dựa vào title).
