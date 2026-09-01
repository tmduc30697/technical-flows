# Activity — Google OAuth Login Flow

**Lớp: black-box** — **Nhóm: Thay đổi**. Diagram này thể hiện toàn bộ quy trình nghiệp vụ của "Login with Google" dưới dạng các bước rẽ nhánh, gộp cả 2 use case (Login thành công và Deny consent) cùng các điều kiện phụ (CSRF state, redirect_uri whitelist, domain restriction, account linking) vào 1 bức tranh duy nhất. Tín hiệu chọn Activity thay vì Flowchart: quy trình có nhiều rẽ nhánh thật và có 3 actor tham gia qua lại (Employee, App, Google) — đúng bảng tín hiệu ở Bước 1 ("quy trình nghiệp vụ nhiều bước, nhiều actor duyệt qua lại").

```mermaid
flowchart TD
    subgraph Employee
        A1(["Bấm nút 'Sign in with Google'"])
        A5{Allow hay Deny?}
        A9([Thấy kết quả])
    end

    subgraph App["Dashboard / Auth Service"]
        B1[Build authorize URL + state]
        B2[Nhận callback]
        B3{state khớp?}
        B4{redirect_uri hợp lệ?}
        B5{error=access_denied?}
        B6[Đổi code lấy access_token]
        B7[Gọi Google lấy profile]
        B8{Domain thuộc công ty?}
        B9{Email đã tồn tại?}
        B10[Link vào user cũ]
        B11[Tạo user mới]
        B12[Issue session token riêng]
        B13[Reject request]
    end

    subgraph Google["Google (OAuth Provider)"]
        C1([Hiện consent screen])
        C2([Redirect kèm code hoặc error])
    end

    A1 --> B1 --> C1
    C1 --> A5
    A5 -->|Allow| C2
    A5 -->|Deny| C2
    C2 --> B2 --> B3
    B3 -->|Không khớp| B13
    B3 -->|Khớp| B5
    B5 -->|Có, error=access_denied| A9
    B5 -->|Không, có code| B4
    B4 -->|Sai whitelist| B13
    B4 -->|Hợp lệ| B6 --> B7 --> B8
    B8 -->|Domain khác| B13
    B8 -->|Domain đúng| B9
    B9 -->|Đã có| B10 --> B12
    B9 -->|Chưa có| B11 --> B12
    B12 --> A9
    B13 --> A9
```

**Ghi chú:** `access_token` của Google (bước B6) chỉ dùng để gọi B7 rồi bỏ, không lưu vào DB (`Session token` ở B12 là token do hệ thống tự phát, tách biệt hoàn toàn) — đúng yêu cầu đề bài.
