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
