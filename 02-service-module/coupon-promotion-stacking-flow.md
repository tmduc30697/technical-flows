# Coupon/promotion stacking flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (e-commerce checkout, giao đồ ăn, subscription SaaS, marketplace flash sale, ride-hailing) để luyện việc áp dụng nhiều mã giảm giá/khuyến mãi cùng lúc một cách nhất quán, đúng thứ tự, và chống lạm dụng.

---

## Áp nhiều mã giảm giá cùng lúc tại checkout e-commerce

**Repository:** `coupon-stacking-ecommerce-checkout`

**Hệ thống:** Một sàn e-commerce cho phép khách hàng áp dụng đồng thời nhiều loại khuyến mãi trong một đơn hàng: mã giảm giá phần trăm, mã giảm giá số tiền cố định, và miễn phí vận chuyển.

**Vai trò của flow:** Flow này quyết định thứ tự và cách kết hợp các loại giảm giá để ra số tiền cuối cùng chính xác, tránh giảm giá âm hoặc vượt quá giá trị đơn hàng.

**Yêu cầu cụ thể:**
- Định nghĩa rõ thứ tự áp dụng khi có nhiều loại giảm giá cùng lúc (ví dụ: giảm phần trăm áp dụng trước, giảm số tiền cố định áp dụng sau trên số đã giảm, miễn phí vận chuyển tính riêng) và áp dụng nhất quán, có thể giải thích được cho khách hàng qua breakdown rõ ràng.
- Xử lý trường hợp tổng giảm giá từ nhiều mã vượt quá giá trị đơn hàng — số tiền cuối cùng phải có sàn tối thiểu (ví dụ không được âm, hoặc không dưới một mức tối thiểu theo chính sách) và không phần nào của giảm giá bị "lãng phí" theo cách gây nhầm lẫn cho khách.
- Mỗi mã giảm giá phải khai báo rõ có được stack (kết hợp) với các mã khác hay không (một số mã chỉ dùng độc lập, không kết hợp với khuyến mãi khác) — hệ thống phải validate và từ chối kết hợp không hợp lệ ngay khi khách thêm mã thứ hai, kèm thông báo lý do rõ ràng.
- Xử lý race condition khi một mã giảm giá có giới hạn số lượt sử dụng tổng (ví dụ chỉ 100 lượt đầu) và nhiều khách hàng cùng áp dụng gần như đồng thời khi sắp hết lượt — không được để vượt quá giới hạn đã định do đọc/ghi không atomic.
- Khi một phần đơn hàng bị hủy/hoàn trả sau khi đã áp nhiều mã giảm giá, phải tính đúng lại phần giảm giá tương ứng bị mất hiệu lực cho phần hàng hoàn trả đó, không hoàn toàn bộ giảm giá hay giữ nguyên như đơn hàng đầy đủ.

---

## Khuyến mãi combo (mua kèm giảm giá) cho app giao đồ ăn

**Repository:** `coupon-stacking-food-delivery-combo`

**Hệ thống:** Một app giao đồ ăn có nhiều loại khuyến mãi chạy đồng thời: giảm giá theo nhà hàng, mã giảm giá của nền tảng, và ưu đãi thẻ ngân hàng liên kết.

**Vai trò của flow:** Vì ba loại khuyến mãi này do ba bên tài trợ khác nhau (nhà hàng, nền tảng, ngân hàng), flow phải phân bổ đúng ai chịu phần giảm giá nào để đối soát tài chính chính xác.

**Yêu cầu cụ thể:**
- Khi một đơn hàng áp dụng đồng thời cả ba loại khuyến mãi, phải tính và lưu lại rõ ràng phần giảm giá nào do nhà hàng chịu (trừ vào doanh thu nhà hàng nhận), phần nào nền tảng chịu (trừ vào doanh thu nền tảng), và phần nào ngân hàng hoàn lại sau (cashback riêng, không trừ ngay vào đơn hàng) — không gộp lẫn ba nguồn khiến đối soát sai.
- Xử lý trường hợp khuyến mãi của nhà hàng có điều kiện riêng (ví dụ chỉ áp dụng cho một số món trong menu) trong khi mã giảm giá của nền tảng áp dụng trên tổng đơn hàng — phải tính đúng thứ tự: giảm giá theo món trước, rồi mới áp mã giảm trên tổng còn lại.
- Đảm bảo tổng số tiền giảm giá không vượt quá ngân sách khuyến mãi mà nhà hàng hoặc nền tảng đã đặt ra cho chiến dịch đó (budget cap) — khi ngân sách gần hết, phải tự động ngừng áp dụng khuyến mãi đó cho các đơn hàng tiếp theo và thông báo rõ cho khách (thay vì áp dụng rồi từ chối thanh toán).
- Xử lý trường hợp một đơn hàng bị hủy một phần (một món trong đơn không còn khi nhà hàng xác nhận) — phải tính lại đúng phần giảm giá tương ứng với món bị hủy, cả ba nguồn giảm giá đều phải được điều chỉnh lại chính xác.
- Cung cấp báo cáo đối soát rõ ràng cho từng nhà hàng và ngân hàng đối tác về tổng số tiền khuyến mãi họ đã tài trợ trong một kỳ, để tránh tranh chấp về công nợ khuyến mãi.

---

## Mã giảm giá kết hợp với chương trình giới thiệu cho subscription SaaS

**Repository:** `coupon-stacking-saas-referral`

**Hệ thống:** Một dịch vụ SaaS cho cá nhân có mã giảm giá theo chiến dịch marketing và một chương trình giới thiệu bạn bè (referral) riêng, khách hàng có thể có cả hai cùng lúc khi đăng ký.

**Vai trò của flow:** Vì đây là subscription định kỳ (không phải một giao dịch một lần), flow phải xử lý việc giảm giá áp dụng qua nhiều kỳ thanh toán, không chỉ một lần tại thời điểm đăng ký.

**Yêu cầu cụ thể:**
- Định nghĩa rõ mã giảm giá marketing (thường chỉ áp dụng cho kỳ đầu hoặc vài kỳ đầu) và ưu đãi referral (có thể áp dụng lâu dài hoặc theo số lượng người được giới thiệu) có được cộng gộp hay không, và nếu cộng gộp thì tổng mức giảm tối đa là bao nhiêu phần trăm giá gốc.
- Khi một trong hai ưu đãi hết hiệu lực giữa chu kỳ subscription (ví dụ mã marketing chỉ áp dụng 3 tháng đầu, tới tháng 4 hết hiệu lực), hóa đơn tháng đó phải tự động tính lại đúng giá mới (chỉ còn ưu đãi referral nếu còn hiệu lực) và có thông báo trước cho khách hàng về việc giá sẽ thay đổi.
- Xử lý trường hợp khách hàng hủy subscription rồi đăng ký lại (re-subscribe) — phải có rule rõ ràng liệu các ưu đãi cũ (đặc biệt mã marketing chỉ dành cho khách mới) còn được áp dụng lại hay không, tránh bị lợi dụng hủy/đăng ký lại liên tục để hưởng ưu đãi khách mới nhiều lần.
- Đảm bảo khi khách hàng upgrade/downgrade gói giữa chừng, phần giảm giá (phần trăm hoặc số tiền cố định) được áp dụng đúng lại theo giá gói mới, không giữ nguyên số tiền giảm tuyệt đối tính theo giá gói cũ gây sai lệch tỷ lệ giảm giá đã cam kết.
- Ghi log lịch sử áp dụng ưu đãi cho mỗi subscription (loại ưu đãi, kỳ áp dụng, giá trị) để hỗ trợ customer support giải thích khi khách hàng thắc mắc về sự thay đổi giá hóa đơn giữa các tháng.

---

## Voucher giới hạn theo giờ trong flash sale trên marketplace

**Repository:** `coupon-stacking-flash-sale-time-limited`

**Hệ thống:** Một marketplace tổ chức flash sale theo giờ, phát hành voucher số lượng giới hạn (ví dụ 500 voucher/giờ) có thể kết hợp với mã giảm giá chung của người bán.

**Vai trò của flow:** Đặc thù ở đây là traffic dồn cực lớn vào đúng thời điểm mở voucher, đòi hỏi flow phải chịu được tải cao và tuyệt đối không phát vượt số lượng giới hạn.

**Yêu cầu cụ thể:**
- Việc "giữ" một voucher cho một khách hàng (khi họ bấm áp dụng) phải là một hoạt động atomic chống race condition tuyệt đối — với hàng nghìn request cùng lúc tranh nhau 500 voucher, không được để phát ra 501 voucher hoặc hơn do điều kiện tranh chấp giữa nhiều request đồng thời.
- Nếu khách hàng giữ được voucher nhưng không hoàn tất thanh toán trong một khoảng thời gian xác định (ví dụ 10 phút), voucher phải được tự động trả lại vào pool để người khác có thể dùng, không bị "khóa chết" bởi khách không thanh toán.
- Xử lý việc kết hợp voucher flash sale với mã giảm giá của người bán — tổng giảm giá không được khiến người bán phải bán dưới giá vốn đã cam kết tối thiểu với nền tảng (nếu chính sách flash sale có điều khoản bảo vệ biên lợi nhuận tối thiểu cho người bán).
- Đảm bảo trải nghiệm công bằng: thời điểm request "áp dụng voucher" được ghi nhận phải chính xác theo thứ tự thực tế đến server (không bị ảnh hưởng bởi độ trễ mạng khác nhau của từng khách hàng một cách bất công), và hệ thống phải minh bạch khi thông báo "hết voucher" ngay khi số lượng đã chạm giới hạn, không để khách hàng chờ vô ích.
- Viết test tải mô phỏng hàng chục nghìn request đồng thời tranh 500 voucher, verify đúng số lượng voucher được phát ra chính xác bằng giới hạn, không thừa không thiếu.

---

## Combo mã giảm giá và tín dụng khuyến mãi (promo credit) cho ứng dụng đặt xe

**Repository:** `coupon-stacking-ride-hailing-promo-credit`

**Hệ thống:** Một ứng dụng đặt xe có hai loại ưu đãi có thể tồn tại cùng lúc trong tài khoản khách: mã giảm giá theo chuyến (một lần dùng) và số dư tín dụng khuyến mãi (promo credit, có thể dùng dần cho nhiều chuyến, thường có ngày hết hạn).

**Vai trò của flow:** Vì promo credit là một dạng "số dư" dùng dần (khác mã giảm giá dùng một lần), flow phải quản lý đúng thứ tự sử dụng và hết hạn của số dư này khi kết hợp với mã giảm giá theo chuyến.

**Yêu cầu cụ thể:**
- Định nghĩa rõ thứ tự ưu tiên khi thanh toán một chuyến đi có cả mã giảm giá và promo credit khả dụng — ví dụ áp mã giảm giá trước để giảm giá chuyến, sau đó phần còn lại mới được trừ vào promo credit, và phần vượt quá số dư mới trừ vào phương thức thanh toán chính.
- Promo credit có ngày hết hạn phải được trừ theo nguyên tắc "hết hạn sớm nhất dùng trước" (giống FIFO) khi một tài khoản có nhiều khoản credit từ các chiến dịch khác nhau với ngày hết hạn khác nhau, tránh để khoản sắp hết hạn bị bỏ quên không dùng tới rồi mất.
- Xử lý trường hợp chuyến đi bị hủy sau khi đã trừ cả mã giảm giá và promo credit — phải hoàn lại đúng phần promo credit đã trừ (cộng lại vào số dư, giữ nguyên ngày hết hạn ban đầu của khoản đó) trong khi mã giảm giá một lần có thể không được hoàn theo chính sách (do đã dùng), và phải phân biệt rõ hai cách xử lý này.
- Đảm bảo tính atomic khi trừ đồng thời từ nhiều nguồn (mã giảm giá + promo credit + phương thức thanh toán chính) trong một giao dịch — nếu bước cuối (charge thẻ phần còn thiếu) thất bại, phải rollback đúng cả phần đã trừ promo credit và không đánh dấu mã giảm giá đã dùng.
- Hiển thị rõ cho khách hàng trước khi xác nhận chuyến đi: số tiền đã giảm từ mã, số promo credit sẽ bị trừ, và số tiền thực tế sẽ charge vào phương thức thanh toán chính, tránh gây bất ngờ về số dư promo credit giảm nhanh hơn khách tưởng.
