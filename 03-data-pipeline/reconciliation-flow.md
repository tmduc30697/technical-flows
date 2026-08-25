# Reconciliation flow giữa nhiều hệ thống thanh toán — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — e-commerce, marketplace đa seller, SaaS thuê bao, ứng dụng gọi xe, ngân hàng số, và sàn giao dịch crypto — nhằm luyện đủ các góc của flow đối soát (reconciliation) dữ liệu tiền giữa nhiều hệ thống (tính đúng đắn tuyệt đối, xử lý lệch/mismatch, idempotency, xử lý giao dịch bị hoàn/hủy, báo cáo audit).

---

## E-commerce đối soát giao dịch giữa cổng thanh toán và hệ thống đơn hàng

**Repository:** `reconciliation-ecommerce-payment-gateway-orders`

**Hệ thống:** Một sàn e-commerce nhận thanh toán qua cổng thanh toán bên thứ ba (ví dụ Stripe/VNPay) cho các đơn hàng.

**Vai trò của flow:** Đối soát định kỳ giữa danh sách giao dịch từ cổng thanh toán và trạng thái đơn hàng trong hệ thống nội bộ để phát hiện sai lệch (đơn báo đã thanh toán nhưng cổng không ghi nhận, hoặc ngược lại).

**Yêu cầu cụ thể:**
- Mỗi giao dịch từ cổng thanh toán phải được khớp (match) với đúng một đơn hàng nội bộ dựa trên mã tham chiếu duy nhất, phát hiện được các trường hợp không khớp (giao dịch không có đơn tương ứng, hoặc đơn báo đã trả tiền nhưng không tìm thấy giao dịch).
- Xử lý đúng các giao dịch bị hoàn tiền (refund) hoặc bị đảo (chargeback) xảy ra sau khi đơn hàng đã đối soát thành công trước đó — phải cập nhật lại trạng thái đối soát tương ứng, không để đơn hàng hiển thị "đã đối soát khớp" khi thực tế tiền đã được hoàn.
- Báo cáo đối soát phải phân loại rõ các loại sai lệch (thiếu giao dịch, thừa giao dịch, sai số tiền, sai trạng thái) để đội vận hành xử lý đúng quy trình cho từng loại, không gộp chung một danh sách "lỗi".
- Job đối soát phải chạy được lại nhiều lần trên cùng một khoảng thời gian dữ liệu mà không tạo ra báo cáo trùng lặp hoặc đánh dấu sai lệch nhiều lần cho cùng một giao dịch đã được xử lý.
- Các sai lệch không tự giải quyết được (cần con người can thiệp) phải được đưa vào hàng đợi xử lý thủ công có theo dõi trạng thái, đảm bảo không có sai lệch nào bị bỏ quên không xử lý.

---

## Marketplace đối soát payout cho seller giữa ví nội bộ và ngân hàng

**Repository:** `reconciliation-marketplace-seller-payout`

**Hệ thống:** Một marketplace giữ số dư (ví nội bộ) cho từng seller từ doanh thu bán hàng, định kỳ chuyển tiền (payout) ra tài khoản ngân hàng của seller.

**Vai trò của flow:** Đối soát giữa số tiền được ghi nhận đã chuyển đi từ hệ thống nội bộ và số tiền thực tế ngân hàng xác nhận đã chuyển thành công tới seller.

**Yêu cầu cụ thể:**
- Mỗi lệnh payout phải theo dõi được đủ 3 trạng thái: đã tạo lệnh nội bộ, đã gửi tới ngân hàng, và ngân hàng xác nhận thành công/thất bại — đối soát phải phát hiện được lệnh bị "treo" ở trạng thái giữa quá lâu.
- Khi ngân hàng báo lệnh chuyển thất bại (sai số tài khoản, tài khoản đóng), số tiền phải được hoàn lại đúng vào ví nội bộ của seller và không được coi là "đã chi" trong đối soát doanh thu tổng.
- Không được tạo ra hai lệnh payout cho cùng một seller với cùng một kỳ thanh toán dù job payout bị chạy lại do lỗi hệ thống (idempotency theo seller + kỳ thanh toán).
- Tổng số tiền payout theo đối soát phải khớp chính xác với tổng doanh thu seller đã ghi nhận trừ đi phí hoa hồng và các khoản giữ lại (hold do tranh chấp/hoàn hàng) — có báo cáo chi tiết breakdown khi có sai lệch.
- Cung cấp cho seller lịch sử payout minh bạch (đã nhận, đang xử lý, thất bại kèm lý do) để giảm ticket hỗ trợ, và đội vận hành có công cụ tra soát một payout cụ thể xuyên suốt cả ba hệ thống liên quan.

---

## SaaS thuê bao đối soát giữa Stripe subscription và billing ledger nội bộ

**Repository:** `reconciliation-saas-stripe-billing-ledger`

**Hệ thống:** Một SaaS B2B tính phí thuê bao hàng tháng qua Stripe, đồng thời duy trì một billing ledger nội bộ để tính doanh thu công nhận (revenue recognition) và báo cáo tài chính.

**Vai trò của flow:** Đối soát giữa các sự kiện thanh toán/gia hạn/hủy từ Stripe và bản ghi tương ứng trong billing ledger nội bộ để đảm bảo doanh thu ghi nhận chính xác.

**Yêu cầu cụ thể:**
- Mọi sự kiện từ Stripe (thanh toán thành công, thất bại, subscription bị downgrade/upgrade, hủy) phải được webhook nhận và phản ánh vào ledger nội bộ; đối soát định kỳ phải phát hiện được sự kiện nào bị thiếu do webhook lỗi/timeout.
- Xử lý đúng trường hợp khách hàng nâng/hạ cấp gói giữa kỳ thanh toán (proration) — số tiền ghi nhận trong ledger phải khớp với cách Stripe tính proration, không tính sai lệch dẫn đến báo cáo doanh thu sai.
- Khi Stripe retry một thanh toán thất bại (dunning) nhiều lần trước khi thành công hoặc hủy hẳn, ledger nội bộ phải phản ánh đúng trạng thái cuối cùng, không ghi nhận doanh thu ở các lần thử thất bại trung gian.
- Đối soát phải chạy được theo từng kỳ kế toán (tháng/quý) và tạo ra báo cáo sai lệch có thể dùng trực tiếp cho mục đích kế toán, không chỉ là log kỹ thuật khó đọc cho người không chuyên.
- Webhook trùng lặp từ Stripe (gửi lại cùng một event do retry ở phía Stripe) không được làm ghi nhận doanh thu hai lần trong ledger — phải khử trùng theo event ID.

---

## Ứng dụng gọi xe đối soát doanh thu giữa tài xế, phí platform, và bên xử lý thanh toán

**Repository:** `reconciliation-ride-hailing-driver-revenue`

**Hệ thống:** Một app gọi xe có ba bên liên quan đến tiền của mỗi chuyến đi: khách trả tiền qua cổng thanh toán, platform giữ lại phần trăm hoa hồng, và tài xế nhận phần còn lại.

**Vai trò của flow:** Đối soát số tiền của từng chuyến đi qua ba bên để đảm bảo tổng tiền thu từ khách đúng bằng tổng tiền chia cho tài xế cộng hoa hồng platform, không có khoản nào biến mất hoặc trùng lặp.

**Yêu cầu cụ thể:**
- Với mỗi chuyến đi, đối soát phải xác nhận: số tiền cổng thanh toán ghi nhận đã thu từ khách = số tiền ghi vào tài khoản tài xế + hoa hồng platform + các khoản phụ phí/khuyến mãi áp dụng, sai lệch dù nhỏ cũng phải được gắn cờ.
- Xử lý đúng các chuyến đi bị hủy giữa đường hoặc bị tranh chấp (khách khiếu nại tính sai tiền) — phần tiền đã tạm giữ/tạm chia cho tài xế phải được điều chỉnh lại đúng khi tranh chấp được giải quyết, và đối soát phải phản ánh trạng thái "đang tranh chấp" riêng biệt.
- Khuyến mãi/giảm giá (mã giảm giá, trợ giá theo khu vực) áp dụng cho chuyến đi phải được tính đúng vào phần chênh lệch giữa tiền khách trả và tiền tài xế nhận, không để platform tự chịu lỗ số liệu không rõ nguồn hoặc tính nhầm vào phần tài xế.
- Payout cho tài xế thường được gộp theo lô (nhiều chuyến đi trong một kỳ trả tiền) — đối soát phải truy được ngược từ một lô payout về đúng danh sách chuyến đi cấu thành, phục vụ tra soát khi tài xế khiếu nại về một chuyến cụ thể.
- Báo cáo đối soát tổng theo ngày/tuần phải khớp với báo cáo tài chính tổng của công ty; mọi sai lệch giữa hai báo cáo phải được điều tra và có nguyên nhân gốc trước khi đóng sổ kỳ đó.

---

## Ngân hàng số đối soát giao dịch với đối tác thanh toán/mạng thẻ

**Repository:** `reconciliation-digital-bank-card-network`

**Hệ thống:** Một ngân hàng số (neobank) xử lý giao dịch thẻ và chuyển tiền qua các đối tác bên ngoài (mạng thẻ, cổng chuyển mạch liên ngân hàng).

**Vai trò của flow:** Đối soát cuối ngày giữa sổ cái (ledger) nội bộ của ngân hàng và file/báo cáo giao dịch nhận từ đối tác thanh toán để đảm bảo mọi giao dịch được ghi nhận đúng và số dư khách hàng chính xác.

**Yêu cầu cụ thể:**
- Đối soát phải xử lý được file batch từ đối tác thanh toán có thể đến muộn hoặc theo múi giờ khác với hệ thống nội bộ — phải định nghĩa rõ "ngày giao dịch" theo cùng một chuẩn giữa hai bên để tránh lệch ngày khi đối soát.
- Phát hiện chính xác các loại sai lệch khác nhau: giao dịch có ở đối tác nhưng thiếu trong ledger nội bộ, giao dịch có trong ledger nhưng đối tác không xác nhận, và giao dịch khớp nhưng sai số tiền — mỗi loại phải có quy trình xử lý và mức độ ưu tiên khác nhau vì đây là tiền thật của khách hàng.
- Giao dịch bị đối tác báo hoàn/đảo sau một khoảng thời gian (ví dụ tranh chấp thẻ) phải được xử lý đúng dù giao dịch gốc đã đối soát khớp từ trước, và số dư khách hàng phải được điều chỉnh đúng thời điểm phát hiện.
- Toàn bộ quy trình đối soát và mọi điều chỉnh số dư phát sinh từ đó phải có audit trail đầy đủ, không thể sửa trực tiếp số dư mà không qua bước ghi nhận giao dịch điều chỉnh tương ứng, để đáp ứng yêu cầu kiểm toán ngân hàng.
- Phải có ngưỡng cảnh báo tự động khi tổng sai lệch chưa xử lý trong ngày vượt quá một số tiền nhất định, yêu cầu can thiệp khẩn của đội vận hành trước khi đóng sổ ngày đó.

---

## Sàn giao dịch crypto đối soát số dư giữa ví on-chain và ledger nội bộ

**Repository:** `reconciliation-crypto-onchain-ledger`

**Hệ thống:** Một sàn giao dịch crypto giữ tài sản khách hàng trong ví nóng/lạnh (on-chain) và duy trì một ledger nội bộ ghi số dư khả dụng của từng khách hàng để phục vụ giao dịch tức thời trên sàn.

**Vai trò của flow:** Đối soát định kỳ giữa tổng số dư thực tế trên các ví on-chain và tổng số dư khách hàng cộng lại trong ledger nội bộ, đảm bảo sàn luôn có đủ tài sản backing cho số dư đã ghi nhận.

**Yêu cầu cụ thể:**
- Đối soát phải tính đúng số dư on-chain tại một block height/thời điểm xác định và so khớp với snapshot ledger nội bộ tại đúng thời điểm tương ứng, tránh so sánh lệch thời điểm gây sai lệch giả.
- Giao dịch nạp/rút tiền on-chain đang chờ đủ số confirmation (chưa final) không được tính là "đã hoàn tất" trong ledger nội bộ cho đến khi đạt ngưỡng xác nhận an toàn theo từng loại tài sản, đối soát phải phân biệt rõ trạng thái pending và confirmed.
- Phát hiện được trường hợp tổng số dư ledger nội bộ vượt quá tổng tài sản thực có trên ví on-chain (dấu hiệu nghiêm trọng: lỗi hệ thống hoặc gian lận nội bộ) và phải cảnh báo khẩn cấp ngay lập tức, không chờ báo cáo đối soát định kỳ theo lịch thông thường.
- Xử lý đúng các giao dịch on-chain bị thay thế/hủy do phí gas thấp (replace-by-fee) hoặc bị đảo do fork mạng — không ghi nhận nhầm là hoàn tất khi giao dịch gốc thực chất không còn hiệu lực trên chuỗi.
- Toàn bộ quá trình đối soát và mọi sai lệch phát hiện phải được lưu trữ minh bạch, có khả năng phục vụ báo cáo proof-of-reserves cho khách hàng/cơ quan quản lý khi được yêu cầu.
