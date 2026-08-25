# Auction/Bidding flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống đấu giá (marketplace đồ hiếm, đấu giá quảng cáo real-time, đấu giá thanh lý B2B, đấu giá NFT, đấu giá ngược cho gói thầu) để luyện xử lý bid đồng thời, xác định người thắng đúng, và xử lý các thao tác cạnh tranh sát giờ chốt.

---

## Đấu giá đồ sưu tầm hiếm với nhiều người bid sát giờ chốt (sniping)

**Repository:** `auction-bidding-collectibles-sniping`

**Hệ thống:** Một sàn đấu giá đồ sưu tầm (tranh, đồ cổ), mỗi phiên đấu giá có thời gian kết thúc cố định, người bid cao nhất khi hết giờ thắng.

**Vai trò của flow:** Flow xử lý bid phải xác định đúng bid nào hợp lệ và cao nhất khi có nhiều người bid gần sát giờ chốt (vài trăm milliseconds trước khi kết thúc), tránh race condition khiến 2 bid gần như đồng thời đều được chấp nhận là "cao nhất".

**Yêu cầu cụ thể:**
- Mỗi bid mới phải được xử lý trong transaction có `SELECT ... FOR UPDATE` trên dòng phiên đấu giá để đọc giá cao nhất hiện tại và so sánh, không cho phép 2 bid đọc cùng lúc giá cao nhất cũ rồi cả 2 đều tưởng mình là bid cao nhất mới.
- Mô tả cụ thể: 2 người bid 1.000.000đ và 1.050.000đ gửi request cách nhau 50ms ngay trước giờ chốt (còn 1 giây), yêu cầu xử lý tuần tự theo đúng thứ tự server nhận được (không theo thứ tự client gửi, vì độ trễ mạng khác nhau), và giá cao nhất cuối cùng phải phản ánh đúng bid có giá cao hơn hợp lệ tại thời điểm đó, kèm giá thấp hơn bị từ chối ngay với lý do "đã có giá cao hơn".
- Quy định rõ ràng thời điểm nào được coi là "giờ chốt": bid đến server trước mốc kết thúc (theo thời gian server, không phải thời gian client) dù chỉ 1 milisecond vẫn hợp lệ, bid đến sau bị từ chối — không dùng thời gian gửi từ client vì có thể bị chỉnh sai hoặc lag mạng.
- Cân nhắc và mô tả rõ 1 trong 2 cơ chế chống "snipe" (bid sát giờ chốt để không ai kịp bid tiếp): tự động gia hạn thêm vài phút nếu có bid trong khoảng thời gian cuối, hoặc giữ nguyên chính sách chốt cứng nhưng phải xử lý atomic tuyệt đối để không có tranh cãi ai bid trước.
- Viết test giả lập 100 bid đồng thời với các mức giá ngẫu nhiên gửi trong cùng 1 khoảng thời gian rất ngắn, assert giá thắng cuối cùng đúng là giá cao nhất hợp lệ trong tất cả các bid, và không có 2 bid nào cùng được đánh dấu là "đang dẫn đầu" tại 1 thời điểm.

---

## Đấu giá quảng cáo real-time (RTB) trong hệ thống ad exchange

**Repository:** `auction-bidding-adtech-rtb`

**Hệ thống:** Một ad exchange nhận request đấu giá quảng cáo từ nhiều nhà quảng cáo trong thời gian rất ngắn (dưới 100ms) cho mỗi lượt hiển thị quảng cáo, chọn ra bid cao nhất để hiển thị.

**Vai trò của flow:** Flow phải thu thập bid từ nhiều nhà quảng cáo gửi song song trong khung thời gian cực ngắn, chọn đúng bid cao nhất và xử lý các bid đến muộn (timeout) mà không làm chậm việc hiển thị quảng cáo.

**Yêu cầu cụ thể:**
- Quy định rõ deadline cứng cho mỗi phiên đấu giá (ví dụ 80ms), mọi bid đến sau deadline phải bị loại khỏi phiên đó dù giá cao hơn, không chờ thêm dù chỉ vài milisecond (đảm bảo trải nghiệm hiển thị trang không bị chậm vì chờ 1 nhà quảng cáo).
- Mô tả cụ thể: 2 nhà quảng cáo gửi bid với giá bằng nhau (tie) trong cùng phiên — yêu cầu quy tắc phá vỡ hòa (tie-breaking) rõ ràng và xác định trước (ví dụ ai gửi bid trước theo timestamp server, hoặc random có seed cố định để tái lập được), không để việc chọn người thắng phụ thuộc vào thứ tự xử lý không xác định của hệ thống (non-deterministic).
- Việc thu thập bid từ nhiều nhà quảng cáo phải chạy song song (không tuần tự) để tổng thời gian đấu giá không cộng dồn theo số nhà quảng cáo, nhưng việc ghi nhận "ai là người thắng" cuối cùng vẫn phải là 1 quyết định atomic duy nhất sau khi deadline kết thúc hoặc tất cả bid đã về.
- Khi 1 nhà quảng cáo thắng phiên đấu giá nhưng ngay sau đó report "không thể serve creative" (lỗi kỹ thuật phía họ), phải có phương án dự phòng chọn bid cao thứ 2 (second-price hoặc runner-up) để hiển thị, trong 1 khoảng thời gian rất ngắn để không làm chậm trải nghiệm người dùng cuối.
- Có log chi tiết mọi bid nhận được trong mỗi phiên (kể cả bid bị loại do đến muộn) để phục vụ audit khi nhà quảng cáo khiếu nại về việc bid của họ không được tính.

---

## Đấu giá thanh lý hàng tồn kho B2B giữa các doanh nghiệp

**Repository:** `auction-bidding-b2b-liquidation`

**Hệ thống:** Một platform B2B cho phép doanh nghiệp đấu giá mua lô hàng thanh lý số lượng lớn từ nhà sản xuất, mỗi lô chỉ bán cho 1 người thắng duy nhất, giá trị giao dịch lớn nên cần xác nhận qua nhiều bước (không tự động chốt tức thời như đấu giá tiêu dùng).

**Vai trò của flow:** Flow bidding phải xử lý các bid có giá trị lớn với yêu cầu xác thực (ví dụ đặt cọc trước khi được phép bid) đồng thời đảm bảo tính đúng đắn khi nhiều doanh nghiệp bid gần giờ chốt.

**Yêu cầu cụ thể:**
- Trước khi được phép đặt bid, doanh nghiệp phải đặt cọc (escrow) một khoản tiền tối thiểu — quy định rõ việc đặt cọc và việc ghi nhận bid phải kiểm tra atomic (không cho bid nếu cọc chưa được xác nhận xử lý xong), tránh trường hợp bid được ghi nhận trước khi cọc thực sự hoàn tất.
- Mô tả cụ thể: 2 doanh nghiệp bid gần như đồng thời với giá cao hơn giá hiện tại, một đã đặt cọc đủ, một đang trong quá trình xử lý cọc (chưa xác nhận xong) — bid của doanh nghiệp chưa hoàn tất cọc phải bị từ chối/giữ ở trạng thái "chờ xác thực", không được tính là bid hợp lệ cho tới khi cọc xác nhận xong, và tại thời điểm đó phải re-check giá hiện tại (có thể đã bị người khác bid cao hơn trong lúc chờ).
- Khi phiên đấu giá kết thúc, người thắng phải được xác nhận qua bước riêng (không tự động chốt giao dịch ngay), và trong lúc chờ xác nhận, các cọc của người không thắng phải được giải phóng đúng và atomic, không giữ treo cọc của người thua vô thời hạn.
- Nếu người thắng từ chối hoàn tất giao dịch sau khi thắng (từ chối thanh toán số tiền lớn), phải có quy trình chuyển sang người bid cao thứ 2 trong khoảng thời gian quy định, và giải phóng cọc của người thắng ban đầu theo chính sách phạt cọc nếu có.
- Giá bid tối thiểu tăng theo bước giá cố định (ví dụ mỗi bid phải cao hơn giá hiện tại ít nhất 5%) phải được validate atomic ngay tại thời điểm ghi nhận bid dựa trên giá cao nhất thực tế tại thời điểm đó, không dựa vào giá đã hiển thị trên UI của người bid (có thể đã lỗi thời vài giây).

---

## Đấu giá NFT trên marketplace blockchain với giao dịch on-chain

**Repository:** `auction-bidding-nft-onchain`

**Hệ thống:** Một marketplace NFT cho phép đấu giá vật phẩm số, các bid được ghi nhận cả ở database nội bộ (off-chain, để UX nhanh) và cuối cùng giao dịch chuyển quyền sở hữu thực hiện on-chain khi phiên kết thúc.

**Vai trò của flow:** Flow phải đồng bộ đúng giữa trạng thái bid off-chain (nhanh, để hiển thị real-time) và giao dịch chốt on-chain (chậm, có phí gas và độ trễ xác nhận), xử lý đúng khi 2 nguồn này có thể tạm thời không khớp.

**Yêu cầu cụ thể:**
- Bid off-chain phải nhanh và atomic giống các hệ thống đấu giá thông thường (lock dòng phiên đấu giá, so sánh giá, ghi nhận), nhưng phải rõ ràng đây chỉ là "cam kết tạm" (off-chain commitment), giao dịch thật (chuyển token) chỉ xảy ra on-chain sau khi phiên kết thúc.
- Mô tả cụ thể: phiên đấu giá kết thúc off-chain xác định người thắng, nhưng giao dịch on-chain để hoàn tất chuyển NFT bị thất bại (ví dụ người thắng không đủ tiền on-chain thực tế lúc đó, hoặc phí gas biến động khiến giao dịch fail) — quy định quy trình fallback rõ ràng: chuyển sang bid cao thứ 2 hoặc hủy phiên và hoàn cọc, không để NFT bị "treo" ở trạng thái không rõ chủ.
- Vì giao dịch on-chain không thể rollback sau khi đã confirm, bước ký và gửi giao dịch chuyển NFT chỉ được thực hiện sau khi tất cả điều kiện off-chain đã chốt chắc chắn (phiên đã kết thúc, người thắng đã xác nhận, không còn tranh chấp) — không được thực hiện song song với việc vẫn đang nhận bid off-chain.
- Nếu có 2 bid off-chain gần như đồng thời sát giờ chốt trên cùng phiên, xử lý giống các hệ thống đấu giá thông thường (transaction atomic xác định thứ tự theo thời điểm server nhận), nhưng phải log rõ ràng bid nào được chọn và tại sao để giải quyết tranh chấp (khiếu nại phổ biến trong NFT do giá trị biến động nhanh).
- Có cơ chế hiển thị rõ cho người dùng 2 trạng thái riêng biệt: "đang dẫn đầu (off-chain)" và "đã hoàn tất on-chain" để tránh người dùng hiểu nhầm đã sở hữu NFT khi giao dịch on-chain chưa xác nhận xong.

---

## Đấu giá ngược (reverse auction) cho gói thầu cung cấp dịch vụ B2B

**Repository:** `auction-bidding-reverse-b2b-procurement`

**Hệ thống:** Một platform cho doanh nghiệp đăng gói thầu (ví dụ thuê dịch vụ vận chuyển), các nhà cung cấp bid giá thấp nhất để giành gói thầu (ngược với đấu giá thông thường, giá thấp nhất thắng).

**Vai trò của flow:** Flow bidding ngược phải xử lý đúng khi nhiều nhà cung cấp bid giá thấp gần như đồng thời, xác định đúng giá thấp nhất hợp lệ, và xử lý việc nhà cung cấp sửa/rút bid trước giờ chốt.

**Yêu cầu cụ thể:**
- Logic so sánh phải đảo ngược so với đấu giá thường (bid mới chỉ hợp lệ nếu THẤP hơn giá thấp nhất hiện tại), vẫn phải dùng lock atomic trên dòng gói thầu để tránh 2 bid giá thấp gần như đồng thời đều được coi là "giá thấp nhất mới" cùng lúc.
- Mô tả cụ thể: nhà cung cấp A bid 100 triệu, sau đó muốn sửa xuống 95 triệu (thấp hơn để chắc thắng) đúng lúc nhà cung cấp B cũng vừa bid 96 triệu — transaction sửa bid của A phải re-check giá thấp nhất hiện tại (96 triệu của B) tại thời điểm sửa, không dựa vào giá A đã biết trước đó (100 triệu của chính A), để xác định đúng 95 triệu của A vẫn thắng B.
- Cho phép nhà cung cấp rút bid trước giờ chốt (nếu chính sách cho phép), nhưng phải xử lý atomic: nếu bid bị rút đúng lúc đang là giá thấp nhất, phải tính toán lại đúng giá thấp nhất mới từ các bid còn lại một cách nhất quán, không để hiển thị "giá thấp nhất" bị treo ở giá trị vừa rút trong khoảng thời gian ngắn.
- Vì gói thầu B2B thường có yêu cầu kỹ thuật kèm giá (không chỉ chọn giá thấp nhất tuyệt đối mà còn điểm đánh giá năng lực), quy định rõ bid mới có được coi là "cải thiện" hay không phải dựa trên tổng điểm/giá kết hợp, và việc tính điểm này phải được thực hiện atomic cùng lúc ghi nhận bid, tránh tình huống 2 bid có điểm tổng gần bằng nhau được xử lý không nhất quán do thứ tự tính toán khác nhau.
- Khi gói thầu hết giờ chốt, phải khóa (freeze) toàn bộ bid ngay tại đúng một mốc thời gian server xác định, thông báo kết quả cho tất cả nhà cung cấp tham gia đồng thời (không để 1 nhà cung cấp biết kết quả trước những người khác do xử lý callback không đồng bộ).
