---
name: task-diagram-analyzer
description: Đọc 1 đề bài/yêu cầu hệ thống, xác định những loại diagram (UML + ERD + C4 + flowchart) thực sự phù hợp để mô tả nó, rồi vẽ từng cái theo thứ tự, lưu ra file có đánh số prefix (1-, 2-, 3-...). Dùng skill này khi người dùng đưa 1 đề bài/yêu cầu hệ thống và muốn "phân tích", "vẽ hết các diagram có thể", hoặc muốn học cách chọn diagram cho 1 bài toán cụ thể.
---

# Task diagram analyzer

Mục tiêu: từ 1 đề bài, chọn đúng tập diagram cần thiết (không vẽ dư, không vẽ thiếu), vẽ tuần tự, và đặt tên file theo thứ tự để người dùng dễ theo dõi khi tự học lại.

## Quy ước thư mục đề bài

Mỗi đề bài nằm trong **1 folder riêng**, và nội dung đề bài luôn là file **README.md** — folder đó chỉ chứa đúng 1 file README.md này.

Người dùng luôn cung cấp 1 **path** trỏ tới file hoặc folder đề bài; nếu không cung cấp path nào, tự động lấy **file đang mở** trong editor làm đề bài. Việc đầu tiên là xác định đúng folder chứa README.md rồi đọc file này để lấy nội dung — không yêu cầu người dùng paste lại đề nếu đã có file.

Toàn bộ file diagram tạo ra ở bước 5 lưu trong các **folder con** bên trong folder đề bài (`black-box/`, `white-box/`, hoặc biến thể `baseline-black-box/`, `baseline-white-box/`, `change-black-box/`, `change-white-box/` — xem tên và điều kiện tạo folder ở Bước 5) — không lưu ra ngoài folder đề bài.

## Danh sách diagram được phép chọn (danh sách đầy đủ — không tự thêm loại nào khác ngoài đây)

Chỉ được chọn diagram trong danh sách này, kể cả khi đề bài gợi ý 1 loại UML chuẩn khác không có ở đây (Object, Composite structure, Package, Communication, Interaction overview, Timing, Profile...) — những loại đó không áp dụng được cho bài tập chỉ có đề bài dạng text, không có code/runtime thật, nên không nằm trong danh sách.

**Structural**
- Class — tự thiết kế breakdown hợp lý cho phần white-box, không cần đề bài nêu rõ OOP; chỉ thêm khi breakdown OOP thực sự làm rõ thiết kế, không bắt buộc luôn phải có
- Component — tự thiết kế breakdown service/module hợp lý cho phần white-box, không cần đề bài nêu rõ; gần như luôn vẽ được trừ hệ thống quá đơn giản
- Deployment — tự thiết kế hạ tầng hợp lý cho phần white-box, không cần đề bài nêu rõ; chỉ thêm khi việc triển khai có điểm đáng nói cho bài toán, không bắt buộc luôn phải có

Class/Component/Deployment được tự do tự đề xuất thiết kế (xem "Phân lớp black-box/white-box" bên dưới) — ràng buộc duy nhất: đề xuất phải nhất quán với những gì đề bài đã mô tả, không tự bịa công nghệ/ràng buộc trái với đề bài.

**Behavioral**
- **Use case — bắt buộc, luôn vẽ** — actor nào làm được gì
- Activity — quy trình nhiều bước, rẽ nhánh, song song, swimlane
- State machine — vòng đời trạng thái của 1 object
- **Sequence — bắt buộc, luôn vẽ** — thứ tự message giữa các thành phần theo thời gian; số lượng = số action trong Use case (1 action ↔ 1 sequence)

**Ngoài UML**
- **ERD — bắt buộc, luôn vẽ** — bảng, quan hệ dữ liệu
- Flowchart — tổ tiên của Activity, đơn giản hơn, không swimlane/song song
- C4 model (Context/Container) — 4 tầng zoom kiến trúc; dùng khi cần góc nhìn kiến trúc tổng thể/hạ tầng hơn là 1 Component đơn lẻ (xem Bước 3)

## Phân lớp black-box / white-box (áp dụng ở Bước 5)

Mỗi diagram thuộc đúng 1 trong 2 lớp, quyết định nó được lưu vào folder nào — đi từ ngoài (black-box) vào trong (white-box):

- **black-box** (vỏ ngoài — mô tả hệ thống nhìn từ bên ngoài, không quan tâm code bên trong tổ chức thế nào): Use case, ERD, Sequence, State machine, Activity, Flowchart, C4 Context.
- **white-box** (lõi trong — mô tả cấu trúc/thiết kế bên trong hệ thống được tổ chức thế nào): Class, Component, Deployment, C4 Container.

Use case, ERD, Sequence luôn bắt buộc nên **black-box luôn có nội dung**. Trong white-box, Component, Class, và Deployment đều là **tự đề xuất thiết kế**, không cần tín hiệu tường minh từ đề bài; riêng C4 Container vẫn tuân theo điều kiện tín hiệu kiến trúc như cũ (xem Bước 3). **Mỗi context luôn có đủ cả 2 folder black-box và white-box** (xem Bước 5) — việc chọn diagram nào bên trong mỗi folder vẫn theo đúng logic bắt buộc/tuỳ chọn/tự thiết kế ở trên, chỉ là không bao giờ bỏ hẳn 1 folder.

## Quy trình

### Bước 1 — Đọc đề bài, liệt kê tín hiệu

Xác định folder đề bài và đọc file đề bài bên trong (xem "Quy ước thư mục đề bài" ở trên).

**3 loại luôn bắt buộc, không phụ thuộc tín hiệu:**
- **Use case** — liệt kê actor và toàn bộ action/capability mà đề bài mô tả (kể cả khi chỉ có 1 actor duy nhất, vẫn vẽ để làm nền so sánh sau này).
- **ERD** — mô hình hoá dữ liệu/entity liên quan, kể cả khi đề bài chỉ ngầm định (vd "lưu thông tin user") chứ không liệt kê rõ bảng.
- **Sequence** — vẽ **1 sequence cho mỗi action** đã liệt kê trong Use case (1 action ↔ 1 file sequence). Đây là cách suy ra số lượng Sequence, không cần dò tín hiệu "có flow cụ thể" riêng như trước nữa.

Sau khi có 3 loại trên, đọc kỹ phần còn lại của đề bài để gạch ra các tín hiệu cho nhóm diagram **tuỳ chọn** (chỉ vẽ khi tín hiệu thực sự xuất hiện):

| Tín hiệu trong đề bài | Diagram gợi ý |
|---|---|
| Có 1 object/entity trải qua nhiều trạng thái (order, task, ticket...) | State machine |
| Có quy trình nghiệp vụ nhiều bước, nhiều phòng ban/actor duyệt qua lại, hoặc có việc chạy song song thật | Activity |
| Có quy trình đơn giản, tuyến tính, không cần swimlane/song song | Flowchart |
| Có nhắc tới nhiều service/module tách biệt gọi nhau, cần góc nhìn kiến trúc tổng thể hơn là 1 Component đơn lẻ (microservices, message queue...) | C4 Container (thay cho Component, xem bước 3) |
| Cần giải thích tổng quan hệ thống cho người không rành kỹ thuật (stakeholder, khách hàng) | C4 Context |

Nếu đề bài **không** có tín hiệu nào cho 1 nhóm tuỳ chọn ở bảng trên — **không vẽ nhóm đó**. Use case, ERD, Sequence thì luôn vẽ như quy định ở trên, không xét tín hiệu; Component, Class (khi hợp lý), và Deployment (khi hợp lý) cũng luôn được tự thiết kế cho white-box mà không cần tín hiệu riêng.

### Bước 2 — Phát hiện đề bài có "hiện trạng" + "thay đổi"

Nhiều đề bài không mô tả 1 hệ thống tĩnh, mà có dạng: hệ thống **đã có sẵn** một phần (baseline/hiện trạng), rồi yêu cầu **thêm/sửa/nâng cấp** một tính năng lên trên đó — vd "hiện chỉ có đăng nhập email/password... thêm nút Sign in with Google", "đang dùng single DB... nay cần shard theo tenant". Tín hiệu nhận biết: đề bài có cụm kiểu "hiện tại/hiện có/hiện chỉ có/đang dùng/đã có" đi kèm "thêm/bổ sung/nâng cấp/migrate/mở rộng/thay thế/song song với".

Nếu phát hiện đúng pattern này:
- Tách nội dung đề bài thành 2 nhóm: **Hiện trạng** (behaviour/data/flow đang có — chỉ phần liên quan trực tiếp tới thay đổi sắp tới, không vẽ lan man toàn bộ hệ thống cũ nếu đề không mô tả chi tiết) và **Thay đổi** (behaviour/data/flow sau khi thêm/sửa tính năng mới).
- **Cả 2 nhóm đều bắt buộc có đủ Use case + ERD + Sequence riêng** (đúng quy định bắt buộc ở Bước 1) — nhóm Hiện trạng vẽ Use case/ERD/Sequence phản ánh đúng actor/data/action **trước** khi có thay đổi, nhóm Thay đổi vẽ lại phản ánh đúng actor/data/action **sau** khi có thay đổi. Đây là cách chính để người đọc so sánh trực quan "trước vẽ gì — sau vẽ gì", nên không được bỏ qua Use case/ERD ở nhóm nào dù nhóm đó "có vẻ đơn giản".
- Áp dụng logic chọn diagram tuỳ chọn ở Bước 1 cho từng nhóm riêng — tín hiệu xuất hiện ở phần nào thì chọn diagram tuỳ chọn cho nhóm đó. 2 nhóm có thể ra diagram tuỳ chọn khác nhau (vd chỉ nhóm Thay đổi có State machine vì entity mới phát sinh trạng thái).
- Nếu đề bài **không** có pattern này (mô tả 1 hệ thống ngay từ đầu, không tách biệt baseline riêng) — bỏ qua bước này, đi thẳng theo quy trình chọn diagram thông thường như 1 nhóm duy nhất.

Mỗi nhóm (Hiện trạng/Thay đổi) khi vẽ ở Bước 5 tiếp tục được tách thành folder black-box/white-box riêng — xem cấu trúc folder cụ thể ở Bước 5.

### Bước 3 — Ưu tiên C4 hơn Component/Deployment riêng lẻ khi mô tả kiến trúc

Nếu đề bài cần cả "kiến trúc tổng thể" lẫn "hạ tầng deploy", ưu tiên vẽ theo **C4 model** (Context rồi tới Container) thay vì vẽ riêng Component diagram + Deployment diagram — C4 dễ đọc hơn và ít trùng lặp thông tin hơn. Chỉ tách riêng Component/Deployment khi người dùng cần đúng format UML chuẩn (vd tài liệu thi/học thuật).

### Bước 4 — Xác nhận danh sách trước khi vẽ

Nếu đề bài có pattern hiện trạng/thay đổi (Bước 2), liệt kê theo 2 nhóm: "Nhóm Hiện trạng cần N1 diagram: [tên] vì [lý do]. Nhóm Thay đổi cần N2 diagram: [tên] vì [lý do]." Nếu không có pattern này, liệt kê 1 danh sách bình thường: "Đề bài này cần N diagram: [tên] vì [lý do 1 câu]". Nếu danh sách này rõ ràng (đa số trường hợp), tiến thẳng qua bước 5 mà không cần hỏi lại. Chỉ hỏi lại người dùng khi có sự mơ hồ thực sự (vd đề bài vừa có thể hiểu là monolith vừa có thể hiểu là microservices).

### Bước 5 — Tạo folder black-box/white-box, vẽ từng diagram, lưu file có prefix số

**Tên folder — hardcode đúng tên sau, không tự đổi:**

- Đề bài **không** có pattern hiện trạng/thay đổi (Bước 2) — luôn tạo đúng 2 folder ngay trong folder đề bài:
  - `black-box/` — Use case + ERD + Sequence bắt buộc nằm ở đây, cộng thêm diagram tuỳ chọn nào có tín hiệu (State machine/Activity/Flowchart/C4 Context)
  - `white-box/` — Component (và Class/Deployment khi hợp lý) tự thiết kế nằm ở đây, cộng thêm C4 Container nếu có tín hiệu kiến trúc

- Đề bài **có** pattern hiện trạng/thay đổi — luôn tạo đúng 4 folder ngay trong folder đề bài (không lồng thêm cấp folder `baseline`/`change` cha, tên context ghép thẳng vào tên folder):
  - `baseline-black-box/`
  - `baseline-white-box/`
  - `change-black-box/`
  - `change-white-box/`

Cả 2 (hoặc 4) folder này **luôn được tạo**, không có ngoại lệ bỏ folder — việc "lựa diagram nào để vẽ" chỉ diễn ra **bên trong** mỗi folder (theo đúng logic bắt buộc/tuỳ chọn/tự thiết kế ở Bước 1), không phải ở việc có tạo folder hay không.

**Thứ tự vẽ:** nếu có pattern hiện trạng/thay đổi, giữ nguyên quy tắc không xen kẽ 2 nhóm — vẽ **hết nhóm Hiện trạng** (cả black-box lẫn white-box của nhóm này) rồi mới sang **nhóm Thay đổi**. Trong mỗi nhóm (hoặc trong trường hợp không có pattern, coi cả đề bài là 1 nhóm), luôn vẽ **hết black-box trước, rồi mới tới white-box** — đúng thứ tự đi từ ngoài vào trong:

Trong `black-box/`:
1. C4 Context (nếu có) — tổng quan nhất
2. **Use case (bắt buộc)** — ai làm được gì, liệt kê đủ action
3. **ERD (bắt buộc)** — dữ liệu
4. **Sequence (bắt buộc, 1 file cho mỗi action đã liệt kê ở Use case)** — vẽ đủ, không bỏ sót action nào
5. State machine (nếu có)
6. Activity / Flowchart (nếu có)

Trong `white-box/`:
7. C4 Container (nếu có tín hiệu kiến trúc nhiều service + hạ tầng, xem Bước 3 — thay thế cho Component khi đúng trường hợp này)
8. **Component (tự thiết kế breakdown hợp lý)** — mặc định luôn thử vẽ, trừ khi hệ thống quá đơn giản không có gì để tách
9. Class (tự thiết kế nếu OOP breakdown thực sự làm rõ thiết kế — không bắt buộc luôn phải có)
10. Deployment (tự thiết kế hạ tầng hợp lý nếu việc triển khai có điểm đáng nói cho bài toán — không bắt buộc luôn phải có, giống Class)

Mỗi diagram lưu thành 1 file Markdown riêng trong đúng folder ở trên, đặt tên **bằng tiếng Anh** (kể cả khi đề bài và nội dung bên trong file là tiếng Việt). **Số thứ tự bắt đầu lại từ 1 trong mỗi folder** (không dùng số chạy xuyên suốt toàn bộ đề bài nữa — folder đã phân nhóm rõ theo context + lớp):

```
<tên-folder>/<số thứ tự trong folder>-<loại-diagram>-<mô-tả-ngắn-tiếng-Anh>.md
```

Ví dụ (không có pattern hiện trạng/thay đổi, Use case có 2 action "Login" và "Reset password", tự thiết kế 1 Auth Service cho phần white-box):
```
black-box/1-usecase-role-permission.md
black-box/2-erd-core-schema.md
black-box/3-sequence-login.md
black-box/4-sequence-reset-password.md
black-box/5-state-order-lifecycle.md
white-box/1-component-auth-service.md
```

Ví dụ (có pattern hiện trạng/thay đổi — cả 2 nhóm đều tự thiết kế được Component, riêng Class chỉ nhóm Thay đổi mới cần vì phát sinh strategy pattern cho 2 cách login):
```
baseline-black-box/1-usecase-login.md
baseline-black-box/2-erd-user-schema.md
baseline-black-box/3-sequence-email-password-login.md
baseline-white-box/1-component-auth-service.md
change-black-box/1-usecase-add-google-login.md
change-black-box/2-erd-add-google-identity.md
change-black-box/3-sequence-oauth-login.md
change-white-box/1-component-auth-service.md
change-white-box/2-class-oauth-strategy.md
```

### Bước 6 — Kỹ thuật vẽ cho từng loại

Dùng **mermaid code fence** (```` ```mermaid ... ``` ````, render trực tiếp trong markdown preview của VSCode/GitHub) cho các loại mermaid hỗ trợ tốt: **ERD (`erDiagram`), Class (`classDiagram`), State machine (`stateDiagram-v2`), Sequence (`sequenceDiagram`), Flowchart/Activity (`flowchart` hoặc `graph`), C4 (`C4Context`, `C4Container`)**.

Dùng **SVG vẽ tay** nhúng trực tiếp (raw `<svg>...</svg>` ngay trong file markdown, theo design system — xem `visualize:read_me` module `diagram` nếu cần chi tiết box/màu/spacing) cho: **Use case** (mermaid không hỗ trợ tốt — actor stick figure + ellipse use case + system boundary box + include/extend), và **Component/Deployment** khi được vẽ (dùng dạng structural diagram — box lồng nhau, không phải icon 3D kiểu UML cũ).

**Lưu ý bắt buộc khi nhúng SVG trong markdown:** tuyệt đối không để dòng trống bên trong khối `<svg>...</svg>` — markdown (CommonMark HTML block type 7) coi dòng trống là điểm kết thúc block HTML, nên 1 dòng trống ở giữa sẽ cắt khối SVG thành nhiều mảnh rời rạc và preview sẽ không lên hình/lên sai. Toàn bộ nội dung từ `<svg ...>` tới `</svg>` phải liền mạch, không dòng trống; đặt luôn `width="100%"` (kèm `viewBox`) trên thẻ `<svg>` để tránh bị co kích thước mặc định khi hiển thị.

Mỗi file markdown là 1 tài liệu độc lập: bắt đầu bằng `# <tên diagram>`, sau đó vài câu mô tả ngắn gồm **2 phần**: (1) diagram này thể hiện gì, và (2) **lý do vì sao chọn loại diagram này** — tín hiệu nào trong đề bài dẫn tới lựa chọn đó (đối chiếu bảng tín hiệu ở Bước 1). Câu đầu tiên phải nói rõ diagram này thuộc lớp **black-box** hay **white-box**, và nếu đề bài có pattern hiện trạng/thay đổi (Bước 2) thì nói rõ luôn thuộc nhóm **Hiện trạng** hay **Thay đổi**. Sau đó tới khối mermaid hoặc SVG. Không cần phụ thuộc gì từ file khác.

### Bước 7 — Sau khi vẽ xong 1 vòng, tự check lại

Sau khi vẽ hết danh sách ở bước 4 (cả 2 nhóm nếu có), tự hỏi lại: "Còn khía cạnh nào của đề bài chưa được diagram nào cover không?" — trong đó luôn kiểm tra riêng:
- Mỗi action trong (các) Use case đã có đúng 1 file Sequence tương ứng trong đúng folder `black-box` (hoặc `baseline-black-box`/`change-black-box`) chưa.
- Mọi entity đã liệt kê ở Use case/đề bài đã có mặt trong ERD chưa.
- Folder `white-box`/`baseline-white-box`/`change-white-box` đã có ít nhất file Component (tự thiết kế) chưa — folder này luôn được tạo (xem Bước 5) nên không được để trống, nếu thấy trống nghĩa là đang bỏ sót chứ không phải hệ thống "không cần" white-box.

Nếu có thiếu — vẽ bổ sung, đánh số tiếp theo trong đúng folder (không chèn số ở giữa, không đổi số file cũ). Nếu không phát hiện gì thiếu — dừng lại, không cố vẽ thêm cho đủ bộ.

### Bước 8 — Liệt kê lại toàn bộ file

Liệt kê đường dẫn tất cả file đã tạo trong folder đề bài, nhóm theo từng folder (black-box/white-box, hoặc baseline-*/change-* nếu có pattern), theo đúng thứ tự số đã đặt trong từng folder. Tóm tắt ngắn gọn: đã tạo folder nào, mỗi folder có bao nhiêu diagram, loại nào, lý do chọn — không lặp lại nội dung diagram trong text vì file đã thể hiện đầy đủ.
