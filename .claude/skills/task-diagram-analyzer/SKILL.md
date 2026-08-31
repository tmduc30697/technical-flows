---
name: task-diagram-analyzer
description: Đọc 1 đề bài/yêu cầu hệ thống, xác định những loại diagram (UML + ERD + C4 + flowchart) thực sự phù hợp để mô tả nó, rồi vẽ từng cái theo thứ tự, lưu ra file có đánh số prefix (1-, 2-, 3-...). Dùng skill này khi người dùng đưa 1 đề bài/yêu cầu hệ thống và muốn "phân tích", "vẽ hết các diagram có thể", hoặc muốn học cách chọn diagram cho 1 bài toán cụ thể. KHÔNG dùng khi người dùng chỉ hỏi lý thuyết chung về 1 loại diagram (vd "state diagram là gì") mà không có đề bài cụ thể nào để áp dụng.
---

# Task diagram analyzer

Mục tiêu: từ 1 đề bài, chọn đúng tập diagram cần thiết (không vẽ dư, không vẽ thiếu), vẽ tuần tự, và đặt tên file theo thứ tự để người dùng dễ theo dõi khi tự học lại.

## Quy ước thư mục đề bài

Mỗi đề bài nằm trong **1 folder riêng**, và nội dung đề bài thường là **1 file** bên trong folder đó (vd `de-bai.md`, `problem.md`, `README.md`, `*.txt`...). Khi người dùng trỏ tới 1 đề (đường dẫn folder, hoặc đường dẫn file đề bài, hoặc chỉ nói tên đề), việc đầu tiên là xác định đúng folder này rồi đọc file đề bài bên trong để lấy nội dung — không yêu cầu người dùng paste lại đề nếu đã có file. Nếu folder có nhiều file, ưu tiên file có tên/rõ ràng nhất là đề bài (README, đề-bài, problem, statement...); nếu không chắc, hỏi lại người dùng.

Toàn bộ file diagram tạo ra ở bước 5 **lưu ngay trong folder đó** — cùng cấp với file đề bài, không lưu ra thư mục khác.

## Giới hạn quan trọng: chỉ có đề bài, không có codebase

Input của skill này luôn là 1 đề bài/yêu cầu dạng text — **không có code thật, không có hệ thống đang chạy**. Một số diagram UML mô tả *chi tiết implementation* hoặc *trạng thái runtime thật* — không thể vẽ chính xác nếu chỉ dựa vào đề bài, vẽ ra sẽ là bịa đặt chi tiết chứ không phải mô tả đúng hệ thống. Nhóm này **loại khỏi danh sách được chọn**, trừ khi người dùng tự cung cấp thêm code/schema thật trong cùng lúc:

- ❌ **Object** — cần dữ liệu instance thật tại 1 thời điểm chạy, đề bài không có
- ❌ **Composite structure** — cần chi tiết implementation nội bộ 1 class/component đã viết
- ❌ **Package** — cần cấu trúc thư mục/module thật của codebase
- ❌ **Communication** — cùng loại thông tin với Sequence nhưng notation khác, không thêm giá trị khi chỉ có đề bài, luôn ưu tiên Sequence thay thế
- ❌ **Interaction overview** — cần đã có sẵn nhiều Sequence diagram chi tiết để tổng hợp lại, không tự sinh ra từ đề bài thuần
- ❌ **Timing** — cần số đo thời gian runtime thật (latency, mốc mili giây...), đề bài không có
- ❌ **Profile** — mở rộng UML bằng stereotype riêng, gần như không dùng thực tế, càng không thể suy ra từ đề bài
- ⚠️ **Class** — chỉ vẽ nếu đề bài mô tả rõ ràng OOP design (interface, kế thừa cụ thể); nếu đề bài chỉ nói về data/entity thông thường thì dùng ERD thay thế, không suy diễn method/attribute không có căn cứ
- ⚠️ **Deployment** — chỉ vẽ nếu đề bài **nêu rõ** hạ tầng (server, cloud provider, container...); không tự bịa ra hạ tầng khi đề bài không nhắc tới
- ⚠️ **Component** — chỉ vẽ nếu đề bài mô tả rõ các module/service tách biệt; nếu chỉ là 1 hệ thống nguyên khối chưa rõ cách chia module, ưu tiên C4 Container ở mức khái quát hơn là vẽ Component chi tiết bịa ra ranh giới module

## Danh sách diagram được phép chọn (đã lọc, dùng được từ đề bài thuần)

**Structural**
- Class — chỉ khi đề bài nêu rõ OOP design (có điều kiện, xem trên)
- Component — chỉ khi đề bài nêu rõ module/service tách biệt (có điều kiện, xem trên)
- Deployment — chỉ khi đề bài nêu rõ hạ tầng (có điều kiện, xem trên)

**Behavioral**
- Use case — actor nào làm được gì
- Activity — quy trình nhiều bước, rẽ nhánh, song song, swimlane
- State machine — vòng đời trạng thái của 1 object
- Sequence — thứ tự message giữa các thành phần theo thời gian

**Ngoài UML**
- ERD — bảng, quan hệ dữ liệu
- Flowchart — tổ tiên của Activity, đơn giản hơn, không swimlane/song song
- C4 model (Context/Container) — 4 tầng zoom kiến trúc, dùng Context/Container là đủ vì Component/Code cần chi tiết code thật

## Quy trình

### Bước 1 — Đọc đề bài, liệt kê tín hiệu

Xác định folder đề bài và đọc file đề bài bên trong (xem "Quy ước thư mục đề bài" ở trên). Đọc kỹ đề bài, gạch ra các "tín hiệu" mà đề bài có, mỗi tín hiệu trỏ tới 1 nhóm diagram:

| Tín hiệu trong đề bài | Diagram gợi ý |
|---|---|
| Có nhắc tới dữ liệu, bảng, model, entity | ERD |
| Có 1 object/entity trải qua nhiều trạng thái (order, task, ticket...) | State machine |
| Có 1 flow cụ thể, nhiều thành phần gọi nhau theo thứ tự (API call, OAuth, checkout...) | Sequence |
| Có nhắc tới actor/role/quyền hạn khác nhau (user, admin, guest...) | Use case |
| Có quy trình nghiệp vụ nhiều bước, nhiều phòng ban/actor duyệt qua lại, hoặc có việc chạy song song thật | Activity |
| Có quy trình đơn giản, tuyến tính, không cần swimlane/song song | Flowchart |
| Có nhắc tới nhiều service/module tách biệt gọi nhau (microservices, message queue...) | Component hoặc C4 Container |
| Có nhắc tới hạ tầng, server, cloud, container, region deploy | Deployment (hoặc C4, xem bước 3) |
| Có OOP phức tạp, nhiều class kế thừa/interface — không chỉ là data thuần | Class |
| Cần giải thích tổng quan hệ thống cho người không rành kỹ thuật (stakeholder, khách hàng) | C4 Context |

Nếu đề bài **không** có tín hiệu nào cho 1 nhóm — **không vẽ nhóm đó**. Mặc định ưu tiên bộ core: **ERD, Sequence, State machine, Use case** — đây là 4 loại phù hợp với phần lớn đề bài web/backend thông thường và luôn vẽ được từ đề bài thuần; chỉ thêm loại khác khi tín hiệu ở bảng trên thực sự xuất hiện rõ trong đề.

Object, Composite structure, Package, Communication, Interaction overview, Timing, Profile — đã loại hẳn khỏi phạm vi skill này vì cần codebase/runtime thật (xem phần "Giới hạn quan trọng" ở trên), không cân nhắc dù đề bài có tín hiệu gì đi nữa.

### Bước 2 — Phát hiện đề bài có "hiện trạng" + "thay đổi"

Nhiều đề bài không mô tả 1 hệ thống tĩnh, mà có dạng: hệ thống **đã có sẵn** một phần (baseline/hiện trạng), rồi yêu cầu **thêm/sửa/nâng cấp** một tính năng lên trên đó — vd "hiện chỉ có đăng nhập email/password... thêm nút Sign in with Google", "đang dùng single DB... nay cần shard theo tenant". Tín hiệu nhận biết: đề bài có cụm kiểu "hiện tại/hiện có/hiện chỉ có/đang dùng/đã có" đi kèm "thêm/bổ sung/nâng cấp/migrate/mở rộng/thay thế/song song với".

Nếu phát hiện đúng pattern này:
- Tách nội dung đề bài thành 2 nhóm: **Hiện trạng** (behaviour/data/flow đang có — chỉ phần liên quan trực tiếp tới thay đổi sắp tới, không vẽ lan man toàn bộ hệ thống cũ nếu đề không mô tả chi tiết) và **Thay đổi** (behaviour/data/flow sau khi thêm/sửa tính năng mới).
- Áp dụng lại đúng logic chọn diagram ở Bước 1 cho từng nhóm riêng — tín hiệu xuất hiện ở phần nào thì chọn diagram cho nhóm đó. 2 nhóm có thể ra diagram type khác nhau, hoặc cùng loại nhưng nội dung khác nhau (vd Sequence hiện trạng là login email/password, Sequence thay đổi là thêm nhánh OAuth).
- Nếu đề bài **không** có pattern này (mô tả 1 hệ thống ngay từ đầu, không tách biệt baseline riêng) — bỏ qua bước này, đi thẳng theo quy trình chọn diagram thông thường như 1 nhóm duy nhất.

### Bước 3 — Ưu tiên C4 hơn Component/Deployment riêng lẻ khi mô tả kiến trúc

Nếu đề bài cần cả "kiến trúc tổng thể" lẫn "hạ tầng deploy", ưu tiên vẽ theo **C4 model** (Context rồi tới Container) thay vì vẽ riêng Component diagram + Deployment diagram — C4 dễ đọc hơn và ít trùng lặp thông tin hơn. Chỉ tách riêng Component/Deployment khi người dùng cần đúng format UML chuẩn (vd tài liệu thi/học thuật).

### Bước 4 — Xác nhận danh sách trước khi vẽ

Nếu đề bài có pattern hiện trạng/thay đổi (Bước 2), liệt kê theo 2 nhóm: "Nhóm Hiện trạng cần N1 diagram: [tên] vì [lý do]. Nhóm Thay đổi cần N2 diagram: [tên] vì [lý do]." Nếu không có pattern này, liệt kê 1 danh sách bình thường: "Đề bài này cần N diagram: [tên] vì [lý do 1 câu]". Nếu danh sách này rõ ràng (đa số trường hợp), tiến thẳng qua bước 5 mà không cần hỏi lại. Chỉ hỏi lại người dùng khi có sự mơ hồ thực sự (vd đề bài vừa có thể hiểu là monolith vừa có thể hiểu là microservices).

### Bước 5 — Vẽ từng diagram theo thứ tự, lưu file có prefix số

Thứ tự vẽ mặc định trong 1 nhóm (vẽ cái nào trước cũng được nhưng nên theo logic từ tổng quan → chi tiết):
1. C4 Context (nếu có) — tổng quan nhất
2. Use case (nếu có) — ai làm được gì
3. ERD (nếu có) — dữ liệu
4. Sequence (nếu có, có thể nhiều cái — 1 file/flow)
5. State machine (nếu có)
6. Activity / Flowchart (nếu có)
7. Component / Deployment / Class (nếu đủ điều kiện, xem phần "Giới hạn quan trọng")

Nếu đề bài có pattern hiện trạng/thay đổi (Bước 2): vẽ **hết toàn bộ diagram nhóm Hiện trạng trước** (theo đúng thứ tự ưu tiên trên), rồi mới vẽ **hết toàn bộ diagram nhóm Thay đổi** — không xen kẽ 2 nhóm. Số thứ tự file vẫn tăng dần liên tục xuyên suốt cả 2 nhóm, không reset về 1 khi chuyển nhóm.

Mỗi diagram lưu thành 1 file Markdown riêng **ngay trong folder chứa đề bài** (cùng cấp với file đề bài đã đọc ở bước 1), đặt tên **bằng tiếng Anh** (kể cả khi đề bài và nội dung bên trong file là tiếng Việt):

```
<số thứ tự>-<loại-diagram>-<mô-tả-ngắn-tiếng-Anh>.md
```

Ví dụ (không có pattern hiện trạng/thay đổi): `1-usecase-role-permission.md`, `2-erd-core-schema.md`, `3-sequence-google-oauth.md`, `4-state-order-lifecycle.md`

Ví dụ (có pattern hiện trạng/thay đổi — chèn thêm `baseline`/`change` vào tên để dễ theo dõi): `1-sequence-baseline-email-password-login.md`, `2-sequence-change-add-google-oauth.md`, `3-erd-change-add-google-identity.md`.

Số thứ tự tăng dần theo đúng thứ tự vẽ thực tế trong bước này, không phải theo bảng ưu tiên cố định — nếu đề bài chỉ cần Sequence + State thì file là `1-sequence-...md` và `2-state-...md`, không nhảy số.

### Bước 6 — Kỹ thuật vẽ cho từng loại

Dùng **mermaid code fence** (```` ```mermaid ... ``` ````, render trực tiếp trong markdown preview của VSCode/GitHub) cho các loại mermaid hỗ trợ tốt: **ERD (`erDiagram`), Class (`classDiagram`), State machine (`stateDiagram-v2`), Sequence (`sequenceDiagram`), Flowchart/Activity (`flowchart` hoặc `graph`), C4 (`C4Context`, `C4Container`)**.

Dùng **SVG vẽ tay** nhúng trực tiếp (raw `<svg>...</svg>` ngay trong file markdown, theo design system — xem `visualize:read_me` module `diagram` nếu cần chi tiết box/màu/spacing) cho: **Use case** (mermaid không hỗ trợ tốt — actor stick figure + ellipse use case + system boundary box + include/extend), và **Component/Deployment** nếu đủ điều kiện được vẽ (dùng dạng structural diagram — box lồng nhau, không phải icon 3D kiểu UML cũ).

**Lưu ý bắt buộc khi nhúng SVG trong markdown:** tuyệt đối không để dòng trống bên trong khối `<svg>...</svg>` — markdown (CommonMark HTML block type 7) coi dòng trống là điểm kết thúc block HTML, nên 1 dòng trống ở giữa sẽ cắt khối SVG thành nhiều mảnh rời rạc và preview sẽ không lên hình/lên sai. Toàn bộ nội dung từ `<svg ...>` tới `</svg>` phải liền mạch, không dòng trống; đặt luôn `width="100%"` (kèm `viewBox`) trên thẻ `<svg>` để tránh bị co kích thước mặc định khi hiển thị.

Mỗi file markdown là 1 tài liệu độc lập: bắt đầu bằng `# <tên diagram>`, sau đó vài câu mô tả ngắn gồm **2 phần**: (1) diagram này thể hiện gì, và (2) **lý do vì sao chọn loại diagram này** — tín hiệu nào trong đề bài dẫn tới lựa chọn đó (đối chiếu bảng tín hiệu ở Bước 1). Nếu đề bài có pattern hiện trạng/thay đổi (Bước 2), câu đầu tiên phải nói rõ diagram này thuộc nhóm **Hiện trạng** hay **Thay đổi**. Sau đó tới khối mermaid hoặc SVG. Không cần phụ thuộc gì từ file khác.

### Bước 7 — Sau khi vẽ xong 1 vòng, tự check lại

Sau khi vẽ hết danh sách ở bước 4 (cả 2 nhóm nếu có), tự hỏi lại: "Còn khía cạnh nào của đề bài chưa được diagram nào cover không?" Nếu có (vd đề bài có nhắc tới 1 luồng phụ chưa vẽ sequence riêng, hoặc có thêm 1 entity chưa đưa vào ERD) — vẽ bổ sung, đánh số tiếp theo (không chèn số ở giữa, không đổi số file cũ).

Nếu không phát hiện gì thiếu — dừng lại, không cố vẽ thêm cho đủ bộ.

### Bước 8 — Liệt kê lại toàn bộ file

Liệt kê đường dẫn tất cả file đã tạo trong folder đề bài, theo đúng thứ tự số đã đặt. Tóm tắt ngắn gọn: đã vẽ bao nhiêu diagram, loại nào, lý do chọn — không lặp lại nội dung diagram trong text vì file đã thể hiện đầy đủ.
