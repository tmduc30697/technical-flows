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

## Cập nhật hồ sơ khách hàng trong CRM khi sales và CS cùng thao tác

**Repository:** `locking-crm-customer-record-concurrent`

**Hệ thống:** Một CRM cho phép nhân viên sales và nhân viên customer support (CS) cùng truy cập và chỉnh sửa hồ sơ 1 khách hàng (thông tin liên hệ, ghi chú, trạng thái deal), 2 nhóm này thường thao tác gần như đồng thời khi có sự kiện (khách hàng gọi vào đúng lúc sales đang cập nhật deal).

**Vai trò của flow:** Vì tần suất 2 nhóm cùng sửa 1 hồ sơ trong cùng khung giờ khá cao ở các khách hàng lớn, cần đánh giá kỹ giữa optimistic (rủi ro conflict thường xuyên gây khó chịu) và pessimistic locking (khóa hồ sơ khi 1 người đang sửa) cho từng loại thao tác cụ thể.

**Yêu cầu cụ thể:**
- Với các trường ít xung đột (ví dụ ghi chú tự do, mỗi người thêm ghi chú riêng không đè lên nhau), dùng append-only (thêm dòng ghi chú mới, không phải sửa đè 1 trường chung) để tránh hoàn toàn nhu cầu locking cho phần này.
- Với trường có xung đột thực sự (ví dụ trạng thái deal: sales đổi từ "đang thương lượng" sang "chốt deal" đúng lúc CS đổi sang "khách hủy" do vừa nhận được thông tin khách không mua nữa), mô tả cụ thể lý do nên dùng pessimistic locking ngắn hạn (lock hồ sơ trong vài giây khi đang submit thay đổi trạng thái) thay vì optimistic, vì đây là trường quan trọng cần chắc chắn đúng ngay, không muốn 1 bên phải retry sau khi conflict trong lúc đang xử lý khách hàng thực.
- Mô tả cụ thể triển khai pessimistic lock: khi sales bắt đầu sửa trạng thái deal, hệ thống lock dòng hồ sơ đó (ví dụ `SELECT ... FOR UPDATE` trong transaction ngắn, hoặc advisory lock có TTL để tránh treo vô hạn nếu client rớt kết nối giữa lúc đang giữ lock), CS thao tác cùng lúc phải nhận thông báo "đang được người khác cập nhật, vui lòng thử lại sau vài giây" thay vì chờ vô thời hạn.
- Quy định rõ TTL tối đa cho pessimistic lock (ví dụ 10 giây) để nếu client giữ lock bị crash/rớt mạng giữa lúc đang sửa, lock phải tự động được giải phóng sau TTL, không khóa hồ sơ vĩnh viễn.
- Với thông tin liên hệ cơ bản (email, số điện thoại) ít bị 2 người cùng sửa đồng thời, dùng optimistic locking thông thường là đủ — yêu cầu người thiết kế phải phân loại rõ từng trường/nhóm trường trong hồ sơ khách hàng theo chiến lược locking phù hợp (không áp dụng 1 chiến lược chung cho toàn bộ hồ sơ), và giải thích rõ tiêu chí phân loại (tần suất xung đột, mức độ nghiêm trọng nếu conflict xảy ra).

---

## Cập nhật tồn kho thủ công bởi nhiều nhân viên kho trong hệ thống quản lý kho

**Repository:** `locking-warehouse-manual-inventory-update`

**Hệ thống:** Một hệ thống quản lý kho cho phép nhiều nhân viên kho cùng thao tác kiểm kho/nhập-xuất hàng qua thiết bị quét mã, thường xảy ra tình huống 2 nhân viên cùng quét mã 1 sản phẩm gần như đồng thời ở 2 khu vực khác nhau.

**Vai trò của flow:** Vì tồn kho là số liệu cần chính xác tuyệt đối và tần suất 2 nhân viên cùng động vào 1 SKU khá thường xuyên trong giờ kiểm kho cao điểm, flow cập nhật tồn kho nên dùng pessimistic locking (hoặc update atomic không cần đọc trước) thay vì optimistic để tránh retry liên tục làm chậm công việc thủ công của nhân viên.

**Yêu cầu cụ thể:**
- Với thao tác nhập/xuất kho theo số lượng (cộng/trừ, không phải set giá trị tuyệt đối), nên dùng update atomic dạng `UPDATE stock SET qty = qty + delta` thay vì đọc số lượng hiện tại rồi tính rồi ghi đè — cách này tránh hoàn toàn nhu cầu optimistic hay pessimistic lock cho trường hợp cộng/trừ đơn giản, vì phép cộng có tính giao hoán, thứ tự xử lý không ảnh hưởng kết quả cuối.
- Mô tả cụ thể tình huống CẦN pessimistic lock thật: nhân viên thực hiện "kiểm kho lại toàn bộ" (set giá trị tuyệt đối dựa trên đếm tay thực tế) cho 1 SKU, cần lock SKU đó trong lúc đang đếm và ghi nhận để không có giao dịch nhập/xuất khác chen vào giữa lúc đang kiểm kho gây sai số liệu cuối — mô tả rõ cách lock (khóa theo SKU, có thông báo cho các thiết bị quét khác đang cố thao tác vào SKU đó biết "đang kiểm kho, vui lòng đợi").
- Nếu nhân viên đang giữ pessimistic lock để kiểm kho nhưng thiết bị quét mã bị mất kết nối mạng (phổ biến trong kho có sóng yếu), lock phải có TTL và tự giải phóng sau khoảng thời gian hợp lý (ví dụ 2 phút), không được để khóa treo cả SKU khiến nhân viên khác không thể nhập/xuất hàng bình thường trong khi nhân viên đầu đã rời khỏi khu vực.
- Khi 2 nhân viên cùng cố bắt đầu kiểm kho 1 SKU gần như đồng thời (cả 2 đều bấm "Bắt đầu kiểm kho"), chỉ đúng 1 người giữ được lock, người thứ 2 phải nhận thông báo rõ ràng ngay (không phải chờ timeout mới biết) là "SKU này đang được đồng nghiệp X kiểm kho".
- Có log đầy đủ lịch sử thay đổi tồn kho (ai, khi nào, delta bao nhiêu, loại thao tác gì: nhập/xuất/kiểm kho) để có thể truy vết và tái tính tồn kho từ lịch sử khi phát hiện số liệu bất thường, không phụ thuộc hoàn toàn vào giá trị hiện tại trong 1 cột duy nhất.

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
