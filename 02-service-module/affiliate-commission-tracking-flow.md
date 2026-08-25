# Affiliate commission tracking flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (e-commerce, SaaS B2B referral, nền tảng nội dung/media, đặt vé du lịch, mobile app attribution) để luyện việc theo dõi nguồn giới thiệu và tính hoa hồng chính xác, chống gian lận, xử lý được hoàn tiền/hủy đơn.

---

## Chương trình affiliate cho sàn e-commerce

**Repository:** `affiliate-ecommerce-program`

**Hệ thống:** Một sàn e-commerce có chương trình affiliate cho phép người ảnh hưởng (influencer) gắn link giới thiệu, nhận hoa hồng theo phần trăm giá trị đơn hàng của khách được giới thiệu.

**Vai trò của flow:** Flow này theo dõi hành trình từ lúc khách click vào link affiliate tới lúc đặt hàng thành công, gán đúng công cho affiliate, và tính hoa hồng chính xác kể cả khi có hoàn hàng.

**Yêu cầu cụ thể:**
- Khi khách click vào link affiliate, phải lưu lại thông tin gán nguồn (attribution) gắn với thiết bị/session của khách trong một khoảng thời gian xác định (cookie window, ví dụ 30 ngày), và đơn hàng đặt trong khoảng thời gian đó được tính công cho affiliate đó dù khách không mua ngay lần click đầu.
- Xử lý trường hợp khách click qua nhiều link affiliate khác nhau trước khi mua (ví dụ click link của affiliate A rồi vài ngày sau click link của affiliate B rồi mới mua) — định nghĩa rõ chính sách gán công (last-click, first-click, hay chia theo tỷ lệ) và áp dụng nhất quán.
- Hoa hồng chỉ được tính (và trả) sau khi đơn hàng qua giai đoạn có thể hoàn trả (ví dụ sau 15 ngày không có yêu cầu trả hàng) — nếu khách hoàn hàng/hủy đơn trước đó, hoa hồng tương ứng phải bị hủy hoặc trừ lại, không trả hoa hồng cho đơn hàng đã hoàn tiền.
- Phát hiện gian lận: một tài khoản tự click vào link affiliate của chính mình hoặc dùng nhiều tài khoản giả để tạo đơn hàng khống nhằm nhận hoa hồng — cần đối chiếu các tín hiệu như IP/thiết bị trùng giữa tài khoản affiliate và tài khoản mua hàng.
- Cung cấp cho affiliate dashboard theo thời gian gần thực về số click, số đơn hàng được gán công, trạng thái hoa hồng (đang chờ/đã xác nhận/đã hủy) để họ tự theo dõi mà không cần liên hệ hỏi thủ công.

---

## Chương trình giới thiệu khách hàng cho SaaS B2B

**Repository:** `affiliate-b2b-saas-referral`

**Hệ thống:** Một nền tảng SaaS B2B có chương trình "giới thiệu bạn nhận thưởng" — khách hàng hiện tại giới thiệu công ty khác đăng ký dùng dịch vụ, nhận hoa hồng theo giá trị hợp đồng subscription dài hạn của công ty được giới thiệu.

**Vai trò của flow:** Khác với e-commerce (đơn hàng một lần), ở đây hoa hồng phải theo dõi một mối quan hệ hợp đồng kéo dài nhiều tháng/năm, với khả năng subscription bị hủy hoặc thay đổi giá trị theo thời gian.

**Yêu cầu cụ thể:**
- Khi công ty được giới thiệu đăng ký thành công, phải xác định đúng affiliate/người giới thiệu gắn với deal đó (qua mã giới thiệu hoặc link duy nhất), và lưu lại mối liên kết này cho toàn bộ vòng đời hợp đồng, không chỉ tính một lần tại thời điểm ký hợp đồng.
- Với mô hình hoa hồng định kỳ (recurring commission — trả hoa hồng theo mỗi kỳ khách hàng được giới thiệu tiếp tục thanh toán), phải tự động tính lại hoa hồng mỗi kỳ dựa trên giá trị subscription hiện tại (có thể đã upgrade/downgrade), không dùng cố định giá trị ban đầu.
- Khi công ty được giới thiệu hủy subscription hoặc không thanh toán được (thẻ hết hạn), phải ngừng tính hoa hồng từ kỳ đó và có chính sách rõ ràng về các khoản hoa hồng đã trả trước (có bị thu hồi hay không) theo hợp đồng với affiliate.
- Xử lý trường hợp một affiliate là chính sale nội bộ của công ty (không phải bên ngoài) muốn "tự giới thiệu" khách hàng mình đang chăm sóc để hưởng thêm hoa hồng — cần có business rule rõ ràng phân biệt affiliate hợp lệ với xung đột lợi ích nội bộ.
- Đảm bảo affiliate có thể xem được báo cáo hoa hồng dự kiến theo từng kỳ tương lai (dựa trên các hợp đồng đang active của khách được giới thiệu) để họ dự đoán được thu nhập, không chỉ xem hoa hồng đã trả trong quá khứ.

---

## Hoa hồng cho creator chia sẻ nội dung trên nền tảng media

**Repository:** `affiliate-media-creator-commission`

**Hệ thống:** Một nền tảng nội dung (giống blog/podcast platform) cho phép creator chia sẻ link tới nội dung trả phí của người khác, nhận hoa hồng khi người xem qua link đó đăng ký mua gói xem nội dung đó.

**Vai trò của flow:** Flow phải theo dõi được việc một creator giới thiệu nội dung của một creator khác (không phải của chính họ), và chia hoa hồng ba bên: nền tảng, creator gốc tạo nội dung, và creator giới thiệu.

**Yêu cầu cụ thể:**
- Khi có một giao dịch mua nội dung qua link giới thiệu, hệ thống phải phân chia đúng ba phần: phần nền tảng giữ lại, phần trả cho creator sở hữu nội dung, và phần hoa hồng cho creator đã giới thiệu — tổng ba phần phải luôn bằng đúng 100% giá trị giao dịch, có test kiểm tra không rò rỉ/thừa thiếu do lỗi làm tròn.
- Xử lý trường hợp một người xem tới nội dung qua nhiều lớp giới thiệu (ví dụ creator A giới thiệu link, người xem chia sẻ tiếp cho người khác qua đúng link đó) — chỉ creator gốc trong link mới được tính công, không tự nhân bản hoa hồng qua nhiều lớp chia sẻ trừ khi có chương trình multi-level được thiết kế riêng.
- Nếu người xem yêu cầu hoàn tiền cho nội dung đã mua (theo chính sách hoàn tiền của nền tảng), phải thu hồi đúng phần hoa hồng đã trả cho cả creator sở hữu nội dung và creator giới thiệu, không chỉ thu hồi một bên.
- Đảm bảo creator giới thiệu không thể nhận hoa hồng khi giới thiệu chính nội dung của mình qua một tài khoản phụ đóng vai "người xem" (self-referral gian lận) — cần chặn theo quan hệ chủ sở hữu nội dung, không chỉ theo tài khoản giao dịch.
- Cung cấp báo cáo minh bạch cho cả hai loại creator (chủ nội dung và người giới thiệu) về nguồn gốc mỗi khoản thanh toán họ nhận được, phân biệt rõ đâu là doanh thu trực tiếp và đâu là hoa hồng giới thiệu.

---

## Affiliate cho đại lý du lịch giới thiệu khách đặt tour/vé

**Repository:** `affiliate-travel-agency-referral`

**Hệ thống:** Một nền tảng đặt tour du lịch cho phép các đại lý/blogger du lịch đăng ký làm affiliate, nhận hoa hồng theo từng booking hoàn tất, với đặc thù booking có thể bị hủy/đổi ngày nhiều lần trước ngày khởi hành thực tế.

**Vai trò của flow:** Vì khoảng thời gian từ lúc đặt tới lúc "chuyến đi thực sự diễn ra" (và không còn khả năng hủy) có thể kéo dài nhiều tháng, flow phải xử lý trạng thái hoa hồng "tạm tính" qua một quá trình dài.

**Yêu cầu cụ thể:**
- Hoa hồng chỉ được xác nhận chính thức (chuyển từ trạng thái tạm tính sang có thể thanh toán) sau khi qua mốc không thể hủy/hoàn tiền của booking (ví dụ sau ngày khởi hành hoặc sau chính sách hủy miễn phí đã hết hạn) — không thanh toán hoa hồng khi booking vẫn còn khả năng bị hủy.
- Xử lý trường hợp khách đổi ngày/đổi tour giữa chừng (vẫn thuộc cùng affiliate giới thiệu ban đầu) — hoa hồng phải được tính lại theo giá trị booking mới, không giữ nguyên giá trị booking cũ đã bị thay đổi.
- Nếu khách hủy booking sau khi đã qua giai đoạn hoàn tiền một phần (ví dụ chỉ được hoàn 50% do chính sách hủy trễ), hoa hồng phải được tính theo tỷ lệ tương ứng với số tiền nền tảng thực sự giữ lại, không tính trên giá trị booking gốc đã bị hoàn một phần.
- Đảm bảo với các booking theo nhóm (một khách đặt cho nhiều người, có thể có người trong nhóm hủy riêng lẻ) affiliate vẫn được tính hoa hồng đúng theo phần giá trị thực tế còn hiệu lực của booking đó.
- Thiết kế cơ chế thanh toán hoa hồng theo lô định kỳ (ví dụ hàng tháng) chỉ bao gồm các booking đã qua mốc xác nhận chính thức, kèm báo cáo chi tiết từng booking để affiliate đối soát, tránh tranh chấp về những booking chưa đủ điều kiện thanh toán.

---

## Attribution cài đặt app di động qua chiến dịch quảng cáo/affiliate

**Repository:** `affiliate-mobile-install-attribution`

**Hệ thống:** Một ứng dụng mobile chạy các chiến dịch quảng cáo qua nhiều network và affiliate khác nhau, cần xác định chính xác network/affiliate nào đã dẫn tới một lượt cài đặt app để tính hoa hồng.

**Vai trò của flow:** Khác với web (có thể dùng cookie), attribution trên mobile phải xử lý việc mất liên kết trực tiếp giữa click quảng cáo và hành động cài đặt app (chuyển qua app store), đòi hỏi kỹ thuật khác để nối lại hành trình.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế attribution phù hợp với đặc thù mobile (ví dụ deferred deep link, hoặc đối chiếu tín hiệu như thiết bị, thời điểm click gần nhất với thời điểm cài đặt trong một khung thời gian hợp lý) để gán đúng lượt cài đặt cho network/affiliate đã dẫn tới nó.
- Xử lý trường hợp một người dùng click vào nhiều quảng cáo từ các network khác nhau trước khi cài app (ví dụ click quảng cáo network A trên Facebook rồi vài giờ sau click quảng cáo network B trên một app khác) — cần chính sách gán công rõ ràng (thường last-click trong cửa sổ thời gian ngắn) và nhất quán giữa các network để tránh tranh chấp giữa các đối tác.
- Chỉ tính hoa hồng cho các lượt cài đặt dẫn tới hành động có giá trị thực (ví dụ hoàn tất đăng ký, thực hiện giao dịch đầu tiên) theo mô hình CPA, không tính hoa hồng chỉ vì có lượt cài đặt (nếu chính sách hợp tác quy định như vậy) — phải theo dõi được toàn bộ funnel sau cài đặt, không chỉ event cài đặt.
- Phát hiện gian lận attribution phổ biến trên mobile (click flooding — network gửi hàng loạt click giả để "đón" các lượt cài đặt tự nhiên, hoặc click injection giữa lúc app đang cài đặt) bằng cách đối chiếu tỷ lệ click-to-install bất thường và khoảng thời gian giữa click và install quá ngắn để hợp lý.
- Cung cấp báo cáo đối soát cho mỗi network/affiliate về số lượt cài đặt được gán công và hoa hồng tương ứng, có khả năng giải trình khi một network tranh chấp về số liệu do dùng phương pháp attribution khác với nền tảng.
