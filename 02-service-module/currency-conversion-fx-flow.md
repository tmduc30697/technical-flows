# Currency conversion/FX flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (marketplace xuyên biên giới, remittance fintech, đặt vé du lịch, SaaS billing đa tiền tệ, sàn giao dịch crypto) để luyện việc quy đổi tiền tệ chính xác, minh bạch và chịu được các trường hợp tỷ giá thay đổi giữa lúc giao dịch.

---

## Marketplace bán hàng xuyên biên giới hiển thị giá theo tiền tệ người mua

**Repository:** `fx-marketplace-cross-border-pricing`

**Hệ thống:** Một marketplace cho phép người bán ở một quốc gia niêm yết giá bằng tiền tệ của họ, người mua ở quốc gia khác xem giá đã quy đổi sang tiền tệ của mình.

**Vai trò của flow:** Quy đổi tỷ giá dùng để hiển thị giá tham khảo cho người mua và tính toán số tiền thực tế cần thanh toán/thanh toán lại cho người bán, phải nhất quán giữa lúc xem hàng và lúc thanh toán.

**Yêu cầu cụ thể:**
- Giá hiển thị lúc xem sản phẩm chỉ là giá tham khảo (ước tính theo tỷ giá tại thời điểm xem); phải chốt (lock) tỷ giá thực tế dùng để thanh toán tại đúng thời điểm người mua xác nhận đặt hàng, và hiển thị rõ tỷ giá đã chốt cho người mua trước khi họ trả tiền.
- Xử lý trường hợp tỷ giá thay đổi giữa lúc người mua xem giá và lúc bấm thanh toán (có thể cách vài phút tới vài giờ nếu để hàng trong giỏ) — định nghĩa rõ khoảng thời gian tỷ giá chốt còn hợp lệ, hết hạn thì phải tính lại và thông báo cho người mua nếu giá thay đổi đáng kể.
- Số tiền thực trả cho người bán (bằng tiền tệ của người bán) phải được tính và lưu lại tại thời điểm giao dịch, độc lập với các lần tỷ giá thay đổi sau đó — không tính lại số tiền trả cho người bán theo tỷ giá mới khi có tranh chấp/hoàn tiền xảy ra sau nhiều ngày.
- Làm tròn số tiền đúng theo quy tắc của từng loại tiền tệ (ví dụ JPY không có phần thập phân, một số tiền tệ có 3 chữ số thập phân) — tránh lỗi làm tròn gây lệch tổng khi tính nhiều dòng trong một đơn hàng.
- Ghi log đầy đủ nguồn tỷ giá (provider nào, thời điểm lấy) cho mỗi giao dịch để phục vụ đối soát và giải quyết tranh chấp khi khách hàng thắc mắc về số tiền quy đổi.

---

## Chuyển tiền quốc tế (remittance) trong ứng dụng fintech

**Repository:** `fx-fintech-remittance`

**Hệ thống:** Một ứng dụng cho phép người dùng chuyển tiền từ tài khoản tiền tệ A sang cho người nhận ở quốc gia khác nhận bằng tiền tệ B.

**Vai trò của flow:** FX flow phải tính chính xác số tiền người nhận sẽ nhận được, bao gồm cả phí quy đổi, và đảm bảo người gửi biết chính xác số tiền cuối cùng trước khi xác nhận giao dịch không hoàn lại.

**Yêu cầu cụ thể:**
- Trước khi người dùng xác nhận giao dịch, phải hiển thị đầy đủ và minh bạch: tỷ giá áp dụng, phí quy đổi (margin so với tỷ giá thị trường), và số tiền cuối cùng người nhận sẽ nhận — không được ẩn phí quy đổi vào trong tỷ giá một cách khó nhận biết.
- Tỷ giá hiển thị cho người dùng xác nhận phải được "giữ" (quote lock) trong một khoảng thời gian ngắn xác định (ví dụ 30-60 giây); nếu người dùng xác nhận sau khi hết hạn, phải lấy lại tỷ giá mới và yêu cầu xác nhận lại, không dùng tỷ giá cũ đã hết hạn để thực hiện giao dịch thật.
- Xử lý trường hợp giao dịch được xác nhận đúng lúc tỷ giá thị trường biến động mạnh (ví dụ do tin tức kinh tế) — có ngưỡng chặn tự động nếu biến động vượt quá mức bất thường trong thời gian ngắn, yêu cầu xác nhận thủ công hoặc cảnh báo cho người dùng.
- Đảm bảo tính idempotent cho việc trừ tiền tài khoản gửi và ghi nhận số tiền cho tài khoản nhận trong cùng một giao dịch quy đổi — nếu một bước bị lỗi giữa đường (ví dụ hệ thống crash sau khi trừ tiền người gửi nhưng trước khi hoàn tất quy đổi), phải có cơ chế đảm bảo tiền không bị mất hoặc nhân đôi khi retry.
- Lưu trữ đầy đủ audit trail cho mỗi giao dịch quy đổi (tỷ giá áp dụng, nguồn tỷ giá, phí, thời điểm chốt) theo yêu cầu compliance của ngành tài chính, không được cho phép sửa đổi sau khi giao dịch đã hoàn tất.

---

## Đặt vé máy bay/khách sạn hiển thị giá đa tiền tệ

**Repository:** `fx-travel-booking-multi-currency`

**Hệ thống:** Một nền tảng đặt vé du lịch cho phép người dùng ở nhiều quốc gia xem và đặt vé với giá hiển thị theo tiền tệ địa phương, nhưng thanh toán thực tế xử lý qua một tiền tệ gốc (ví dụ USD) với đối tác hàng không/khách sạn.

**Vai trò của flow:** Quy đổi tiền tệ ở đây phải đồng bộ giữa nhiều bước của một booking dài (tìm kiếm, giữ chỗ, thanh toán, có thể hủy/đổi vé sau) trải qua thời gian đáng kể.

**Yêu cầu cụ thể:**
- Khi người dùng giữ chỗ (hold booking) trong một khoảng thời gian trước khi thanh toán, giá hiển thị bằng tiền tệ địa phương phải được chốt tại thời điểm giữ chỗ và giữ nguyên cho tới khi hold hết hạn, không thay đổi theo tỷ giá dao động trong lúc người dùng đang cân nhắc.
- Khi người dùng hủy vé và được hoàn tiền, số tiền hoàn phải được tính theo tỷ giá đã áp dụng tại thời điểm đặt vé ban đầu (không phải tỷ giá hiện tại lúc hủy), trừ khi chính sách hoàn tiền của nền tảng quy định khác — và phải thông báo rõ cho người dùng cách tính.
- Xử lý trường hợp giá gốc từ đối tác (hàng không/khách sạn) được báo bằng tiền tệ khác với tiền tệ người dùng thanh toán và khác cả tiền tệ nền tảng dùng để đối soát nội bộ — phải tính đúng qua đủ các bước quy đổi trung gian, không làm tròn nhân dồn lỗi qua từng bước.
- Đảm bảo khi hiển thị nhiều lựa chọn (nhiều chuyến bay/khách sạn) cùng lúc theo tiền tệ địa phương, toàn bộ danh sách phải dùng cùng một tỷ giá tại một thời điểm lấy dữ liệu (snapshot), tránh tình huống lấy tỷ giá riêng cho từng item khiến kết quả sắp xếp theo giá bị sai lệch nếu tỷ giá thay đổi giữa các lần gọi.
- Thiết kế cơ chế fallback khi nguồn tỷ giá bên ngoài tạm thời không khả dụng — dùng tỷ giá gần nhất còn hợp lệ kèm cảnh báo độ trễ dữ liệu, không để toàn bộ trang tìm kiếm bị lỗi vì thiếu tỷ giá.

---

## SaaS billing đa tiền tệ cho khách hàng doanh nghiệp toàn cầu

**Repository:** `fx-saas-billing-global`

**Hệ thống:** Một nền tảng SaaS B2B tính phí subscription hàng tháng, khách hàng ở các quốc gia khác nhau được chọn tiền tệ thanh toán theo hợp đồng.

**Vai trò của flow:** FX flow dùng để tính hóa đơn định kỳ và xử lý các thay đổi hợp đồng (upgrade/downgrade giữa kỳ) một cách nhất quán qua nhiều kỳ thanh toán.

**Yêu cầu cụ thể:**
- Mỗi khách hàng ký hợp đồng bằng một tiền tệ cố định (không đổi theo mỗi kỳ thanh toán trừ khi có thỏa thuận lại), giá gốc niêm yết trong hợp đồng có thể ở tiền tệ khác (ví dụ USD) — phải định nghĩa rõ tỷ giá dùng để tính hóa đơn được chốt vào thời điểm nào (đầu kỳ hợp đồng, đầu mỗi kỳ thanh toán, hay theo ngày xuất hóa đơn) và áp dụng nhất quán cho toàn bộ chu kỳ hợp đồng.
- Khi khách hàng upgrade/downgrade giữa kỳ (proration), số tiền chênh lệch phải được tính đúng theo tiền tệ hợp đồng của khách hàng, không tính nhầm bằng tiền tệ niêm yết gốc rồi quy đổi lại gây sai số cộng dồn.
- Xử lý trường hợp một khách hàng muốn đổi tiền tệ thanh toán giữa chu kỳ hợp đồng (ví dụ công ty mở rộng sang thị trường khác) — phải có quy trình rõ ràng để chốt số dư/công nợ theo tiền tệ cũ trước khi chuyển sang tính theo tiền tệ mới, tránh lẫn lộn công nợ giữa hai tiền tệ trong cùng một tài khoản.
- Đảm bảo báo cáo doanh thu nội bộ (dùng để đối soát tài chính) quy đổi toàn bộ hóa đơn đa tiền tệ về một tiền tệ báo cáo chuẩn, có ghi rõ tỷ giá dùng cho mỗi hóa đơn để kế toán có thể truy vết và đối soát với ngân hàng.
- Thiết kế thông báo trước cho khách hàng khi tỷ giá cập nhật ảnh hưởng tới số tiền hóa đơn kỳ tới (nếu hợp đồng có điều khoản điều chỉnh theo tỷ giá), tránh khách hàng bị "bất ngờ" bởi số tiền hóa đơn thay đổi mà không có cảnh báo trước.

---

## Quy đổi giữa tiền pháp định và tài sản số trên sàn giao dịch

**Repository:** `fx-crypto-exchange-fiat-conversion`

**Hệ thống:** Một sàn giao dịch tài sản số cho phép người dùng nạp tiền pháp định (VND/USD) và quy đổi sang các loại tài sản số, với tỷ giá biến động liên tục theo thời gian thực.

**Vai trò của flow:** Khác với các bối cảnh trên (tỷ giá tương đối ổn định), ở đây FX flow phải xử lý tỷ giá biến động từng giây và rủi ro trượt giá (slippage) trong lúc khớp lệnh.

**Yêu cầu cụ thể:**
- Khi người dùng đặt lệnh quy đổi, phải hiển thị rõ đây là giá ước tính hay giá đã chốt (đối với lệnh market vs limit) — với lệnh market, phải cảnh báo rõ mức trượt giá tối đa có thể xảy ra trước khi người dùng xác nhận.
- Xử lý race condition khi nhiều người dùng đặt lệnh quy đổi cùng lúc trong lúc giá đang biến động nhanh — đảm bảo mỗi lệnh được khớp theo đúng giá tại đúng thời điểm xử lý của nó, không có lệnh nào "chen ngang" được hưởng giá của lệnh khác.
- Định nghĩa ngưỡng trượt giá tối đa cho phép (ví dụ 1%) mà hệ thống tự động hủy lệnh nếu giá thị trường di chuyển vượt ngưỡng này trong khoảng thời gian xử lý lệnh, thay vì khớp lệnh với giá quá bất lợi cho người dùng.
- Đảm bảo số dư tài sản số và tiền pháp định của người dùng được cập nhật atomic (cả hai cùng thành công hoặc cùng thất bại) — không để xảy ra trường hợp trừ tiền pháp định nhưng không cộng tài sản số do lỗi giữa giao dịch.
- Lưu lại đầy đủ lịch sử tỷ giá tại mọi thời điểm khớp lệnh (không chỉ tỷ giá hiện tại) để phục vụ tính thuế, báo cáo lãi/lỗ, và giải quyết tranh chấp khi người dùng khiếu nại về giá khớp lệnh.
