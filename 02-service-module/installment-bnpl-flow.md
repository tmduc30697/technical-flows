# Installment/BNPL flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (e-commerce checkout, marketplace tài trợ vốn cho người bán, nền tảng giáo dục, subscription SaaS, dịch vụ tài trợ mua thiết bị) để luyện việc thiết kế trả góp/mua trước trả sau đúng, xử lý được nợ xấu và các trường hợp hủy/hoàn tiền giữa kỳ.

---

## BNPL (mua trước trả sau) tại checkout của sàn e-commerce

**Repository:** `bnpl-ecommerce-checkout`

**Hệ thống:** Một sàn e-commerce tích hợp tùy chọn "mua trước, trả sau" cho phép khách hàng chia đơn hàng thành 3-4 kỳ trả góp không lãi suất, được một đối tác tài chính bên thứ ba bảo lãnh.

**Vai trò của flow:** Flow này xử lý việc duyệt hạn mức BNPL tại thời điểm checkout, tạo lịch trả góp, và đồng bộ trạng thái thanh toán từng kỳ với đối tác tài chính.

**Yêu cầu cụ thể:**
- Tại thời điểm checkout, phải gọi đối tác tài chính để duyệt hạn mức trong thời gian thực (vài giây), và xử lý rõ 3 kết quả: duyệt toàn phần, duyệt một phần (hạn mức thấp hơn giá trị đơn hàng), và từ chối — mỗi trường hợp có luồng UX khác nhau, không chặn cứng người dùng nếu có thể chuyển sang phương thức thanh toán khác.
- Khi đơn hàng được duyệt BNPL và merchant giao hàng, merchant phải được thanh toán đủ giá trị đơn hàng ngay (do đối tác tài chính bảo lãnh), độc lập với việc khách hàng có trả góp đúng hạn sau này hay không — tách rõ luồng tiền cho merchant với luồng thu nợ khách hàng.
- Xử lý trường hợp khách hàng hủy đơn/trả hàng sau khi đã có lịch trả góp được tạo — phải hủy hoặc điều chỉnh đúng các kỳ trả góp chưa tới hạn, và xử lý hoàn tiền đúng cho các kỳ đã trả (nếu có) mà không gây sai lệch với đối tác tài chính.
- Định nghĩa rõ hành vi khi khách hàng trả trễ một kỳ: tính phí trễ hạn (nếu có) theo đúng chính sách, gửi nhắc nhở tự động trước và sau ngày đến hạn, và có ngưỡng số lần trễ liên tiếp để tạm khóa tính năng BNPL cho tài khoản đó ở các đơn hàng tiếp theo.
- Đảm bảo idempotency khi đồng bộ trạng thái thanh toán từng kỳ với đối tác tài chính qua webhook — một webhook báo "đã thanh toán kỳ 2" bị gửi lại nhiều lần (do retry mạng) không được cộng tiền/đánh dấu trùng lặp.

---

## Marketplace tài trợ vốn lưu động cho người bán, thu hồi qua doanh thu tương lai

**Repository:** `bnpl-marketplace-seller-financing`

**Hệ thống:** Một marketplace cho phép người bán vừa và nhỏ vay một khoản vốn lưu động, khoản vay này được thu hồi tự động bằng cách trích một phần trăm doanh thu bán hàng hàng ngày của người bán trên chính marketplace đó.

**Vai trò của flow:** Đây là một dạng "trả góp ngược" — số tiền trả mỗi kỳ không cố định mà phụ thuộc vào doanh thu thực tế, cần flow riêng để tính và trích đúng.

**Yêu cầu cụ thể:**
- Mỗi ngày, hệ thống phải tự động tính phần trăm doanh thu cần trích theo đúng tỷ lệ đã thỏa thuận, trừ vào số tiền chuyển cho người bán, và cập nhật số dư nợ còn lại — phải atomic với chính quy trình thanh toán doanh thu định kỳ cho người bán, không tách rời gây rủi ro tính sai.
- Xử lý trường hợp doanh thu một ngày của người bán bằng 0 hoặc rất thấp (mùa thấp điểm) — không được ép trích một số tiền cố định gây người bán nợ thêm phí trễ hạn một cách bất công, vì bản chất mô hình là theo doanh thu thực tế.
- Khi khoản vay đã được trả hết (dựa trên tổng đã trích lũy kế đạt đủ số tiền vay + phí), phải tự động ngừng trích ngay từ kỳ thanh toán tiếp theo, không trích dư dù chỉ một ngày.
- Xử lý trường hợp người bán ngừng hoạt động trên marketplace giữa lúc còn nợ (đóng shop, chuyển sang nền tảng khác) — phải có quy trình rõ ràng chuyển sang thu hồi nợ theo cách khác (nhắc nợ trực tiếp, chuyển sang đối tác thu hồi nợ) khi không còn doanh thu để trích tự động.
- Cung cấp cho người bán bảng minh bạch theo thời gian thực: số tiền đã vay, đã trả, còn lại, và lịch sử trích theo từng ngày — để tránh tranh chấp về số tiền bị trích khỏi doanh thu của họ.

---

## Trả góp học phí trên nền tảng giáo dục trực tuyến

**Repository:** `bnpl-edtech-tuition-installment`

**Hệ thống:** Một nền tảng học trực tuyến cho phép học viên đăng ký một khóa học dài hạn (ví dụ khóa học 12 tháng) và trả học phí theo từng tháng thay vì trả trọn gói ngay từ đầu.

**Vai trò của flow:** Flow trả góp ở đây gắn liền với việc cấp quyền truy cập nội dung khóa học — quyền truy cập phải đồng bộ chính xác với trạng thái thanh toán từng kỳ.

**Yêu cầu cụ thể:**
- Quyền truy cập nội dung khóa học của học viên phải được cấp/duy trì theo đúng trạng thái thanh toán: nếu một kỳ thanh toán thất bại (thẻ hết hạn, không đủ tiền) và không được khắc phục trong một khoảng thời gian gia hạn xác định, phải tạm khóa quyền truy cập nội dung mới trong khi vẫn giữ được tiến độ học đã hoàn thành trước đó.
- Xử lý việc thử lại thanh toán tự động (dunning) cho kỳ bị thất bại theo một lịch retry hợp lý (ví dụ vài lần trong vài ngày) với thông báo rõ ràng cho học viên trước khi tạm khóa quyền truy cập, tránh khóa đột ngột không báo trước.
- Nếu học viên hoàn tất khóa học (hoàn thành 100% nội dung) trước khi trả hết các kỳ còn lại, phải định nghĩa rõ chính sách: vẫn phải hoàn tất nghĩa vụ thanh toán còn lại theo hợp đồng ban đầu, hay được điều chỉnh — và phản ánh đúng trên hệ thống thanh toán, không để mập mờ.
- Xử lý trường hợp học viên muốn hủy giữa khóa học — tính đúng số tiền đã học/đã dùng dịch vụ (ví dụ theo % nội dung đã truy cập) để quyết định số tiền hoàn lại hoặc số nợ còn phải trả, theo chính sách hoàn tiền đã công bố.
- Đảm bảo việc thay đổi giá khóa học (nếu nền tảng tăng giá sau này) không ảnh hưởng tới các học viên đã đăng ký trả góp theo giá cũ — giá và lịch trả góp phải được chốt tại thời điểm đăng ký.

---

## Nâng cấp từ subscription hàng năm trả trước sang trả góp hàng tháng

**Repository:** `bnpl-saas-subscription-installment-upgrade`

**Hệ thống:** Một dịch vụ SaaS cho cá nhân, cho phép người dùng chọn trả trọn gói theo năm (giá ưu đãi hơn) hoặc trả góp hàng tháng với giá quy đổi tương đương cộng thêm một phần phí xử lý.

**Vai trò của flow:** Flow trả góp ở đây phải xử lý được việc chuyển đổi giữa hai hình thức thanh toán giữa chừng một chu kỳ, điều mà các flow BNPL thương mại thông thường ít gặp.

**Yêu cầu cụ thể:**
- Khi người dùng đang trả góp hàng tháng muốn chuyển sang trả trọn gói năm giữa chừng, phải tính đúng số tiền còn thiếu (giá trọn gói năm trừ đi số đã trả góp, không đơn giản là cộng dồn các kỳ còn lại theo giá trả góp vì hai mức giá khác nhau).
- Ngược lại, khi người dùng đang ở gói năm trả trước muốn chuyển sang trả góp hàng tháng (ít gặp nhưng cần xử lý, ví dụ do khó khăn tài chính), phải tính toán hoàn lại phần đã trả dư và dựng lại lịch trả góp mới từ thời điểm chuyển đổi, không tính lại từ đầu gây thiệt cho người dùng.
- Nếu người dùng trả góp hàng tháng bị thất bại thanh toán liên tiếp và bị hủy subscription, phải xử lý đúng phần đã sử dụng dịch vụ (ví dụ đã dùng 3/12 tháng của một "cam kết năm" nhưng trả theo tháng) để tính công nợ còn lại nếu hợp đồng có điều khoản cam kết tối thiểu.
- Định nghĩa rõ cách tính phí xử lý trả góp (ví dụ thêm 5% so với giá trả trước) áp dụng minh bạch ngay từ lúc người dùng chọn hình thức thanh toán, không được để phí này biến động qua các tháng.
- Đảm bảo hệ thống billing không tính trùng phí khi một chu kỳ trả góp giao với thời điểm gia hạn subscription (ví dụ kỳ trả góp cuối của năm 1 rơi đúng lúc bắt đầu tính phí năm 2) — cần lịch tính rõ ràng tránh double billing.

---

## Trả góp mua thiết bị (điện thoại) kèm dịch vụ viễn thông

**Repository:** `bnpl-telecom-device-installment`

**Hệ thống:** Một nhà mạng cho phép khách hàng mua điện thoại trả góp kèm theo gói cước hàng tháng, khoản trả góp thiết bị và phí gói cước được gộp vào cùng một hóa đơn hàng tháng.

**Vai trò của flow:** Flow này phải tách bạch rõ hai nghĩa vụ tài chính khác nhau (nợ thiết bị và phí dịch vụ định kỳ) dù xuất hiện chung trên một hóa đơn, vì hệ quả pháp lý và xử lý khi có sự cố là khác nhau.

**Yêu cầu cụ thể:**
- Hóa đơn hàng tháng phải tách rõ hai dòng: phần trả góp thiết bị (giảm dần số dư nợ gốc) và phần phí dịch vụ viễn thông (không tích lũy nợ, chỉ là phí sử dụng kỳ đó) — không được gộp thành một số tiền duy nhất không minh bạch.
- Khi khách hàng thanh toán không đủ toàn bộ hóa đơn (chỉ trả được một phần), phải có chính sách rõ ràng về việc phần trả được ưu tiên trừ vào nghĩa vụ nào trước (ví dụ ưu tiên trả góp thiết bị để tránh mất quyền sở hữu thiết bị, hay ưu tiên phí dịch vụ để duy trì kết nối) và áp dụng nhất quán.
- Xử lý trường hợp khách hàng muốn ngừng dịch vụ viễn thông giữa lúc còn nợ tiền thiết bị — dịch vụ viễn thông có thể ngừng nhưng nghĩa vụ trả góp thiết bị vẫn tiếp tục theo hợp đồng riêng, hệ thống phải theo dõi được nghĩa vụ này độc lập với trạng thái gói cước.
- Nếu khách hàng trả trễ nợ thiết bị nhiều kỳ liên tiếp, định nghĩa rõ chính sách hậu quả (khóa một phần dịch vụ, chuyển nợ sang bộ phận thu hồi) khác với việc trễ phí dịch vụ thông thường (thường chỉ tạm khóa dịch vụ), và không nhầm lẫn xử lý hai loại nợ theo cùng một quy tắc.
- Đảm bảo khi khách hàng trả hết nợ thiết bị trước hạn (trả sớm toàn bộ số dư còn lại), hệ thống tính đúng số tiền cần trả (không cộng thêm lãi cho các kỳ chưa tới, nếu chính sách không có lãi hoặc có định nghĩa cách tính lãi trả sớm rõ ràng).
