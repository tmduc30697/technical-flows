# Distributed transaction trong Split payment/Marketplace payout — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần chia một khoản tiền cho nhiều bên qua nhiều hệ thống tài chính — split payment marketplace, chia cước ride-hailing, chia doanh thu creator, payout crowdfunding, và escrow marketplace B2B — nhằm luyện cách đảm bảo tổng tiền chia luôn khớp, xử lý payout thất bại từng phần, và audit trail đầy đủ trong web app thực tế.

---

## Split payment giữa marketplace, seller và affiliate

**Repository:** `split-payment-marketplace-seller-affiliate`

**Hệ thống:** Marketplace cho phép người mua trả tiền một lần, sau đó tự động chia tiền cho seller, platform (hoa hồng), và affiliate (người giới thiệu) nếu có.

**Vai trò của flow:** Điều phối giao dịch phân tán đảm bảo tổng số tiền chia ra luôn khớp đúng 100% số tiền buyer trả, và mọi bên liên quan đều nhận đúng phần dù hệ thống có sự cố giữa đường.

**Yêu cầu cụ thể:**
- Toàn bộ quá trình chia tiền (tính hoa hồng, ghi nhận từng phần cho seller/platform/affiliate) phải là atomic ở mức nghiệp vụ — nếu một phần ghi nhận thất bại, phải rollback/compensate toàn bộ, không để trạng thái tiền bị "chia dở".
- Có transaction ID gốc duy nhất cho giao dịch buyer trả tiền, và mọi bản ghi con (payout cho seller, affiliate) phải tham chiếu về transaction ID này để tra soát/audit đầy đủ.
- Xử lý được trường hợp affiliate không xác định được ngay lúc thanh toán (do delay tracking) — phải có cơ chế "giữ lại" phần hoa hồng affiliate trong một khoảng thời gian hợp lý trước khi chốt chia tiền hoàn toàn cho seller/platform.
- Nếu một phần payout (ví dụ chuyển tiền cho seller) thất bại do lỗi hệ thống ngân hàng, phải retry với idempotency key riêng cho từng payout, không retry toàn bộ giao dịch gốc (tránh chia tiền 2 lần cho phần đã thành công).
- Có báo cáo đối soát định kỳ: tổng tiền buyer trả phải luôn bằng tổng các phần đã chia (seller + platform fee + affiliate + refund nếu có), phát hiện ngay nếu có sai lệch.

---

## Chia tiền cước ride-hailing cho driver, platform và thuế

**Repository:** `split-payment-ride-hailing-driver-tax`

**Hệ thống:** App gọi xe nhận tiền từ khách, cần tự động chia cho driver (phần lớn), platform (phí dịch vụ), và trích một phần cho nghĩa vụ thuế theo quy định.

**Vai trò của flow:** Đảm bảo mỗi chuyến đi hoàn thành đều được chia tiền đúng tỉ lệ, đúng đối tượng, và có thể truy vết ngược khi có tranh chấp giữa driver và platform về số tiền nhận được.

**Yêu cầu cụ thể:**
- Việc chia tiền chỉ được thực hiện sau khi chuyến đi được xác nhận hoàn thành và thanh toán từ khách đã thành công — không chia tiền dựa trên trạng thái "đang xử lý" chưa chắc chắn.
- Tỉ lệ chia (driver/platform/thuế) áp dụng tại thời điểm chuyến đi phải được lưu lại cùng giao dịch (snapshot), để nếu sau này tỉ lệ chia thay đổi (chính sách mới), các giao dịch cũ không bị tính sai lại.
- Nếu payout cho driver thất bại (ví dụ tài khoản ngân hàng driver không hợp lệ), tiền đó phải được giữ ở trạng thái "chờ xử lý" rõ ràng, không được platform tự động giữ luôn hay chia tiếp cho phần khác.
- Driver phải có thể xem chi tiết breakdown từng chuyến đi (giá cước, phần platform giữ, phần thực nhận) để minh bạch, giảm tranh chấp/khiếu nại.
- Có cơ chế đối soát cuối ngày/cuối kỳ: tổng tiền khách trả trong kỳ phải khớp với tổng đã chia cho driver + platform + thuế + các khoản treo (pending), không có khoản tiền "biến mất" không rõ đi đâu.

---

## Chia doanh thu cho nhiều content creator trong một gói subscription chung

**Repository:** `split-payment-creator-subscription-revenue-share`

**Hệ thống:** Nền tảng nội dung số nơi user trả một khoản subscription chung, doanh thu được chia cho các creator dựa trên tỉ lệ view/engagement nội dung của họ trong kỳ.

**Vai trò của flow:** Điều phối việc phân chia một khoản tiền lớn (tổng doanh thu subscription) cho nhiều creator theo công thức tính toán phức tạp, đảm bảo tổng chia ra không vượt hoặc thiếu so với doanh thu thực.

**Yêu cầu cụ thể:**
- Việc tính tỉ lệ engagement của từng creator phải hoàn tất và "đóng băng" (freeze) dữ liệu trước khi bắt đầu chia tiền của kỳ đó, tránh trường hợp dữ liệu view vẫn đang thay đổi trong lúc chia tiền gây sai lệch.
- Tổng số tiền chia cho tất cả creator trong kỳ phải bằng chính xác doanh thu kỳ đó sau khi trừ phần platform giữ lại — có bước validate tổng trước khi thực sự transfer tiền đi.
- Nếu quá trình chia tiền cho một creator cụ thể thất bại (ví dụ thông tin thanh toán chưa cập nhật), phải cách ly lỗi đó — không làm chậm/thất bại việc chia tiền cho các creator khác trong cùng kỳ.
- Phải lưu lại đầy đủ công thức và dữ liệu đầu vào đã dùng để tính payout của từng creator trong từng kỳ (audit trail), để có thể giải thích/tranh chấp khi creator thắc mắc về số tiền nhận được.
- Có quy trình xử lý điều chỉnh hồi tố (ví dụ phát hiện gian lận view sau khi đã chia tiền) — định nghĩa rõ cách trừ vào kỳ payout tiếp theo, không được tự ý rút lại tiền đã chuyển.

---

## Payout cho nhiều bên trong nền tảng gây quỹ (crowdfunding)

**Repository:** `split-payment-crowdfunding-payout`

**Hệ thống:** Nền tảng crowdfunding nhận tiền góp từ nhiều backer cho một dự án, sau khi dự án đạt mục tiêu cần chuyển tiền cho người tạo dự án, trừ phí platform, và xử lý hoàn tiền nếu dự án không đạt mục tiêu.

**Vai trò của flow:** Điều phối giao dịch phân tán quyết định "tất cả hoặc không có gì" ở mức toàn dự án — chỉ payout khi đạt điều kiện, và phải hoàn tiền toàn bộ backer nếu không đạt, xử lý đúng dù có hàng nghìn giao dịch góp tiền riêng lẻ.

**Yêu cầu cụ thể:**
- Việc "chốt" dự án đạt/không đạt mục tiêu phải là một quyết định atomic tại đúng thời điểm deadline, dựa trên tổng tiền góp đã xác nhận (không tính các giao dịch đang pending/chưa capture) tại thời điểm đó.
- Nếu dự án đạt mục tiêu, quá trình chuyển tiền cho người tạo dự án (sau khi trừ phí) phải theo dõi được trạng thái từng phần, và nếu thất bại phải retry an toàn (idempotent) không chuyển trùng.
- Nếu dự án không đạt mục tiêu, phải hoàn tiền cho toàn bộ backer đã góp — xử lý theo batch với khả năng resume nếu quá trình hoàn tiền bị gián đoạn giữa đường (ví dụ mới hoàn được 60% backer thì hệ thống crash).
- Có cơ chế idempotency ở cấp từng backer cho việc hoàn tiền, để nếu job hoàn tiền bị chạy lại (do retry ở cấp hệ thống) không hoàn tiền trùng 2 lần cho cùng một backer.
- Cung cấp cho cả người tạo dự án và backer khả năng theo dõi trạng thái payout/hoàn tiền real-time (đang xử lý/hoàn tất/thất bại cần hỗ trợ), tránh tình trạng "tiền đi đâu không biết" gây mất niềm tin nền tảng.

---

## Giao dịch nhiều bên trong marketplace B2B có escrow

**Repository:** `split-payment-b2b-marketplace-escrow`

**Hệ thống:** Marketplace B2B nơi buyer trả tiền vào một tài khoản escrow trung gian, tiền chỉ thực sự chuyển cho seller sau khi buyer xác nhận nhận hàng đúng, có thể có bên vận chuyển/bảo hiểm tham gia chia phí.

**Vai trò của flow:** Điều phối giao dịch phân tán phức tạp qua nhiều giai đoạn (giữ tiền escrow → xác nhận giao hàng → release tiền cho seller/logistics), xử lý đúng các trường hợp tranh chấp giữa buyer/seller.

**Yêu cầu cụ thể:**
- Tiền phải được giữ đúng trạng thái "escrow - chưa release" trong toàn bộ thời gian chờ xác nhận, và chỉ chuyển sang trạng thái "released" qua một transaction rõ ràng khi buyer xác nhận hoặc hết thời hạn tự động xác nhận theo chính sách.
- Nếu buyer khiếu nại/tranh chấp trước khi hết hạn, phải tự động chặn (freeze) việc release tiền tự động, chuyển sang trạng thái "đang tranh chấp" chờ xử lý thủ công/trọng tài, không cho phép luồng tự động vô tình release tiền trong lúc đang tranh chấp.
- Phần phí chia cho logistics/bảo hiểm (nếu có) phải được release riêng biệt và có thể xảy ra ở thời điểm khác so với phần chính chuyển cho seller (ví dụ phí logistics release ngay khi giao hàng, còn tiền seller release sau khi buyer xác nhận) — không gộp chung một transaction duy nhất.
- Toàn bộ lịch sử trạng thái escrow (giữ, tranh chấp, release, hoàn tiền) của một giao dịch phải truy vết được đầy đủ theo thời gian, phục vụ làm chứng cứ khi có tranh chấp pháp lý giữa buyer và seller.
- Có cơ chế idempotency và khả năng resume rõ ràng cho bước release tiền nếu quá trình bị gián đoạn (ví dụ hệ thống thanh toán bên thứ ba timeout giữa lúc release) — không được để tiền ở trạng thái không rõ ràng (đã trừ escrow nhưng chưa chắc đã tới seller).
