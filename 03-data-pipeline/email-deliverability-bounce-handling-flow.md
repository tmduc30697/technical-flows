# Email deliverability/bounce handling flow — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — e-commerce, nền tảng marketing/newsletter, SaaS B2B, fintech, nền tảng tuyển dụng, và hệ thống xác thực người dùng — nhằm luyện đủ các góc của flow gửi email và xử lý bounce/complaint (giữ domain reputation, phân loại bounce, retry hợp lý, làm sạch danh sách gửi, tuân thủ quy định chống spam).

---

## E-commerce gửi email xác nhận đơn hàng/vận chuyển

**Repository:** `email-deliverability-ecommerce-order-confirmation`

**Hệ thống:** Một sàn e-commerce cần gửi email giao dịch (order confirmation, shipping update, invoice) cho khách hàng sau mỗi bước xử lý đơn hàng.

**Vai trò của flow:** Gửi email transactional với độ tin cậy cao, theo dõi trạng thái gửi/bounce để đảm bảo thông tin quan trọng của đơn hàng đến được khách hàng.

**Yêu cầu cụ thể:**
- Email transactional phải được gửi qua hàng đợi riêng, tách biệt hoàn toàn khỏi email marketing, để một chiến dịch marketing bị nhà cung cấp đánh giá thấp không ảnh hưởng đến khả năng gửi email đơn hàng quan trọng.
- Khi nhận bounce loại "hard bounce" (địa chỉ không tồn tại), phải đánh dấu địa chỉ đó không gửi lại nữa cho các email tiếp theo và cảnh báo cho khách hàng qua kênh khác (trong app) để cập nhật email đúng.
- Với "soft bounce" (hộp mail đầy, server tạm lỗi), phải retry theo lịch tăng dần (backoff) trong một khoảng thời gian hợp lý, không retry vô hạn và không coi là hard bounce ngay từ lần thất bại đầu tiên.
- Nếu email chứa thông tin quan trọng (ví dụ mã vận đơn) bị bounce mà không có cách gửi lại được, hệ thống phải đảm bảo thông tin đó vẫn hiển thị được ở nơi khác (trang chi tiết đơn hàng trong app/web) để khách hàng không bị mất thông tin hoàn toàn.
- Theo dõi tỷ lệ bounce/complaint tổng thể của domain gửi theo thời gian thực, cảnh báo sớm khi tỷ lệ vượt ngưỡng an toàn (nguy cơ domain bị đưa vào blacklist) trước khi ảnh hưởng lan rộng tới toàn bộ email transactional.

---

## Nền tảng marketing/newsletter gửi bulk email tới danh sách lớn

**Repository:** `email-deliverability-newsletter-bulk`

**Hệ thống:** Một SaaS cho phép doanh nghiệp gửi newsletter/chiến dịch marketing tới danh sách người nhận lên tới hàng triệu địa chỉ.

**Vai trò của flow:** Gửi bulk email và xử lý bounce/complaint ở quy mô lớn để bảo vệ domain reputation dùng chung cho nhiều khách hàng của SaaS.

**Yêu cầu cụ thể:**
- Danh sách gửi phải được lọc bỏ các địa chỉ đã hard-bounce hoặc complaint (đánh dấu spam) từ các chiến dịch trước đó ngay trước khi gửi, không dựa vào danh sách đã lọc cũ có thể lỗi thời.
- Domain/IP gửi phải được cách ly theo mức độ uy tín của khách hàng (dedicated IP cho khách gửi lớn/uy tín cao, IP pool chung cho khách nhỏ) để một khách hàng gửi nội dung kém chất lượng không kéo giảm reputation ảnh hưởng tới các khách hàng khác dùng chung nền tảng.
- Khi tỷ lệ complaint của một chiến dịch đang chạy vượt ngưỡng nguy hiểm giữa lúc đang gửi, hệ thống phải tự động tạm dừng phần còn lại của chiến dịch đó, không chờ gửi hết toàn bộ danh sách rồi mới phát hiện.
- Phải triển khai warm-up gửi dần (tăng volume từ từ) khi bắt đầu dùng một domain/IP gửi mới, tránh gửi khối lượng lớn ngay từ đầu làm ISP đánh giá là spam.
- Cung cấp cho khách hàng doanh nghiệp báo cáo chi tiết bounce/complaint theo từng chiến dịch (không chỉ tổng số gửi thành công) để họ tự làm sạch danh sách người nhận của mình theo thời gian.

---

## SaaS B2B gửi email transactional theo từng tenant

**Repository:** `email-deliverability-b2b-saas-transactional`

**Hệ thống:** Một SaaS quản lý vận hành cho nhiều doanh nghiệp khách hàng (multi-tenant), mỗi tenant gửi email hệ thống riêng (hóa đơn, cảnh báo, báo cáo) tới người dùng cuối của họ.

**Vai trò của flow:** Gửi email transactional theo từng tenant và theo dõi bounce riêng biệt để một tenant gửi email chất lượng kém không ảnh hưởng tới khả năng gửi của các tenant khác.

**Yêu cầu cụ thể:**
- Tỷ lệ bounce/complaint phải được đo và báo cáo riêng theo từng tenant, không gộp chung một chỉ số tổng của toàn nền tảng, để phát hiện đúng tenant nào đang gây rủi ro cho domain reputation chung.
- Khi một tenant cho phép người dùng cuối của họ tự nhập email nhận thông báo (có thể nhập sai/email rác), hệ thống phải validate định dạng và có bước xác minh (double opt-in hoặc gửi thử) trước khi đưa vào danh sách gửi định kỳ.
- Nếu một tenant vượt ngưỡng bounce rate cho phép, phải tự động giới hạn/tạm ngưng khả năng gửi email của riêng tenant đó và thông báo cho họ lý do, không ảnh hưởng đến việc gửi email của các tenant khác.
- Tenant phải tự cấu hình được domain gửi riêng của họ (custom sending domain với SPF/DKIM/DMARC riêng) nếu muốn, và hệ thống phải theo dõi deliverability riêng cho domain đó tách biệt khỏi domain mặc định chung của SaaS.
- Cho tenant xem được dashboard chi tiết về email đã gửi/bounce/mở của người dùng cuối họ, phục vụ họ tự cải thiện chất lượng danh sách liên hệ.

---

## Ngân hàng số gửi email sao kê và cảnh báo bảo mật

**Repository:** `email-deliverability-digital-bank-statements-alerts`

**Hệ thống:** Một ngân hàng số cần gửi email sao kê giao dịch định kỳ và cảnh báo bảo mật (đăng nhập bất thường, thay đổi thông tin) cho khách hàng.

**Vai trò của flow:** Gửi email với yêu cầu bảo mật và độ tin cậy cao nhất, theo dõi delivery chặt chẽ để phục vụ tuân thủ quy định ngành ngân hàng.

**Yêu cầu cụ thể:**
- Mọi email chứa thông tin tài chính nhạy cảm phải được ký số/mã hóa theo chuẩn phù hợp và gửi qua domain đã được xác thực đầy đủ (SPF/DKIM/DMARC nghiêm ngặt), không dùng domain phụ chưa được thiết lập bảo mật đầy đủ.
- Nếu email cảnh báo bảo mật (ví dụ đăng nhập từ thiết bị lạ) bị bounce, hệ thống phải escalate ngay sang kênh khác (SMS/push) trong thời gian ngắn nhất, vì đây là cảnh báo an toàn tài khoản không thể chấp nhận bị mất hoàn toàn.
- Toàn bộ trạng thái gửi (đã gửi, đã mở, bounce, thời gian) của email liên quan đến bảo mật/tài chính phải được lưu trữ đủ lâu và đầy đủ để phục vụ tra soát khi có tranh chấp hoặc yêu cầu từ cơ quan quản lý.
- Khi phát hiện một địa chỉ email khách hàng liên tục bounce (có thể do khách đổi email nhưng chưa cập nhật hệ thống), phải có quy trình xác minh danh tính qua kênh khác trước khi cho phép đổi email nhận sao kê, tránh kẻ gian chiếm quyền nhận thông tin tài chính.
- Không được gửi thông tin nhạy cảm cụ thể (số dư, số thẻ) trực tiếp trong nội dung email nếu domain/địa chỉ người nhận có lịch sử bị đánh dấu rủi ro (ví dụ từng bị báo cáo compromised), thay vào đó chỉ gửi thông báo yêu cầu đăng nhập vào app để xem chi tiết.

---

## Nền tảng tuyển dụng gửi email cho ứng viên và nhà tuyển dụng

**Repository:** `email-deliverability-recruiting-platform`

**Hệ thống:** Một nền tảng tuyển dụng trực tuyến gửi email thông báo (có đơn ứng tuyển mới, trạng thái hồ sơ, gợi ý việc làm) cho cả ứng viên và nhà tuyển dụng.

**Vai trò của flow:** Gửi email đa dạng loại (transactional và digest định kỳ) và xử lý bounce để giữ danh sách liên hệ sạch, tối ưu tỷ lệ email thực sự đến được người nhận đang tìm việc/tuyển dụng.

**Yêu cầu cụ thể:**
- Ứng viên đăng ký từ lâu nhưng không còn hoạt động (email cũ, đổi công việc) thường có tỷ lệ bounce cao hơn — hệ thống phải phát hiện và giảm dần tần suất gửi digest cho các địa chỉ có dấu hiệu không còn tương tác, trước khi chúng gây hại tới domain reputation.
- Khi nhà tuyển dụng nhận thông báo "có ứng viên mới" bị bounce liên tục (ví dụ công ty đổi hệ thống email nội bộ), phải cảnh báo cho nhà tuyển dụng qua kênh khác (trong app/dashboard) để họ biết và cập nhật, tránh mất cơ hội xem ứng viên mới mà không hay biết.
- Email digest định kỳ (gợi ý việc làm) phải tách khỏi hàng đợi email transactional quan trọng (ví dụ thông báo được nhận offer), để nếu digest có tỷ lệ bounce/complaint cao hơn cũng không ảnh hưởng tới việc gửi các thông báo quan trọng.
- Phải tôn trọng đúng tần suất người dùng đã chọn (hàng ngày/hàng tuần/tắt hẳn) cho từng loại email, và việc thay đổi cấu hình phải có hiệu lực ngay từ lần gửi tiếp theo, không cần chờ chu kỳ xử lý theo batch cũ.
- Địa chỉ email công ty dùng để nhận thông báo tuyển dụng (thường dùng chung nhiều người) phải được theo dõi bounce riêng và có cơ chế cho phép nhiều người trong công ty cùng nhận, tránh một điểm lỗi duy nhất khi người phụ trách nghỉ việc/đổi email.

---

## Nền tảng xác thực gửi email xác minh và đặt lại mật khẩu

**Repository:** `email-deliverability-auth-verification-reset`

**Hệ thống:** Một hệ thống đăng ký/đăng nhập dùng chung cho nhiều sản phẩm web, cần gửi email xác minh tài khoản mới và email đặt lại mật khẩu.

**Vai trò của flow:** Gửi email xác thực với yêu cầu về độ trễ thấp và độ tin cậy cao, xử lý bounce để tránh tình trạng người dùng bị kẹt không thể hoàn tất đăng ký/khôi phục tài khoản.

**Yêu cầu cụ thể:**
- Email xác minh/đặt lại mật khẩu phải được gửi qua hàng đợi ưu tiên cao nhất, tách biệt hoàn toàn với mọi loại email khác (marketing, thông báo thường), để không bị delay do nghẽn hàng đợi khác.
- Nếu gửi lần đầu bị bounce tạm thời (soft bounce), phải retry nhanh trong vài phút vì người dùng đang chờ để hoàn tất luồng đăng ký/khôi phục ngay lúc đó, không áp dụng backoff dài như email thông thường.
- Link xác minh/đặt lại mật khẩu trong email phải có thời hạn hết hiệu lực rõ ràng; nếu email bị gửi lại nhiều lần (người dùng bấm "gửi lại" liên tục), các link cũ phải bị vô hiệu hóa để tránh dùng nhầm link cũ đã lộ.
- Khi phát hiện domain email của người dùng đăng ký nằm trong danh sách domain email rác/dùng một lần (disposable email), phải có chính sách rõ ràng (chặn hoặc cảnh báo) trước khi gửi, tránh lãng phí gửi email và ảnh hưởng số liệu deliverability vì các domain này thường có tỷ lệ bounce/complaint rất cao.
- Cung cấp cho người dùng phương án dự phòng nếu không nhận được email sau một khoảng thời gian hợp lý (ví dụ xác minh qua SMS thay thế), không để người dùng bị kẹt hoàn toàn nếu email của họ liên tục bị nhà cung cấp mail chặn.
