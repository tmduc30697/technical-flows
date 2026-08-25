# Wallet top-up/withdraw flow (bao gồm crypto) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại ví điện tử (ví ngân hàng số, ví game, ví sàn giao dịch crypto, ví thanh toán trong app gọi xe) để luyện xử lý nạp/rút tiền chính xác dưới các thao tác đồng thời, bao gồm đặc thù của rút crypto trên blockchain (không thể rollback).

---

## Nạp/rút tiền đồng thời trên ví ngân hàng số

**Repository:** `wallet-digital-bank-concurrent-topup-withdraw`

**Hệ thống:** Một app ngân hàng số có ví nội bộ, người dùng có thể nạp tiền từ tài khoản ngân hàng liên kết và rút tiền về tài khoản đó bất cứ lúc nào.

**Vai trò của flow:** Flow phải đảm bảo số dư ví luôn chính xác khi có nhiều lệnh nạp/rút xảy ra gần như đồng thời trên cùng 1 tài khoản, kể cả khi user thao tác từ nhiều thiết bị/tab cùng lúc.

**Yêu cầu cụ thể:**
- Mọi thay đổi số dư phải qua `SELECT ... FOR UPDATE` trên dòng số dư của user trước khi cộng/trừ, không dùng đọc-tính-ghi ở tầng application; viết test 2 lệnh rút 600k đồng thời trên tài khoản có 1 triệu, chỉ đúng 1 lệnh được thực hiện, lệnh kia phải báo "không đủ số dư" (không để cả 2 đều thành công dẫn tới số dư âm).
- Lệnh rút tiền phải kiểm tra số dư khả dụng ngay tại thời điểm trừ tiền trong transaction (không dùng số dư đã đọc từ trước khi user bấm xác nhận, vì có thể đã có giao dịch khác thay đổi số dư trong khoảng đó).
- Khi user bấm rút tiền 2 lần liên tiếp do mạng lag tưởng chưa gửi được (double-submit), phải có idempotency key theo request để chỉ 1 lệnh rút được thực hiện, lệnh trùng trả về kết quả của lệnh đầu, không trừ tiền 2 lần.
- Nạp tiền từ ngân hàng liên kết (qua webhook callback) và rút tiền do user chủ động bấm có thể đến gần như đồng thời — cả 2 phải cùng đi qua đúng 1 lock trên số dư user đó, không có "đường tắt" nào (ví dụ 1 luồng dùng cache riêng) bỏ qua lock.
- Có bảng ledger ghi lại từng giao dịch tăng/giảm số dư (không chỉ lưu số dư cuối), để có thể tái tính và kiểm tra số dư hiện tại bất cứ lúc nào từ lịch sử giao dịch, phục vụ đối soát khi có nghi ngờ sai lệch.

---

## Ví trong game có nạp bằng thẻ và rút thưởng bằng tiền thật

**Repository:** `wallet-game-cash-out`

**Hệ thống:** Một platform game casual cho phép người chơi nạp tiền mua vật phẩm và có chương trình đổi điểm thưởng thành tiền thật rút về ví ngân hàng, có giới hạn rút tối thiểu và giới hạn số lần rút mỗi ngày.

**Vai trò của flow:** Flow rút thưởng phải kiểm tra đúng điều kiện (đủ điểm, chưa vượt giới hạn số lần/ngày) một cách atomic, tránh việc user khai thác race condition để rút vượt điểm thưởng thực có hoặc vượt giới hạn số lần.

**Yêu cầu cụ thể:**
- Mô tả cụ thể kịch bản khai thác: user có đúng 100 điểm thưởng (đủ cho 1 lần rút), gửi 5 request rút cùng lúc từ 5 tab/thiết bị khác nhau — yêu cầu chỉ đúng 1 request thành công, 4 request còn lại phải bị chặn bởi transaction atomic kiểm tra-và-trừ điểm (không phải kiểm tra điểm rồi trừ ở 2 bước tách biệt).
- Giới hạn số lần rút/ngày (ví dụ tối đa 3 lần) phải được đếm và validate trong cùng transaction với việc trừ điểm, dùng lock trên dòng đếm số lần rút của user trong ngày đó, tránh 3 request song song đều đọc thấy "đã rút 2 lần, còn được 1 lần" và cả 3 đều pass qua kiểm tra.
- Khi hệ thống thanh toán rút tiền thật (chuyển sang ngân hàng) bị lỗi/timeout sau khi đã trừ điểm thưởng trong game, phải hoàn lại điểm thưởng ngay và atomic, không để user mất điểm mà không nhận được tiền và không có cách nào tự phát hiện.
- Cho phép admin/CS hoàn điểm thưởng thủ công cho user trong trường hợp tranh chấp, nhưng hành động này phải đi qua đúng lock/transaction giống luồng tự động, không dùng câu update trực tiếp bỏ qua kiểm tra tính nhất quán.
- Có giới hạn tốc độ (rate limit) ở tầng API cho hành động rút thưởng theo user_id, để giảm số lượng request đồng thời phải xử lý ở tầng transaction, dù transaction đã atomic vẫn nên chặn sớm các request thừa.

---

## Nạp/rút crypto trên sàn giao dịch (không thể rollback on-chain)

**Repository:** `wallet-crypto-exchange-onchain`

**Hệ thống:** Một sàn giao dịch crypto cho phép user nạp coin từ địa chỉ ví ngoài vào và rút coin từ sàn về địa chỉ ví ngoài, giao dịch on-chain có độ trễ xác nhận (confirmation) và không thể hoàn tác sau khi đã broadcast.

**Vai trò của flow:** Flow rút coin phải trừ số dư nội bộ atomic trước khi gửi lệnh rút lên blockchain, và vì giao dịch on-chain không thể rollback, mọi bước kiểm tra (đủ số dư, giới hạn rút, xác thực 2 lớp) phải hoàn tất chắc chắn trước khi broadcast, không có cơ hội "sửa sai" sau đó.

**Yêu cầu cụ thể:**
- Vì không thể rollback sau khi broadcast lên blockchain, transaction trừ số dư nội bộ và tạo lệnh rút phải hoàn tất atomic và pass toàn bộ validation (KYC, giới hạn rút, xác thực OTP/2FA) trước bước broadcast — quy định rõ thứ tự: trừ số dư nội bộ (đặt trạng thái "pending broadcast") trước, broadcast sau, để nếu broadcast thất bại (lỗi node, phí gas không đủ) có thể hoàn lại số dư nội bộ một cách an toàn (chưa có gì xảy ra on-chain).
- Mô tả cụ thể race: user gửi 2 lệnh rút toàn bộ số dư (100 USDT) gần như đồng thời từ 2 thiết bị — chỉ 1 lệnh được trừ số dư và broadcast thành công, lệnh thứ 2 phải bị chặn ngay ở bước trừ số dư atomic (kiểm tra-và-trừ trong 1 câu update, không tách 2 bước), không được để cả 2 lệnh cùng broadcast lên blockchain.
- Khi node blockchain trả về "đã gửi nhưng chưa rõ kết quả" (timeout khi chờ xác nhận broadcast), hệ thống KHÔNG được tự động broadcast lại giao dịch giống y hệt (rủi ro double-spend on-chain thật) — phải có cơ chế kiểm tra trạng thái giao dịch trên chain qua transaction hash trước, chỉ retry nếu chắc chắn giao dịch gốc chưa được ghi nhận.
- Với nạp coin vào sàn, chỉ ghi nhận số dư cho user sau khi giao dịch on-chain đạt đủ số block xác nhận theo quy định của từng loại coin (ví dụ 6 confirmation cho Bitcoin) để chống rủi ro đảo chuỗi (chain reorg) — mô tả cụ thể việc job theo dõi block phải xử lý đúng khi 1 giao dịch đã tưởng đủ confirmation nhưng sau đó block chứa nó bị đảo (phải rollback lại số dư đã ghi nhận tạm, nếu có).
- Giới hạn rút hàng ngày theo user phải được tính atomic dựa trên tổng các lệnh rút đã "pending broadcast" hoặc đã hoàn tất trong ngày, tránh user gửi nhiều lệnh rút nhỏ liên tiếp vượt giới hạn do mỗi lệnh được kiểm tra dựa trên số liệu đã lỗi thời từ trước các lệnh trước đó xử lý xong.

---

## Ví thanh toán trong app gọi xe cho tài xế nhận tiền và rút về ngân hàng

**Repository:** `wallet-ride-hailing-driver-payout`

**Hệ thống:** Một app gọi xe có ví nội bộ cho tài xế, tiền cước sau mỗi chuyến được cộng vào ví tài xế, tài xế có thể rút tiền về ngân hàng bất cứ lúc nào, kể cả khi đang có chuyến đi đang chạy.

**Vai trò của flow:** Flow phải xử lý đúng khi tiền cước từ chuyến đi vừa hoàn thành đang được cộng vào ví đúng lúc tài xế bấm rút toàn bộ số dư, tránh rút thiếu hoặc rút vượt số dư thật.

**Yêu cầu cụ thể:**
- Mô tả cụ thể race: chuyến đi kết thúc, hệ thống đang cộng 150k cước vào ví tài xế đúng lúc tài xế bấm "Rút toàn bộ số dư" (đang thấy số dư cũ 500k trên app do cache/hiển thị chưa cập nhật) — transaction rút tiền phải lock dòng số dư và đọc giá trị mới nhất tại thời điểm rút (650k nếu cước đã cộng xong, hoặc phải chờ/serialize đúng thứ tự nếu 2 transaction cùng chạm dòng đó), không rút theo số hiển thị cũ trên UI.
- Khi tài xế đang có chuyến đi chưa hoàn thành (tiền cước dự kiến chưa cộng vào ví), lệnh rút toàn bộ số dư chỉ được tính trên số dư đã thực sự ghi nhận (settled), không bao gồm tiền cước "đang treo" của chuyến chưa kết thúc — quy định rõ ranh giới giữa số dư khả dụng để rút và số dư đang chờ xử lý.
- Nếu 2 chuyến đi của tài xế kết thúc gần như đồng thời (tài xế chạy 2 tab hoặc do đồng bộ hệ thống), cả 2 giao dịch cộng cước phải cộng dồn atomic đúng cả 2 khoản, không mất giao dịch nào do ghi đè lẫn nhau.
- Lệnh rút tiền về ngân hàng phải có idempotency key để nếu tài xế bấm rút nhiều lần do app lag, chỉ 1 lệnh rút thật được gửi đi, tránh rút tiền về ngân hàng nhiều lần cho cùng 1 yêu cầu.
- Có cơ chế đối soát cuối ngày giữa tổng tiền cước đã cộng vào ví tất cả tài xế và tổng doanh thu cước ghi nhận từ hệ thống chuyến đi, phát hiện chênh lệch nếu có lỗi cộng thiếu/cộng trùng.

---

## Ví đa tiền tệ cho phép chuyển đổi nội bộ giữa các loại tiền/coin

**Repository:** `wallet-multi-currency-internal-conversion`

**Hệ thống:** Một ví điện tử cho phép user giữ nhiều loại tiền/coin khác nhau (VND, USD, USDT) và tự chuyển đổi qua lại trong ví, kèm nạp/rút mỗi loại riêng.

**Vai trò của flow:** Flow chuyển đổi nội bộ giữa các loại tiền phải atomic trên cả 2 số dư (trừ loại A, cộng loại B theo tỷ giá) và không được xung đột với các lệnh nạp/rút đang diễn ra song song trên cùng loại tiền đó.

**Yêu cầu cụ thể:**
- Chuyển đổi 100 USD sang VND phải là 1 transaction duy nhất lock cả 2 dòng số dư (USD và VND) của user theo thứ tự cố định (ví dụ theo mã tiền tệ sắp theo alphabet) để tránh deadlock khi có nhiều lệnh chuyển đổi ngược hướng (USD→VND và VND→USD) chạy song song cho các user khác nhau đụng chung cơ chế lock.
- Mô tả cụ thể race: user có 100 USD, gửi đồng thời 1 lệnh chuyển đổi 100 USD sang VND và 1 lệnh rút 100 USD về ngân hàng ngoại tệ — chỉ đúng 1 trong 2 được thực hiện (dựa trên số dư khả dụng thực tại thời điểm mỗi transaction chạy, atomic), lệnh sau khi số dư đã hết phải báo lỗi rõ ràng, không cho phép cả 2 đều trừ thành công gây số dư USD âm.
- Tỷ giá chuyển đổi phải được khóa (snapshot) tại thời điểm user xác nhận giao dịch, và giao dịch phải hoàn tất trong 1 khung thời gian ngắn (ví dụ 10 giây) kể từ khi khóa tỷ giá — nếu vượt thời gian này do lỗi hệ thống, phải hủy giao dịch và yêu cầu user thực hiện lại với tỷ giá mới, không dùng tỷ giá cũ đã hết hạn.
- Khi nạp tiền (webhook báo nạp USD thành công) đến đúng lúc user đang thực hiện lệnh chuyển đổi toàn bộ USD hiện có sang VND, quy định rõ khoản nạp mới có được tính vào số dư khả dụng để chuyển đổi hay không tùy vào việc webhook được xử lý xong trước hay sau khi transaction chuyển đổi bắt đầu lock dòng số dư đó.
- Có ledger ghi nhận riêng từng loại tiền tệ cho mọi giao dịch (nạp, rút, chuyển đổi) để có thể tái tính số dư mỗi loại tiền độc lập, phục vụ đối soát khi user khiếu nại số dư một loại tiền cụ thể bị sai.
