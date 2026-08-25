# Cart abandonment recovery flow (kết hợp timing + background job) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống có giỏ hàng/luồng chuyển đổi bị bỏ giữa chừng (e-commerce, đăng ký subscription SaaS, đặt tour du lịch, mua bảo hiểm online, đặt hàng theo yêu cầu B2B) để luyện việc thiết kế background job phát hiện "bỏ giỏ", gửi nhắc nhở đúng lúc, và xử lý đúng khi user hoàn tất giao dịch ngay giữa lúc job đang chạy.

---

## Nhắc nhở giỏ hàng bỏ quên trên sàn e-commerce qua email/push

**Repository:** `cart-abandonment-ecommerce-recovery`

**Hệ thống:** Một sàn e-commerce có background job quét các giỏ hàng không hoạt động sau 1 khoảng thời gian (ví dụ 1 giờ) để gửi email/push nhắc khách hoàn tất mua hàng, kèm ưu đãi nhỏ nếu quay lại trong 24 giờ.

**Vai trò của flow:** Flow phải phát hiện đúng giỏ hàng "bị bỏ" (không phải khách đang xem tiếp trang khác), gửi nhắc nhở đúng lúc, và không gửi nhắc nhở cho giỏ hàng khách vừa hoàn tất mua ngay trước khi job kịp quét.

**Yêu cầu cụ thể:**
- Job quét giỏ hàng bỏ quên phải chạy theo batch định kỳ (ví dụ mỗi 15 phút) và chỉ lấy các giỏ hàng có `last_updated_at` cũ hơn ngưỡng quy định VÀ chưa được đánh dấu đã gửi nhắc nhở, dùng update atomic đánh dấu "đã xử lý" ngay khi chọn ra để job chạy lần kế tiếp không chọn trùng giỏ hàng đó lần nữa.
- Mô tả cụ thể race: job đang lấy danh sách giỏ hàng bỏ quên để gửi email, đúng lúc đó khách quay lại và hoàn tất thanh toán giỏ hàng đó — nếu job gửi email nhắc nhở ngay sau khi khách đã thanh toán xong (do đọc dữ liệu giỏ hàng từ trước khi thanh toán), phải có bước re-check trạng thái giỏ hàng ngay trước khi gửi (không gửi nếu giỏ đã chuyển thành order), tránh gửi email "bạn còn bỏ quên giỏ hàng" cho người vừa mua xong gây khó hiểu.
- Nếu khách mở lại giỏ hàng và tự thêm/xóa sản phẩm ngay trong lúc job đang xử lý batch chứa giỏ hàng đó, job phải dùng dữ liệu giỏ hàng tại thời điểm gửi thực tế (không dùng snapshot đã lấy từ đầu batch có thể đã lỗi thời), để nội dung email nhắc nhở đúng với giỏ hàng hiện tại.
- Không gửi nhắc nhở lần 2 cho cùng 1 giỏ hàng nếu đã gửi lần 1 chưa quá 1 khoảng thời gian tối thiểu (ví dụ 24 giờ), dù job chạy lại nhiều lần — dùng cờ trạng thái/timestamp lần gửi cuối, kiểm tra atomic trước khi gửi để tránh gửi trùng do nhiều worker chạy job song song không đồng bộ với nhau.
- Ưu đãi kèm nhắc nhở (mã giảm giá) chỉ được áp dụng nếu khách hoàn tất mua trong đúng khung thời gian quy định (ví dụ 24 giờ từ lúc gửi) — validate atomic thời điểm áp dụng mã giảm giá so với thời điểm gửi nhắc nhở, không cho phép áp dụng mã dù đã hết hạn do lỗi kiểm tra thời gian ở tầng application không nhất quán với DB.

---

## Nhắc nhở hoàn tất đăng ký subscription bị bỏ giữa chừng ở bước nhập thẻ

**Repository:** `cart-abandonment-subscription-card-entry`

**Hệ thống:** Một SaaS B2B có luồng đăng ký dùng thử/subscription, nhiều khách bỏ giữa chừng ở bước nhập thông tin thẻ thanh toán, cần background job gửi email nhắc hoàn tất đăng ký.

**Vai trò của flow:** Flow phải phát hiện đúng các phiên đăng ký bị bỏ ở bước thanh toán (không phải các bước trước đó chưa đủ thông tin để coi là "có ý định mua"), và không gửi nhắc nhở trùng lặp hoặc gửi sau khi khách đã tự hoàn tất đăng ký qua kênh khác.

**Yêu cầu cụ thể:**
- Chỉ coi là "bỏ giữa chừng" các phiên đã hoàn tất các bước trước bước thanh toán (ví dụ đã chọn gói, đã nhập thông tin công ty) và dừng lại đúng ở bước nhập thẻ trong khoảng thời gian tối thiểu (ví dụ 30 phút không hoạt động) — quy định rõ tiêu chí phân biệt với người mới vào trang chưa có ý định thật.
- Mô tả cụ thể: khách bỏ giữa chừng ở bước thanh toán trên desktop, sau đó tự quay lại hoàn tất đăng ký trên điện thoại trước khi job kịp gửi email nhắc nhở — job phải re-check trạng thái đăng ký (đã hoàn tất hay chưa) ngay trước khi gửi, không dựa vào snapshot đã xác định "bỏ giữa chừng" từ đầu batch, để tránh gửi nhắc nhở cho subscription đã active.
- Nếu khách có nhiều phiên đăng ký bị bỏ (thử nhiều lần trong ngày, mỗi lần bỏ ở bước khác nhau), job phải gộp và chỉ gửi 1 email nhắc nhở dựa trên phiên gần nhất, không gửi nhiều email riêng cho từng phiên bỏ giữa chừng gây spam.
- Background job và luồng thanh toán trực tiếp của khách (nếu khách click vào link trong email nhắc nhở để hoàn tất) phải cùng đi qua đúng 1 lớp kiểm tra idempotency dựa trên session đăng ký gốc, để nếu khách vừa hoàn tất qua web vừa click link email gần như đồng thời, không tạo ra 2 subscription trùng cho cùng 1 công ty.
- Có giới hạn số lần gửi nhắc nhở tối đa cho 1 phiên bỏ giữa chừng (ví dụ tối đa 2 lần trong 7 ngày) để tránh làm phiền khách quá mức, và dừng hẳn nếu khách bấm "không quan tâm"/unsubscribe từ email nhắc nhở.

---

## Nhắc nhở hoàn tất đặt tour du lịch bị bỏ ở bước chọn ngày khởi hành

**Repository:** `cart-abandonment-travel-tour-booking`

**Hệ thống:** Một nền tảng đặt tour du lịch, khách thường xem nhiều tour, chọn ngày khởi hành rồi rời trang để "suy nghĩ thêm", cần job nhắc nhở kèm cảnh báo nếu số chỗ tour đó đang giảm nhanh.

**Vai trò của flow:** Flow phải gửi nhắc nhở kèm thông tin tồn kho chỗ trống thực tế tại thời điểm gửi (không phải tại thời điểm khách xem), tránh tạo cảm giác khẩn cấp giả nếu tồn kho đã thay đổi, đồng thời không nhắc nhở khách đã đặt tour đó qua kênh khác (gọi điện tổng đài).

**Yêu cầu cụ thể:**
- Job phải đọc lại số chỗ trống thực tế của tour đó ngay tại thời điểm chuẩn bị gửi email (không dùng số liệu đã lưu từ lúc khách xem/rời trang), để tránh gửi thông tin "chỉ còn 2 chỗ" trong khi thực tế đã bán hết hoặc đã có thêm chỗ (do khách khác hủy).
- Mô tả cụ thể: khách xem tour A và rời trang, sau đó gọi tổng đài đặt tour A qua nhân viên tư vấn (không qua hệ thống tự động) trước khi job kịp gửi email nhắc nhở — hệ thống phải có cách đánh dấu "khách đã đặt" dù đặt qua kênh khác (tổng đài ghi nhận vào cùng hệ thống), và job phải re-check trạng thái này ngay trước khi gửi, không gửi nhắc nhở cho tour khách đã đặt thành công.
- Nếu tour đó đã hết chỗ hoàn toàn giữa lúc khách xem và lúc job chuẩn bị gửi nhắc nhở, quy định rõ hành vi: không gửi email dạng "còn chỗ, đặt ngay" (sai sự thật), có thể gửi email khác gợi ý tour tương tự còn chỗ, hoặc không gửi gì cả tùy chính sách.
- Khách xem nhiều tour khác nhau trong session, mỗi tour có thể có trạng thái tồn kho khác nhau tại thời điểm gửi nhắc nhở — job phải xử lý độc lập cho từng tour, không gộp chung 1 email với thông tin tồn kho đã lỗi thời của tour đã hết chỗ lẫn với tour còn chỗ.
- Đảm bảo việc đọc tồn kho để hiển thị trong email nhắc nhở không gây side-effect nào lên tồn kho thực (chỉ đọc, không giữ/reserve chỗ chỉ vì gửi email nhắc nhở), tránh nhầm lẫn giữa hành động "xem thông tin để nhắc nhở" và hành động "đặt giữ chỗ" thực sự.

---

## Nhắc nhở hoàn tất mua bảo hiểm online bị bỏ ở bước khai báo sức khỏe

**Repository:** `cart-abandonment-insurance-health-declaration`

**Hệ thống:** Một nền tảng bán bảo hiểm online, khách điền form khai báo sức khỏe (nhiều bước, dài) rồi thường bỏ giữa chừng, cần job nhắc nhở hoàn tất kèm lưu lại đúng dữ liệu đã điền để khách không phải điền lại từ đầu.

**Vai trò của flow:** Flow phải lưu đúng tiến trình khách đã điền, gửi nhắc nhở kèm link quay lại đúng bước đang dừng, và xử lý đúng khi khách sửa lại thông tin đã lưu giữa lúc đang trong danh sách chờ gửi nhắc nhở.

**Yêu cầu cụ thể:**
- Mỗi bước form phải được lưu ngay khi khách hoàn tất (auto-save), không chờ tới cuối cùng mới lưu toàn bộ, để nếu khách bỏ giữa chừng ở bước 5/10, dữ liệu 4 bước trước đó không bị mất.
- Mô tả cụ thể: job đang chuẩn bị gửi email nhắc nhở dựa trên tiến trình "đang ở bước 5" của khách, đúng lúc khách quay lại tự điền tiếp tới bước 8 rồi lại thoát ra — job phải đọc lại tiến trình mới nhất (bước 8) ngay trước khi gửi, không gửi email với link trỏ về bước 5 đã lỗi thời khiến khách bối rối khi click vào thấy tiến trình cũ.
- Nếu khách hoàn tất toàn bộ form và nộp đơn bảo hiểm thành công đúng lúc job đang trong quá trình chuẩn bị gửi email nhắc nhở cho phiên đó, job phải re-check trạng thái "đã nộp đơn" trước khi gửi và hủy gửi nếu đã hoàn tất, tương tự nguyên tắc chung của các job nhắc nhở khác.
- Dữ liệu khai báo sức khỏe là thông tin nhạy cảm — quy định rõ link trong email nhắc nhở phải có token xác thực đủ mạnh và có hạn sử dụng (ví dụ 7 ngày), không để link có thể bị đoán hoặc dùng lại vô thời hạn để truy cập dữ liệu sức khỏe của khách.
- Nếu khách có nhiều phiên khai báo bị bỏ giữa chừng cho các sản phẩm bảo hiểm khác nhau, job phải xử lý và gửi nhắc nhở riêng cho từng phiên độc lập, không gộp nhầm dữ liệu khai báo của sản phẩm A vào email nhắc nhở của sản phẩm B.

---

## Nhắc nhở khách hàng B2B hoàn tất đơn đặt hàng theo yêu cầu (custom order) bị bỏ giữa chừng

**Repository:** `cart-abandonment-b2b-custom-order`

**Hệ thống:** Một platform B2B cho phép khách doanh nghiệp tạo đơn đặt hàng tùy chỉnh (custom order) với nhiều bước cấu hình (số lượng, thông số kỹ thuật, địa điểm giao), quy trình dài và thường có nhiều người trong 1 công ty cùng tham gia chỉnh sửa đơn.

**Vai trò của flow:** Flow phải phát hiện đúng đơn hàng bị "bỏ" (không ai trong công ty đang chỉnh sửa) để gửi nhắc nhở đúng người phụ trách, tránh gửi nhắc nhở khi có đồng nghiệp khác đang tiếp tục chỉnh sửa đơn đó.

**Yêu cầu cụ thể:**
- Vì nhiều người trong công ty có thể cùng truy cập 1 đơn hàng custom order, job phải kiểm tra "không có ai chỉnh sửa trong X thời gian gần nhất" dựa trên hoạt động của TẤT CẢ user liên quan tới đơn đó (không chỉ người tạo đơn ban đầu), tránh gửi nhắc nhở khi thực ra đồng nghiệp B đang tiếp tục hoàn thiện đơn mà job chỉ theo dõi hoạt động của người tạo A.
- Mô tả cụ thể: người tạo đơn A dừng chỉnh sửa lúc 10:00, đồng nghiệp B mở đơn đó lúc 10:05 và đang chỉnh sửa, job quét vào lúc 10:10 với ngưỡng "không hoạt động 15 phút" — job phải tính đúng dựa trên hoạt động gần nhất là của B (10:05), không nhắc nhở nhầm vì tính theo A (dừng lúc 10:00, đã hơn 15 phút nếu chỉ tính riêng A).
- Nếu đơn hàng được hoàn tất (submit) đúng lúc job đang xử lý batch chứa đơn đó, phải re-check trạng thái "đã submit" ngay trước khi gửi thông báo, không gửi nhắc nhở "hoàn tất đơn hàng" cho đơn đã submit xong.
- Gửi nhắc nhở tới đúng người có quyền/trách nhiệm hoàn tất đơn (có thể không phải người tạo ban đầu, ví dụ đã được chuyển giao cho người khác trong quy trình phê duyệt nội bộ công ty) — quy định rõ logic xác định người nhận nhắc nhở dựa trên trạng thái phân quyền hiện tại của đơn, không hardcode luôn gửi cho người tạo đầu tiên.
- Có cơ chế escalate nếu đơn hàng bị bỏ quá lâu (ví dụ sau 3 lần nhắc nhở trong 2 tuần không ai xử lý), tự động thông báo cho quản lý cấp cao hơn hoặc đội sales phụ trách khách hàng đó để can thiệp, tránh mất đơn hàng lớn chỉ vì không ai chú ý tiếp.
