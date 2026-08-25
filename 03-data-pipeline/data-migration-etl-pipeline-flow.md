# Data migration/ETL pipeline flow — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — e-commerce chuyển đổi kiến trúc, fintech đồng bộ dữ liệu ngân hàng, SaaS B2B tích hợp CRM, mạng xã hội sharding lại database, marketplace hợp nhất dữ liệu đa nguồn, và bảo hiểm xử lý batch đêm — nhằm luyện đủ các góc của flow migration/ETL (tính đúng đắn dữ liệu, idempotency, xử lý lỗi từng phần, khả năng rollback, throughput lớn).

---

## E-commerce chuyển đổi từ monolith sang microservices

**Repository:** `etl-ecommerce-monolith-to-microservices`

**Hệ thống:** Một sàn e-commerce đang chạy trên một database monolith duy nhất, cần tách dần sang các service riêng (order, inventory, user) với database riêng cho mỗi service.

**Vai trò của flow:** Di chuyển dữ liệu lịch sử từ database cũ sang các database mới tương ứng, đồng thời đảm bảo dữ liệu vẫn nhất quán trong giai đoạn cả hai hệ thống cùng chạy song song (dual-write/strangler pattern).

**Yêu cầu cụ thể:**
- Migration phải chạy được theo từng batch nhỏ (không load toàn bộ bảng hàng chục triệu dòng vào memory một lần), có khả năng tạm dừng và tiếp tục từ điểm dừng.
- Trong giai đoạn chuyển tiếp, dữ liệu ghi mới vào hệ thống cũ phải được đồng bộ sang hệ thống mới gần thời gian thực (change data capture), không để lệch dữ liệu giữa hai bên quá một ngưỡng thời gian xác định.
- Phải có bước validate đối soát số lượng bản ghi và checksum dữ liệu giữa nguồn và đích sau mỗi batch, phát hiện được sai lệch trước khi coi batch đó là "hoàn tất".
- Xử lý được các bản ghi có dữ liệu không hợp lệ theo schema mới (ví dụ trường bắt buộc bị null ở dữ liệu cũ) — đưa vào một nơi riêng để xử lý thủ công, không làm dừng toàn bộ pipeline.
- Có kế hoạch rollback rõ ràng: nếu phát hiện lỗi nghiêm trọng sau khi cắt traffic sang hệ thống mới, phải quay lại được hệ thống cũ mà không mất dữ liệu ghi mới trong lúc đã chuyển.

---

## Fintech đồng bộ dữ liệu giao dịch từ core banking sang data warehouse

**Repository:** `etl-fintech-core-banking-warehouse`

**Hệ thống:** Một ngân hàng số có hệ thống core banking xử lý giao dịch, cần đẩy dữ liệu sang data warehouse để phục vụ báo cáo tài chính và phân tích rủi ro.

**Vai trò của flow:** Trích xuất giao dịch mới/thay đổi từ core banking theo lịch định kỳ (hoặc gần thời gian thực), biến đổi về mô hình dữ liệu phân tích, và tải vào warehouse.

**Yêu cầu cụ thể:**
- Pipeline phải đảm bảo mỗi giao dịch chỉ được tính đúng một lần trong warehouse dù job ETL chạy lại nhiều lần (idempotent theo transaction ID), không tạo double-count khi retry.
- Số dư/tổng giao dịch tính được ở warehouse phải khớp tuyệt đối với số liệu ở core banking tại cùng một mốc thời gian — có báo cáo đối soát tự động chạy sau mỗi lần ETL và cảnh báo khi lệch dù chỉ một đồng.
- Xử lý đúng các giao dịch bị điều chỉnh/hoàn tác sau khi đã ETL vào warehouse (ví dụ giao dịch bị đảo do gian lận) — phải phản ánh được bản mới nhất, không giữ lại dữ liệu cũ đã lỗi thời.
- Toàn bộ dữ liệu tài chính di chuyển qua pipeline phải được mã hóa và có audit trail đầy đủ (ai/khi nào/dữ liệu gì) để đáp ứng yêu cầu compliance của ngành ngân hàng.
- Nếu core banking tạm ngừng phục vụ (maintenance window), job ETL phải phát hiện, không chạy dở dang gây thiếu dữ liệu, và tự resume đúng từ thời điểm dữ liệu bị gián đoạn khi core banking hoạt động lại.

---

## SaaS B2B đồng bộ dữ liệu khách hàng từ CRM (Salesforce)

**Repository:** `etl-b2b-saas-salesforce-sync`

**Hệ thống:** Một SaaS quản lý vận hành cho doanh nghiệp, cho phép mỗi tenant kết nối CRM (Salesforce) của họ để đồng bộ dữ liệu khách hàng/deal vào hệ thống nội bộ.

**Vai trò của flow:** Định kỳ (hoặc qua webhook) kéo dữ liệu từ CRM của từng tenant, chuẩn hóa schema, và ghi vào database nội bộ tương ứng với tenant đó.

**Yêu cầu cụ thể:**
- Mỗi tenant có schema CRM tùy biến khác nhau (custom field khác nhau); pipeline phải map được cấu hình field theo từng tenant, không dùng cứng một mapping chung cho mọi khách hàng.
- Đồng bộ phải xử lý đúng conflict khi cùng một bản ghi bị sửa cả ở CRM và ở hệ thống nội bộ trước khi đồng bộ chạy — có chiến lược rõ ràng (last-write-wins theo timestamp, hoặc ưu tiên nguồn) và phải nhất quán.
- Nếu API CRM của một tenant bị lỗi/rate limit, job đồng bộ của tenant đó phải fail độc lập và retry theo backoff, không ảnh hưởng đến việc đồng bộ của các tenant khác đang chạy cùng lúc.
- Phát hiện và xử lý được bản ghi trùng lặp (duplicate) phát sinh do đồng bộ chạy lại nhiều lần hoặc do dữ liệu trùng ở nguồn, tránh tạo ra nhiều bản ghi khách hàng giống nhau trong hệ thống nội bộ.
- Cho phép tenant xem lịch sử đồng bộ (thành công/lỗi, số bản ghi đã xử lý) và chủ động trigger đồng bộ lại thủ công khi cần, không phải chờ chu kỳ định kỳ tiếp theo.

---

## Mạng xã hội tái cấu trúc database sang mô hình sharding

**Repository:** `etl-social-database-resharding`

**Hệ thống:** Một mạng xã hội có bảng user/post đã quá lớn cho một database đơn, cần chuyển sang kiến trúc sharding theo user_id.

**Vai trò của flow:** Di chuyển dữ liệu hiện có từ database đơn sang nhiều shard theo đúng logic phân vùng mới, trong khi hệ thống vẫn đang phục vụ traffic sống.

**Yêu cầu cụ thể:**
- Xác định đúng shard đích cho từng bản ghi dựa trên khóa phân vùng (user_id) và di chuyển dữ liệu liên quan (post, comment, like) của cùng một user vào cùng một shard để tránh cross-shard query sau này.
- Trong lúc migration đang chạy, các thao tác ghi mới (post mới, comment mới) từ user vẫn phải đi đúng vào shard đích ngay lập tức, không ghi vào hệ thống cũ rồi phải di chuyển lại.
- Phải test và xử lý được trường hợp một số quan hệ dữ liệu bắc cầu giữa hai user (ví dụ tin nhắn, tag nhau trong post) mà hai user đó rơi vào hai shard khác nhau.
- Có bước verify sau migration: đếm số bản ghi, checksum nội dung theo từng user giữa nguồn và từng shard đích, đảm bảo không có dữ liệu bị thất lạc hoặc trùng lặp.
- Migration phải chạy theo từng nhóm user (cohort) nhỏ, có thể theo dõi tiến độ và tạm dừng nếu phát hiện lỗi ở một cohort, không di chuyển toàn bộ user cùng lúc rồi mới phát hiện lỗi.

---

## Marketplace hợp nhất dữ liệu sản phẩm từ nhiều nguồn seller

**Repository:** `etl-marketplace-product-data-consolidation`

**Hệ thống:** Một marketplace cho phép seller đẩy dữ liệu sản phẩm (giá, tồn kho, mô tả) vào qua nhiều kênh khác nhau: file feed định kỳ, API, và nhập tay trên UI.

**Vai trò của flow:** Hợp nhất dữ liệu sản phẩm từ các nguồn khác nhau này thành một catalog trung tâm nhất quán, xử lý xung đột khi nhiều nguồn cùng cập nhật một sản phẩm.

**Yêu cầu cụ thể:**
- Pipeline phải chuẩn hóa dữ liệu từ các định dạng feed khác nhau (CSV, XML, JSON API) về cùng một schema catalog chung, xử lý được lỗi định dạng của từng dòng riêng lẻ mà không làm fail toàn bộ file feed.
- Khi cùng một sản phẩm được cập nhật từ nhiều nguồn gần cùng thời điểm (feed file và API cùng đẩy), phải có luật rõ ràng để xác định nguồn nào "thắng" theo từng loại trường dữ liệu (ví dụ giá theo nguồn A, tồn kho theo nguồn B).
- File feed lớn (hàng trăm nghìn dòng) phải được xử lý theo streaming/batch, không load toàn bộ vào memory, và phải báo cáo được tiến độ xử lý theo thời gian thực cho seller theo dõi.
- Phát hiện và cảnh báo các thay đổi bất thường (ví dụ giá giảm hơn 90% hoặc tồn kho nhảy vọt bất thường) trước khi áp dụng vào catalog sống, tránh lỗi dữ liệu từ seller ảnh hưởng ngay tới hiển thị cho khách mua.
- Giữ lại lịch sử thay đổi giá/tồn kho theo từng nguồn để phục vụ audit khi có tranh chấp giữa seller và marketplace về thời điểm cập nhật.

---

## Công ty bảo hiểm chạy batch ETL đêm để tính phí bảo hiểm

**Repository:** `etl-insurance-nightly-batch`

**Hệ thống:** Một nền tảng bảo hiểm trực tuyến, cần chạy batch ETL hàng đêm để tính lại premium (phí bảo hiểm) dựa trên dữ liệu rủi ro cập nhật từ nhiều nguồn (hồ sơ khách hàng, dữ liệu claim, dữ liệu thị trường).

**Vai trò của flow:** Tổng hợp dữ liệu từ nhiều nguồn nội bộ và bên ngoài mỗi đêm, tính toán lại premium cho các hợp đồng liên quan, và ghi kết quả vào hệ thống quản lý hợp đồng.

**Yêu cầu cụ thể:**
- Job ETL đêm phải là idempotent hoàn toàn: nếu job chạy lại (do lỗi, do vận hành trigger lại) trên cùng một ngày dữ liệu, kết quả tính premium phải giống hệt lần chạy trước, không tính cộng dồn hay tạo bản ghi trùng.
- Toàn bộ pipeline phải hoàn thành trong khung giờ bảo trì đêm cố định trước khi hệ thống mở lại phục vụ traffic ban ngày; nếu vượt thời gian cho phép phải có cảnh báo sớm và cơ chế chạy phần còn thiếu ưu tiên trước.
- Phải có bước đối soát: tổng số hợp đồng được tính lại phải khớp với tổng số hợp đồng active tại thời điểm chạy, phát hiện và báo cáo rõ các hợp đồng bị bỏ sót hoặc tính lỗi.
- Nếu một nguồn dữ liệu đầu vào (ví dụ dữ liệu claim) chưa sẵn sàng hoặc lỗi vào đêm đó, pipeline phải có chiến lược rõ ràng: dùng lại dữ liệu ngày trước, hoặc trì hoãn tính premium cho các hợp đồng liên quan — không được tính sai âm thầm.
- Mọi thay đổi premium phải lưu lại lịch sử tính toán đầy đủ (input đầu vào, công thức áp dụng, kết quả) để phục vụ giải trình khi khách hàng hoặc cơ quan quản lý yêu cầu kiểm tra.
