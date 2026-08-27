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

---

## Marketplace đa seller đối soát payout cho người bán

**Repository:** `reconciliation-marketplace-seller-payout`

**Hệ thống:** Một marketplace đa seller, sàn thu tiền từ người mua rồi định kỳ trả (payout) cho từng seller sau khi trừ phí hoa hồng sàn và các khoản hoàn tiền phát sinh.

**Vai trò của flow:** Đối soát giữa số tiền hệ thống nội bộ tính seller được nhận (sau phí, sau hoàn tiền) và số tiền thực tế đã được chuyển ra ngoài qua cổng thanh toán cho từng seller.

**Yêu cầu cụ thể:**
- Một seller có thể có hàng trăm đơn hàng hoàn tất trong kỳ, một số đơn bị hoàn tiền/khiếu nại xử lý sau khi đã tính vào kỳ payout — cần xác định rõ mốc thời gian cắt (cutoff) nhất quán cho từng kỳ, tránh trường hợp đơn vừa hoàn tất bị tính hai lần vào hai kỳ khác nhau hoặc bị bỏ sót không kỳ nào tính tới.
- Khi một đơn hàng đã được tính vào payout đã chuyển tiền cho seller, nhưng sau đó phát sinh hoàn tiền cho người mua, hệ thống phải xử lý đúng việc thu hồi/trừ vào kỳ payout tiếp theo của seller đó — đối soát phải phát hiện được các khoản âm này thay vì chỉ xử lý cộng dồn một chiều.
- Lệnh chuyển tiền ra ngoài cho seller qua cổng thanh toán có thể ở trạng thái pending/thất bại/bị trả lại (ví dụ tài khoản ngân hàng seller sai thông tin) — đối soát phải phân biệt rõ "đã lên lệnh chuyển" và "seller thực nhận được tiền", không đóng kỳ payout khi còn lệnh chuyển chưa xác nhận thành công.
- Phí hoa hồng sàn được tính tại thời điểm đơn hàng hoàn tất có thể khác với mức phí áp dụng tại thời điểm đối soát nếu chính sách phí thay đổi giữa chừng — đối soát phải dùng đúng mức phí đã chốt tại thời điểm phát sinh giao dịch, không tính lại theo cấu hình phí hiện tại gây sai lệch số liệu lịch sử.
- Khi phát hiện sai lệch giữa số tiền sàn tính seller được nhận và số tiền thực đã chuyển, cần có cơ chế giữ (hold) kỳ payout tiếp theo của seller đó lại để điều chỉnh, thay vì tiếp tục chuyển tiền theo số liệu có khả năng sai, đồng thời không làm ảnh hưởng payout của các seller khác không liên quan.

---

## SaaS thuê bao đối soát billing nội bộ với payment processor

**Repository:** `reconciliation-saas-subscription-billing-processor`

**Hệ thống:** Một SaaS thu phí thuê bao định kỳ hàng tháng, khách hàng có thể nâng/hạ cấp gói giữa chu kỳ, thanh toán xử lý qua một payment processor bên ngoài.

**Vai trò của flow:** Đối soát giữa hệ thống billing nội bộ (invoice phát sinh, các khoản điều chỉnh proration khi đổi gói) và số tiền thực tế processor đã thu được/báo cáo lại theo từng chu kỳ thanh toán.

**Yêu cầu cụ thể:**
- Khi khách hàng đổi gói giữa chu kỳ, hệ thống billing nội bộ phát sinh nhiều invoice điều chỉnh nhỏ (giảm trừ phần chưa dùng của gói cũ, cộng thêm phần gói mới) trong cùng một khoảng thời gian ngắn — đối soát phải khớp đúng tổng các điều chỉnh này với đúng một lượt thu tiền tương ứng từ processor, tránh hiểu nhầm là nhiều giao dịch riêng lẻ không khớp.
- Processor có thể thử thu tiền thất bại rồi tự động retry theo lịch riêng của họ (dunning) trong vài ngày sau ngày đến hạn — đối soát phải xử lý được trạng thái "đang chờ retry" khác với "thất bại hẳn", không đóng sổ chu kỳ khi khoản thu vẫn còn khả năng thành công ở lần retry tiếp theo.
- Khách hàng hủy gói giữa chu kỳ có thể được hoàn lại phần tiền chưa dùng, khoản hoàn này phát sinh sau khi invoice gốc đã đối soát khớp từ trước — cần cập nhật lại trạng thái đối soát của invoice đó, không để hệ thống báo cáo doanh thu vẫn tính khoản đã hoàn là doanh thu thực.
- Số tiền processor thực báo cáo có thể đã trừ phí xử lý giao dịch trước khi ghi có, trong khi hệ thống billing nội bộ ghi nhận theo giá trị gộp trước phí — đối soát phải tách rõ hai luồng số liệu này để không nhầm chênh lệch phí xử lý thành sai lệch thực sự cần điều tra.
- Khi múi giờ đóng chu kỳ billing nội bộ không trùng khớp với thời điểm processor tổng hợp báo cáo theo lịch của họ, một số giao dịch cận ranh giới ngày có thể bị xếp lệch chu kỳ giữa hai bên — cần định nghĩa rõ quy tắc xếp chu kỳ thống nhất và có bước xử lý riêng cho các giao dịch cận biên này thay vì báo sai lệch giả mỗi tháng.

---

## Ứng dụng gọi xe đối soát cuốc xe, tiền tài xế và hoa hồng

**Repository:** `reconciliation-ride-hailing-trip-driver-payout`

**Hệ thống:** Một ứng dụng gọi xe, khách đặt xe, tài xế hoàn tất cuốc, nền tảng thu tiền và chia lại cho tài xế sau khi trừ hoa hồng, một số cuốc bị hủy giữa chừng hoặc phát sinh tranh chấp giá.

**Vai trò của flow:** Đối soát giữa số tiền khách trả, số tiền tài xế được nhận, và hoa hồng nền tảng giữ lại, đảm bảo ba khoản này khớp đúng cho từng cuốc xe và không có cuốc nào bị tính sai/bỏ sót.

**Yêu cầu cụ thể:**
- Cuốc xe bị hủy giữa chừng sau khi tài xế đã di chuyển một đoạn có thể phát sinh phí hủy một phần cho tài xế nhưng không phải giá cuốc đầy đủ — đối soát phải phân biệt rõ luồng tiền của cuốc hoàn tất và cuốc hủy có phí bồi thường, không gộp chung logic tính hoa hồng như cuốc bình thường.
- Giá cuốc thực tế cuối cùng có thể khác giá ước tính ban đầu hiển thị cho khách (do đổi lộ trình giữa đường, chờ đợi phát sinh, tranh chấp về quãng đường) — đối soát phải dùng đúng giá đã chốt cuối cùng làm cơ sở tính hoa hồng và tiền tài xế, xử lý được trường hợp giá bị điều chỉnh lại sau khi cuốc đã kết thúc và tiền đã tạm ghi nhận.
- Tài xế nhận tiền mặt trực tiếp từ khách cho một số cuốc trong khi hoa hồng nền tảng vẫn phải được thu lại từ tài xế theo cách khác (trừ vào ví/đợt thanh toán sau) — đối soát phải theo dõi riêng luồng tiền mặt này để không nhầm là "tiền chưa về" khi thực chất khách đã trả trực tiếp cho tài xế.
- Khi có tranh chấp giá giữa khách và tài xế được xử lý sau khi cuốc đã kết thúc và tiền đã được chia, khoản điều chỉnh phát sinh sau phải được đối soát lại đúng cho cả ba bên mà không làm sai lệch báo cáo hoa hồng của các cuốc khác không liên quan trong cùng kỳ.
- Với tài xế có khối lượng cuốc lớn mỗi ngày, tổng tiền tài xế nhận được tổng hợp theo đợt thanh toán định kỳ phải khớp chính xác với tổng từng cuốc riêng lẻ cộng lại trừ các khoản điều chỉnh phát sinh trong kỳ — cần cơ chế đối soát ở cả mức chi tiết từng cuốc và mức tổng hợp theo đợt để phát hiện sai lệch dù nhỏ không bị pha loãng/che khuất khi chỉ nhìn tổng số cuối kỳ.
