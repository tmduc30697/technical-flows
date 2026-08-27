# Optimistic vs pessimistic locking flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống web khác nhau (quản lý nội dung/CMS, quản lý dự án cộng tác, chỉnh sửa hồ sơ khách hàng trong CRM, quản lý kho, chỉnh sửa cấu hình hệ thống nội bộ) để luyện việc chọn đúng giữa optimistic locking (version check) và pessimistic locking (khóa trước khi đọc/sửa) tùy theo đặc thù tần suất xung đột và trải nghiệm cần có.

---

## Chỉnh sửa bài viết CMS bởi nhiều biên tập viên

**Repository:** `locking-cms-concurrent-editing`

**Hệ thống:** Một CMS quản lý nội dung cho tòa soạn báo online, nhiều biên tập viên có thể cùng mở và sửa 1 bài viết (ít xảy ra nhưng không hiếm, ví dụ 2 biên tập viên cùng ca trực).

**Vai trò của flow:** Vì xung đột sửa đồng thời hiếm xảy ra nhưng cần trải nghiệm mượt (không muốn khóa bài viết chặn người khác chỉ vì 1 người đang mở xem), flow lưu bài viết nên dùng optimistic locking dựa trên version, phát hiện xung đột tại thời điểm lưu thay vì chặn ngay từ khi mở bài.

**Yêu cầu cụ thể:**
- Mỗi bài viết có cột `version` tăng dần mỗi lần lưu; khi biên tập viên A mở bài ở version 5, sửa nội dung rồi lưu, request lưu phải gửi kèm version 5 đã đọc và server chỉ chấp nhận nếu version hiện tại trong DB vẫn là 5 (update kèm điều kiện `WHERE version = 5`), nếu không khớp phải trả lỗi conflict rõ ràng, không âm thầm ghi đè.
- Mô tả cụ thể: biên tập viên A và B cùng mở bài ở version 5, B lưu trước (bài chuyển sang version 6), A lưu sau vẫn gửi version 5 — request của A phải bị từ chối với thông báo "bài viết đã được người khác cập nhật", kèm hiển thị nội dung mới nhất (version 6) để A quyết định merge tay hoặc ghi đè có ý thức, không tự động merge ngầm.
- Cho phép A chọn "lưu đè" sau khi đã xem nội dung mới của B (ghi đè có ý thức, không phải ghi đè mù), hành động này phải tạo version mới dựa trên version hiện tại thực (6 → 7), không dùng lại version cũ (5) đã lỗi thời.
- Tự động lưu draft định kỳ (auto-save) trong lúc biên tập viên đang soạn không được tính là 1 "lưu chính thức" làm tăng version theo cách gây xung đột giả với người khác — quy định rõ auto-save lưu vào 1 bản riêng (draft riêng theo user) tách khỏi bản chính thức, chỉ khi publish mới áp dụng optimistic lock lên bản chính thức.
- Ghi log lịch sử các version bài viết (ai sửa, khi nào, nội dung gì thay đổi) để biên tập viên có thể xem lại/khôi phục version cũ nếu bị lưu đè nhầm.

---

## Chỉnh sửa kế hoạch dự án (Gantt chart) trong tool quản lý dự án cộng tác

**Repository:** `locking-project-management-gantt-collaborative`

**Hệ thống:** Một tool quản lý dự án (kiểu Asana/Jira) cho phép nhiều thành viên cùng chỉnh sửa lịch trình task, gán người phụ trách, đổi ngày deadline trên cùng 1 dự án.

**Vai trò của flow:** Vì các thành viên thường sửa các task khác nhau (ít đụng cùng 1 task) nhưng đôi khi 2 người cùng sửa 1 task, flow nên dùng optimistic locking ở cấp độ từng task (không lock cả dự án) để tối đa hóa khả năng làm việc song song.

**Yêu cầu cụ thể:**
- Lock optimistic phải áp dụng ở cấp độ từng task (mỗi task có version riêng), không phải ở cấp độ toàn dự án — để thành viên A sửa task 1 và thành viên B sửa task 2 không bị chặn/xung đột lẫn nhau dù đang cùng mở dự án.
- Mô tả cụ thể: thành viên A đổi deadline task X từ ngày 10 sang ngày 15 (đọc task ở version 3), đồng thời thành viên B gán thêm người phụ trách cho cùng task X (cũng đọc ở version 3) — cả 2 đều sửa các trường khác nhau nhưng cùng version, ai lưu trước sẽ thắng và tăng version, người lưu sau bị từ chối theo optimistic lock thông thường; bàn về việc có nên áp dụng field-level merge (merge các trường không trùng nhau) thay vì từ chối toàn bộ, và ưu/nhược điểm của việc đó.
- Khi task đang được kéo-thả để đổi thứ tự trên Gantt chart (thao tác kéo thả có thể gửi nhiều update nhỏ liên tiếp trong thời gian ngắn), cần quy định rõ có nên coi mỗi lần kéo-thả là 1 version riêng (dễ tạo conflict giả nếu 2 người cùng kéo-thả các task khác nhau ảnh hưởng tới thứ tự chung) hay dùng cơ chế khác (ví dụ optimistic lock ở cấp "toàn bộ thứ tự danh sách task" thay vì từng task riêng khi liên quan tới sắp xếp).
- Nếu task bị xóa bởi 1 thành viên đúng lúc thành viên khác đang sửa task đó (gửi update dựa trên version cũ trước khi bị xóa), request update phải nhận lỗi rõ ràng "task đã bị xóa", không tạo lại task đã xóa hoặc gây lỗi không rõ nguyên nhân.
- Với các trường có tần suất tranh chấp cao hơn (ví dụ % hoàn thành task được nhiều người cùng cập nhật liên tục), đánh giá xem optimistic lock có phù hợp hay nên chuyển sang cơ chế khác (ví dụ increment atomic riêng cho trường đó) để giảm tỷ lệ conflict phải retry liên tục gây trải nghiệm khó chịu.

---

## Chỉnh sửa cấu hình hệ thống nội bộ (feature flag, config) bởi nhiều kỹ sư

**Repository:** `locking-internal-config-multi-engineer`

**Hệ thống:** Một dashboard nội bộ cho kỹ sư quản lý feature flag/config runtime của hệ thống production, thay đổi có hiệu lực ngay khi lưu, nhiều kỹ sư ở các team khác nhau có quyền chỉnh sửa.

**Vai trò của flow:** Vì thay đổi config có tác động ngay tới production và tần suất nhiều kỹ sư cùng sửa 1 config trong cùng thời điểm là thấp nhưng hậu quả nếu ghi đè nhầm khá nghiêm trọng, flow nên dùng optimistic locking kết hợp với hiển thị rõ diff thay đổi trước khi cho phép lưu đè.

**Yêu cầu cụ thể:**
- Mỗi config có version, khi kỹ sư A lưu thay đổi dựa trên version cũ mà đã có kỹ sư B lưu thay đổi khác trước đó (version mới hơn), hệ thống phải từ chối lưu và hiển thị rõ diff giữa bản A đang muốn lưu, bản gốc A đã đọc, và bản mới nhất B đã lưu — để A tự quyết định merge tay chính xác, không tự động merge ngầm với config nhạy cảm ảnh hưởng production.
- Mô tả cụ thể: kỹ sư A đang tắt feature flag X (do phát hiện lỗi khẩn cấp), đúng lúc kỹ sư B đang sửa 1 config khác không liên quan trong cùng bộ config đó nhưng version chung bị tăng do B lưu trước — cần đánh giá xem có nên tách optimistic lock ở cấp độ từng flag/config riêng lẻ (để A không bị chặn bởi thay đổi không liên quan của B) thay vì version chung cho cả nhóm config, đặc biệt quan trọng cho action khẩn cấp như tắt flag khi có sự cố.
- Với action khẩn cấp (ví dụ "tắt flag ngay" trong sự cố production), nên có đường xử lý riêng ưu tiên ghi đè ngay không chờ optimistic lock thông thường (bỏ qua version check hoặc có nút "buộc lưu" rõ ràng có cảnh báo), vì trong tình huống khẩn cấp việc bị chặn bởi conflict version có thể gây chậm trễ nguy hiểm hơn rủi ro ghi đè nhầm — nhưng phải log rõ ràng và cảnh báo action này đã bỏ qua kiểm tra xung đột.
- Mọi thay đổi config (dù qua optimistic lock thông thường hay đường khẩn cấp) phải được ghi lịch sử đầy đủ và không thể xóa (audit log immutable): ai, khi nào, giá trị trước/sau, có phải action khẩn cấp không — để phục vụ điều tra sau sự cố nếu 1 thay đổi config gây ảnh hưởng không mong muốn.
- Có cơ chế thông báo real-time cho các kỹ sư khác đang mở màn hình chỉnh sửa cùng 1 config khi có ai đó vừa lưu thay đổi (qua WebSocket/polling), để giảm khả năng họ submit dựa trên version đã lỗi thời và phải xử lý conflict muộn hơn cần thiết.

---

## Sale và chăm sóc khách hàng cùng sửa hồ sơ 1 khách hàng trên CRM

**Repository:** `locking-crm-customer-profile-concurrent-edit`

**Hệ thống:** Một hệ thống CRM quản lý khách hàng cho đội sale và đội chăm sóc khách hàng (CSKH), nhân viên sale cập nhật giai đoạn deal (pipeline stage), nhân viên CSKH cập nhật lịch sử liên hệ/ghi chú, cả 2 nhóm đều có thể mở cùng 1 hồ sơ khách hàng bất kỳ lúc nào.

**Vai trò của flow:** Vì tần suất 2 nhân viên cùng mở đúng 1 hồ sơ tại cùng thời điểm là thấp nhưng hồ sơ khách hàng có nhiều nhóm trường thuộc các nghiệp vụ khác nhau (sale, CSKH, thông tin liên hệ chung), flow lưu hồ sơ cần chọn chiến lược lock cân bằng giữa việc không mất dữ liệu do ghi đè và không gây khó chịu khi nhân viên bị từ chối lưu vì xung đột ở trường hoàn toàn không liên quan tới phần họ đang sửa.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: nhân viên sale mở hồ sơ khách hàng C ở version 8, cập nhật giai đoạn deal từ "đang đàm phán" sang "chốt hợp đồng"; gần như cùng lúc nhân viên CSKH cũng mở hồ sơ C ở version 8, thêm 1 ghi chú cuộc gọi vừa thực hiện — nếu dùng optimistic lock ở cấp toàn bộ record, ai lưu sau sẽ bị từ chối dù 2 người sửa 2 nhóm trường hoàn toàn khác nhau; đánh giá phương án tách optimistic lock theo nhóm trường (version riêng cho nhóm "sale", nhóm "CSKH", nhóm "thông tin chung") để loại bỏ các xung đột giả kiểu này.
- Với trường hợp thực sự xung đột trên cùng 1 trường (ví dụ cả sale và CSKH cùng sửa số điện thoại liên hệ chính của khách hàng gần như đồng thời do phát hiện thông tin cũ sai), quy định rõ optimistic lock ở đúng cấp nhóm trường đó phải từ chối người lưu sau và hiển thị giá trị mới nhất vừa được lưu, không tự động merge 2 số điện thoại khác nhau thành 1 giá trị mơ hồ.
- Mô tả tình huống cần pessimistic lock: khi đang trong quy trình chuyển giao khách hàng từ nhân viên sale này sang nhân viên sale khác, cần khóa cứng toàn bộ hồ sơ trong vài giây để đảm bảo không ai (kể cả CSKH) sửa bất kỳ trường nào giữa chừng quá trình chuyển giao — quy định cơ chế khóa tường minh có timeout ngắn cho thao tác đặc biệt này, tách biệt rõ với cơ chế optimistic mặc định dùng cho các thao tác sửa thông thường hàng ngày.
- Khi nhân viên bị từ chối lưu do conflict, giao diện phải chỉ rõ đúng trường/nhóm trường nào đã bị người khác thay đổi (không coi toàn bộ hồ sơ là "đã cũ"), để họ chỉ cần xác nhận lại đúng phần liên quan thay vì phải đọc lại và nhập lại toàn bộ nội dung đã soạn.
- Mọi thay đổi hồ sơ khách hàng (theo từng nhóm trường) phải ghi lịch sử đầy đủ ai sửa, khi nào, giá trị trước/sau, để xử lý khiếu nại khi có tranh chấp nội bộ về việc ai đã cập nhật sai thông tin khách hàng dẫn đến hậu quả nghiệp vụ.

---

## Kiểm kho định kỳ diễn ra song song với các giao dịch xuất/nhập kho khác

**Repository:** `locking-warehouse-stocktake-concurrent-transactions`

**Hệ thống:** Một hệ thống quản lý kho cho chuỗi cửa hàng/nhà phân phối, định kỳ nhân viên thực hiện kiểm kho (đếm và đối chiếu số lượng thực tế) tại 1 khu vực kho, trong khi các giao dịch xuất/nhập kho khác của khu vực đó hoặc khu vực lân cận vẫn có thể đang chạy bình thường.

**Vai trò của flow:** Flow kiểm kho phải chọn giữa khóa cứng (pessimistic) khu vực đang kiểm để đảm bảo số đếm chính xác tuyệt đối, hay dùng version check (optimistic) để không chặn hoạt động kho, tùy theo mức độ chấp nhận gián đoạn vận hành trong lúc kiểm.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: nhân viên bắt đầu kiểm kho khu vực A lúc 9:00, đếm được SKU X có 50 đơn vị; đúng lúc đó có phiếu xuất kho 5 đơn vị SKU X cho 1 đơn giao hàng đang được xử lý song song ở hệ thống — nếu không khóa khu vực trong lúc kiểm kho, số đếm 50 sẽ lệch so với số hệ thống ghi nhận ngay sau đó (45), gây báo cáo chênh lệch giả không phải do thất thoát thực; đánh giá phương án khóa cứng khu vực A (chặn mọi giao dịch xuất/nhập trong lúc đếm) so với phương án ghi nhận "thời điểm bắt đầu đếm" rồi đối chiếu với log các giao dịch xảy ra trong lúc đếm để tính lại số kỳ vọng đúng.
- Nếu chọn khóa cứng, phải giới hạn phạm vi khóa đúng ở khu vực/kệ hàng đang kiểm (không khóa toàn kho), và quy định thời gian tối đa cho phép giữ khóa (ví dụ không quá 30 phút cho 1 khu vực), kèm cơ chế cảnh báo nếu nhân viên kiểm kho vượt thời gian quy định, để không chặn vận hành kho quá lâu.
- Mô tả cụ thể trường hợp 2 nhân viên kiểm kho cùng lúc ở 2 khu vực có SKU trùng nhau (cùng loại hàng lưu ở 2 kệ khác nhau nhưng thuộc cùng SKU tổng) — quy định rõ ranh giới khóa nên theo vị trí vật lý (kệ/khu vực) hay theo SKU tổng, và hệ quả nếu chọn sai (khóa theo SKU tổng sẽ chặn lẫn nhau dù 2 nhân viên đang đếm 2 vị trí hoàn toàn khác nhau, gây mất thời gian chờ không cần thiết).
- Mô tả race khi nhân viên nhập kết quả kiểm kho (điều chỉnh số lượng thực tế) đúng lúc có giao dịch nhập kho mới về khu vực đó vừa hoàn tất (hàng mới về ngay trong lúc đang chờ nhân viên submit kết quả đếm) — dùng version check để phát hiện số liệu tồn kho đã thay đổi kể từ lúc bắt đầu đếm, từ chối áp trực tiếp kết quả điều chỉnh mà yêu cầu nhân viên xác nhận lại phần chênh lệch trước khi ghi đè số liệu cuối cùng.
- Mọi điều chỉnh số lượng từ kiểm kho phải ghi rõ số liệu trước khi kiểm, số liệu đếm thực tế, số liệu sau điều chỉnh, và danh sách các giao dịch xảy ra trong lúc kiểm đã được tính vào phần chênh lệch, để có thể truy vết khi phát hiện điều chỉnh sai sau này.
