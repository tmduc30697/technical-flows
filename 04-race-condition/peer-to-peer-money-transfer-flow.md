# Peer-to-peer money transfer flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống chuyển tiền cá nhân (ví điện tử, ngân hàng số, app chia tiền nhóm, chuyển tiền quốc tế, tặng tiền trong livestream) để luyện việc chuyển tiền giữa 2 người dùng đảm bảo atomic, không mất/trùng tiền dưới các thao tác đồng thời.

---

## Chuyển tiền tức thời giữa 2 user trong ví điện tử

**Repository:** `p2p-transfer-ewallet-instant`

**Hệ thống:** Một ví điện tử cho phép user chuyển tiền cho nhau tức thời qua số điện thoại/username, phổ biến dùng để trả nợ nhanh giữa bạn bè.

**Vai trò của flow:** Flow chuyển tiền phải trừ tiền người gửi và cộng tiền người nhận trong 1 transaction atomic, đảm bảo không mất tiền/tạo tiền dù có bao nhiêu request chuyển tiền chạy đồng thời liên quan đến 2 tài khoản đó.

**Yêu cầu cụ thể:**
- Lock 2 dòng số dư (người gửi, người nhận) theo thứ tự cố định dựa trên khóa chính (ví dụ account_id nhỏ hơn lock trước) để tránh deadlock khi có giao dịch ngược hướng chạy song song (A chuyển cho B, B chuyển cho A cùng lúc).
- Mô tả cụ thể: user A có 100k, gửi đồng thời 2 lệnh chuyển 100k cho B và 100k cho C (double-submit hoặc cố ý khai thác) — chỉ đúng 1 lệnh được thực hiện thành công dựa trên số dư khả dụng thực tại thời điểm transaction chạy, lệnh còn lại phải báo lỗi "không đủ số dư" một cách atomic (kiểm tra và trừ trong cùng câu lệnh, không tách rời 2 bước).
- Mỗi lệnh chuyển tiền phải có idempotency key riêng theo request của client, để nếu app bị lag và user bấm gửi lại, không tạo ra 2 giao dịch chuyển tiền trùng nhau.
- Khi người nhận không tồn tại hoặc tài khoản bị khóa ngay tại thời điểm transaction đang cộng tiền (ví dụ tài khoản vừa bị khóa do vi phạm), phải rollback toàn bộ giao dịch (cả phần trừ tiền người gửi), không để tiền "biến mất" khỏi số dư người gửi mà không đến được người nhận.
- Có ledger giao dịch ghi rõ từng lần chuyển (người gửi, người nhận, số tiền, thời điểm, trạng thái) để có thể tái dựng lại số dư từ lịch sử, phục vụ tra soát khi user khiếu nại "đã chuyển nhưng người nhận không thấy tiền".

---

## Chuyển tiền liên ngân hàng qua hệ thống thanh toán số của ngân hàng

**Repository:** `p2p-transfer-interbank-digital-payment`

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền tới tài khoản ở ngân hàng khác, phải đi qua hệ thống thanh toán liên ngân hàng (napas/swift-like) với độ trễ và khả năng timeout không rõ kết quả.

**Vai trò của flow:** Flow phải trừ tiền tài khoản người gửi ngay (đảm bảo không dùng tiền đó cho việc khác) trước khi gửi lệnh sang hệ thống liên ngân hàng, và xử lý đúng khi hệ thống liên ngân hàng trả kết quả trễ hoặc không rõ ràng.

**Yêu cầu cụ thể:**
- Khi user bấm chuyển tiền, phải trừ tiền và đặt trạng thái "đang xử lý" (hold) trong 1 transaction atomic trước khi gửi lệnh sang hệ thống liên ngân hàng, để tiền đó không thể bị dùng cho giao dịch khác (ví dụ 1 lệnh chuyển tiền khác) trong lúc đang chờ kết quả.
- Mô tả cụ thể: gửi lệnh sang hệ thống liên ngân hàng bị timeout (không rõ đã xử lý hay chưa ở phía đối tác) — hệ thống KHÔNG được tự động gửi lại lệnh giống y hệt ngay (rủi ro chuyển tiền 2 lần nếu lệnh gốc thực ra đã thành công), phải có bước kiểm tra trạng thái qua mã tham chiếu trước khi quyết định retry hoặc hoàn tiền.
- Nếu kết quả trả về sau (qua callback hoặc polling) là "thất bại" (tài khoản đích không hợp lệ, ngân hàng đích từ chối), phải hoàn tiền lại đúng số tiền đã trừ (bao gồm phí nếu có) vào tài khoản người gửi, atomic và có thể truy vết được giao dịch hoàn tiền này gắn với giao dịch gốc nào.
- Khi user hủy lệnh chuyển tiền (nếu hệ thống cho phép hủy trong 1 khoảng thời gian ngắn) đúng lúc callback báo thành công cũng vừa đến, quy định rõ ai thắng: nếu tiền đã thực sự rời khỏi hệ thống (callback thành công là nguồn sự thật), không được hủy nữa mà phải thông báo cho user giao dịch đã hoàn tất.
- Có cơ chế đối soát với hệ thống liên ngân hàng theo lô định kỳ (ví dụ mỗi giờ) để phát hiện các giao dịch "đang xử lý" quá lâu không nhận được kết quả, tự động escalate cho team vận hành xử lý thủ công.

---

## Chia tiền nhóm (split bill) sau khi ăn uống chung trong app xã hội

**Repository:** `p2p-transfer-split-bill-social`

**Hệ thống:** Một app cho phép nhóm bạn chia tiền hóa đơn sau khi ăn uống/du lịch chung: 1 người tạo "yêu cầu chia tiền" gửi tới nhiều người, mỗi người thanh toán phần của mình.

**Vai trò của flow:** Flow phải xử lý nhiều người thanh toán phần của họ độc lập và gần như đồng thời cho cùng 1 yêu cầu chia tiền, đảm bảo tổng tiền thu về đúng và không ai bị trừ tiền 2 lần cho phần của mình.

**Yêu cầu cụ thể:**
- Mỗi phần chia (theo từng người trong nhóm) phải có 1 dòng dữ liệu riêng với trạng thái độc lập (đã trả/chưa trả), và việc 1 người bấm "Thanh toán phần của tôi" chỉ được cập nhật đúng dòng của người đó — dùng khóa chính hoặc unique constraint theo (bill_id, user_id) để tránh 2 request đồng thời của cùng người tạo 2 lần thanh toán cho cùng 1 phần.
- Mô tả cụ thể: người A bấm thanh toán phần của mình 2 lần liên tiếp do app bị lag (không rõ lần đầu đã thành công), yêu cầu idempotency theo (bill_id, user_id, request_id) để lần thứ 2 không trừ tiền thêm.
- Khi người tạo bill sửa lại số tiền hóa đơn tổng (ví dụ phát hiện tính sai) đúng lúc có người đang thanh toán phần cũ, quy định: các phần đã thanh toán theo số tiền cũ vẫn giữ nguyên (không tự động thu thêm/hoàn lại), chỉ các phần chưa thanh toán được cập nhật theo số tiền mới — tránh tình trạng vừa sửa vừa có người đang trả gây mất đồng bộ giữa số tiền hiển thị và số tiền thực đã trừ.
- Người tạo bill cần thấy được tiến độ "đã có bao nhiêu người trả, còn thiếu ai" chính xác real-time dù nhiều người thanh toán gần như cùng lúc — không dựa vào việc tính lại tổng từ đầu mỗi lần hiển thị mà nên cập nhật atomic 1 bộ đếm/tổng khi mỗi phần được đánh dấu đã trả.
- Xử lý trường hợp 1 người trong nhóm bị xóa khỏi bill (do rời nhóm) đúng lúc họ đang bấm thanh toán phần của mình — quy định rõ giao dịch nào có trước theo thời điểm transaction thực sự bắt đầu, không cho phép vừa xóa phần vừa nhận tiền thanh toán cho phần đã xóa.

---

## Chuyển tiền quốc tế qua nhiều chặng trung gian (correspondent banking)

**Repository:** `p2p-transfer-international-correspondent-banking`

**Hệ thống:** Một dịch vụ chuyển tiền quốc tế (kiểu remittance) chuyển tiền từ người gửi ở nước A tới người nhận ở nước B, phải đi qua 1-2 ngân hàng trung gian và quy đổi tiền tệ.

**Vai trò của flow:** Flow phải trừ tiền người gửi theo tiền tệ nước A, quy đổi và ghi có cho người nhận theo tiền tệ nước B, với callback xác nhận từng chặng trung gian có thể đến rất trễ và không đảm bảo tất cả chặng đều thành công.

**Yêu cầu cụ thể:**
- Toàn bộ hành trình giao dịch (từng chặng qua ngân hàng trung gian) phải được mô hình hóa như 1 saga với trạng thái rõ ràng cho mỗi bước, mỗi bước hoàn tất mới chuyển sang bước kế — không coi giao dịch "thành công" cho tới khi chặng cuối cùng (ghi có cho người nhận) xác nhận xong.
- Mô tả cụ thể: chặng 1 (trừ tiền người gửi, quy đổi) thành công, nhưng chặng 2 (chuyển tới ngân hàng trung gian) bị từ chối sau nhiều giờ — phải có bước hoàn tiền (compensating transaction) trả lại đúng số tiền gốc cho người gửi theo tỷ giá đã áp dụng ban đầu, không phải tỷ giá hiện tại lúc hoàn tiền.
- Nếu 2 callback xác nhận từ 2 chặng trung gian khác nhau đến gần như đồng thời nhưng báo cáo mâu thuẫn nhau về cùng 1 giao dịch (một báo thành công, một báo timeout không rõ), quy định quy trình xử lý: tạm giữ trạng thái "cần điều tra thủ công", không tự động coi là thành công hay thất bại khi có mâu thuẫn.
- Tỷ giá quy đổi phải được khóa tại thời điểm bắt đầu giao dịch và áp dụng nhất quán cho toàn bộ các chặng, dù chặng cuối hoàn tất sau nhiều ngày (trường hợp xử lý chậm) — không để tỷ giá trôi theo thời gian xử lý thực tế.
- Có cơ chế theo dõi SLA cho mỗi chặng (cảnh báo nếu 1 giao dịch ở 1 chặng quá X giờ chưa có xác nhận) để chủ động phát hiện giao dịch bị "kẹt" trước khi khách hàng phải khiếu nại.

---

## Tặng tiền/quà (gifting) trong livestream với hàng nghìn lượt tặng cùng lúc

**Repository:** `p2p-transfer-livestream-gifting`

**Hệ thống:** Một platform livestream cho phép người xem tặng "quà" (mua bằng tiền/coin trong ví) cho streamer, quà được quy đổi thành tiền cho streamer nhận sau khi trừ phí platform.

**Vai trò của flow:** Flow phải xử lý hàng nghìn lượt tặng quà nhỏ từ nhiều người xem khác nhau gửi tới cùng 1 streamer trong khoảng thời gian rất ngắn (ví dụ cao điểm livestream), cộng dồn đúng vào số dư chờ nhận của streamer mà không mất giao dịch nào.

**Yêu cầu cụ thể:**
- Mỗi lượt tặng quà (trừ coin người xem, cộng dồn cho streamer) phải atomic ở tầng DB (`UPDATE balance = balance - amount WHERE balance >= amount` cho người xem, và tăng dồn quỹ tạm cho streamer bằng increment atomic, không phải đọc-tính-ghi), để đảm bảo dưới tải cao (nhiều lượt tặng/giây) không mất hoặc trùng giao dịch.
- Mô tả cụ thể: 1 người xem tặng liên tiếp 10 lượt quà nhỏ trong 2 giây bằng cách bấm nút nhanh — mỗi lượt tặng phải được xử lý độc lập và đúng số lần đã bấm (không gộp nhầm/không mất lượt do xử lý song song race trên cùng dòng số dư người xem), yêu cầu queue xử lý tuần tự hoặc lock ngắn cho mỗi lượt.
- Việc cộng dồn quỹ nhận được cho streamer nên dùng bộ đếm atomic tách riêng khỏi việc trừ tiền người xem (2 bước trong cùng transaction, hoặc pattern outbox nếu cần xử lý bất đồng bộ), để tốc độ trừ tiền người xem (rất nhanh, số lượng lớn) không bị nghẽn bởi việc tổng hợp báo cáo cho streamer.
- Khi streamer rút tiền quỹ nhận được về ví chính, phải dựa trên số liệu đã "chốt" (settled, sau khi trừ phí platform) không phải số liệu đang đếm real-time trên màn hình live, tránh rút nhầm số tiền đang được cộng dồn giữa lúc rút.
- Có cơ chế giới hạn tốc độ tặng quà theo user (rate limit hợp lý, ví dụ tối đa N lượt/giây) để chống bot spam quà giả (nếu có lỗ hổng khai thác coin), đồng thời không ảnh hưởng tới trải nghiệm tặng quà bình thường của người xem thật.
