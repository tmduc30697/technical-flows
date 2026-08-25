# Real-time chat/presence flow (WebSocket connection scaling) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống real-time (chat 1-1/nhóm, ứng dụng cộng tác tài liệu, game multiplayer, hỗ trợ khách hàng live chat, livestream comment) để luyện việc quản lý kết nối WebSocket ở quy mô lớn, trạng thái presence (online/offline), và đảm bảo tin nhắn/sự kiện đến đúng, đúng thứ tự khi user có nhiều kết nối hoặc server bị scale ngang.

---

## Chat 1-1 và nhóm với user có nhiều thiết bị kết nối cùng lúc

**Repository:** `realtime-chat-multi-device-presence`

**Hệ thống:** Một app chat (kiểu Messenger/Zalo) cho phép user đăng nhập và mở nhiều thiết bị cùng lúc (điện thoại, web, desktop), mỗi thiết bị giữ 1 kết nối WebSocket riêng tới server.

**Vai trò của flow:** Flow phải đảm bảo tin nhắn gửi/nhận được đồng bộ đúng trên tất cả thiết bị của user, xử lý đúng trạng thái presence (online) khi user có nhiều kết nối và một số kết nối bị rớt.

**Yêu cầu cụ thể:**
- Khi user có 3 thiết bị đang kết nối và 1 thiết bị bị rớt mạng (mất kết nối WebSocket), trạng thái "online" của user chỉ được đổi thành "offline" khi TẤT CẢ kết nối đều mất — dùng bộ đếm số kết nối active atomic (tăng khi connect, giảm khi disconnect) thay vì cờ boolean đơn giản dễ bị ghi đè sai khi các connect/disconnect xảy ra gần như đồng thời từ các thiết bị khác nhau.
- Mô tả cụ thể race: thiết bị A disconnect đúng lúc thiết bị B (mới) đang connect cho cùng user, thứ tự event đến server có thể là "B connect" trước "A disconnect" hoặc ngược lại tùy độ trễ — bộ đếm kết nối phải xử lý đúng cả 2 thứ tự (tăng rồi giảm, hoặc giảm rồi tăng) và kết quả cuối cùng phải phản ánh đúng số kết nối thực tế đang có, không có race khiến bộ đếm bị lệch (ví dụ về số âm hoặc đếm dư).
- Khi gửi tin nhắn tới 1 user có nhiều thiết bị, phải gửi tới TẤT CẢ kết nối active tại thời điểm gửi, và nếu 1 kết nối vừa bị rớt ngay trước khi gửi (race giữa việc lấy danh sách kết nối và việc gửi), tin nhắn phải được lưu vào hàng đợi offline để thiết bị đó nhận lại khi kết nối lại, không được mất tin nhắn.
- Đảm bảo thứ tự tin nhắn hiển thị nhất quán trên tất cả thiết bị của user dù các thiết bị nhận WebSocket message ở các thời điểm hơi khác nhau — dùng sequence number hoặc timestamp server-side gắn vào mỗi tin nhắn để mỗi client tự sắp xếp đúng thứ tự khi hiển thị, không dựa vào thứ tự nhận qua socket.
- Khi user đang chat nhóm, 2 thành viên gửi tin nhắn gần như đồng thời tới cùng 1 nhóm — server phải gán thứ tự xác định (theo thời điểm server nhận, có sequence tăng dần theo nhóm) để tất cả thành viên khác trong nhóm nhìn thấy đúng cùng 1 thứ tự tin nhắn, không có 2 client hiển thị thứ tự khác nhau.

---

## Đồng bộ chỉnh sửa tài liệu cộng tác real-time (kiểu Google Docs)

**Repository:** `realtime-collaborative-document-editing`

**Hệ thống:** Một ứng dụng cộng tác tài liệu cho phép nhiều người chỉnh sửa cùng 1 văn bản đồng thời qua kết nối WebSocket, mỗi thay đổi (keystroke/đoạn chỉnh sửa) được gửi tới server và phát lại cho các người dùng khác.

**Vai trò của flow:** Flow phải đảm bảo các chỉnh sửa từ nhiều người cùng lúc được áp dụng đúng thứ tự và không mất/không đè lẫn nội dung của nhau, kể cả khi kết nối WebSocket của 1 người bị chập chờn.

**Yêu cầu cụ thể:**
- Khi 2 người cùng chỉnh sửa 1 đoạn văn bản gần như đồng thời (ví dụ cùng gõ chữ vào 2 vị trí gần nhau), server phải dùng cơ chế xử lý xung đột phù hợp (Operational Transformation hoặc CRDT) để áp dụng cả 2 thay đổi mà không làm mất nội dung của bên nào, mô tả rõ thứ tự transform áp dụng dựa trên sequence number từ server, không dựa vào thời điểm client gửi (độ trễ mạng khác nhau).
- Mô tả cụ thể: người A đang gõ liên tục (gửi nhiều event nhỏ) đúng lúc kết nối WebSocket của A bị chập chờn (rớt rồi kết nối lại trong vài giây) — các event đã gửi trước khi rớt phải được server xác nhận đã nhận đủ hay chưa (qua ack), và khi A kết nối lại, phải đồng bộ lại đúng trạng thái tài liệu hiện tại (đã bao gồm các thay đổi của người khác trong lúc A bị rớt) trước khi cho A tiếp tục gửi thay đổi mới, tránh A gửi thay đổi dựa trên phiên bản tài liệu đã lỗi thời.
- Nếu người B đang xem tài liệu (chỉ đọc, chưa kết nối realtime kịp thời do mới mở trang) nhận được 1 sự kiện cập nhật trước khi nhận được snapshot đầy đủ ban đầu của tài liệu, phải xử lý đúng thứ tự: đảm bảo snapshot ban đầu và các sự kiện cập nhật tiếp theo được áp dụng đúng theo sequence, không để sự kiện cập nhật đến trước rồi bị snapshot cũ hơn ghi đè mất.
- Khi server phải chuyển tài liệu đang cộng tác từ 1 instance sang instance khác (do scale/restart), toàn bộ kết nối WebSocket của người đang chỉnh sửa tài liệu đó phải được chuyển tiếp/kết nối lại một cách trong suốt, không làm mất thay đổi đang gửi giữa lúc chuyển đổi và không có 2 instance cùng xử lý tài liệu đó song song gây phân nhánh trạng thái.
- Có cơ chế lưu snapshot định kỳ (checkpoint) của tài liệu để nếu có lỗi đồng bộ nghiêm trọng phát hiện được, có thể phục hồi tới điểm checkpoint gần nhất và replay lại các thay đổi đã ghi log sau đó, không mất toàn bộ nội dung.

---

## Trạng thái presence và matchmaking trong game multiplayer real-time

**Repository:** `realtime-game-presence-matchmaking`

**Hệ thống:** Một game multiplayer online cho phép người chơi thấy bạn bè đang online, mời vào phòng chơi chung, mỗi người chơi giữ 1 kết nối WebSocket với server game.

**Vai trò của flow:** Flow phải quản lý đúng trạng thái presence (online/trong game/đang mời) và xử lý đúng khi nhiều người chơi cùng gửi lời mời vào phòng hoặc phòng bị đầy đúng lúc có lời mời đang chờ xử lý.

**Yêu cầu cụ thể:**
- Trạng thái presence của người chơi (online, đang chơi, đang chờ mời) phải được lưu và cập nhật atomic ngay khi có sự kiện connect/disconnect/vào phòng, và phải broadcast đúng cho danh sách bạn bè đang online của người đó với độ trễ tối thiểu, không để bạn bè thấy trạng thái cũ (ví dụ vẫn hiện "online" dù đã disconnect vài phút).
- Mô tả cụ thể: phòng chơi giới hạn 4 người, đang có 3 người, 2 lời mời được gửi tới 2 người khác nhau gần như đồng thời để lấp chỗ cuối cùng — chỉ đúng 1 người được xác nhận vào phòng theo transaction atomic kiểm tra số chỗ còn trống ngay tại thời điểm xác nhận (không phải tại thời điểm gửi lời mời), người thứ 2 phải nhận thông báo "phòng đã đủ người" ngay dù đã nhận lời mời trước đó.
- Khi người chơi mất kết nối WebSocket giữa trận đấu (rớt mạng tạm thời), quy định rõ khoảng thời gian ân hạn (grace period, ví dụ 30 giây) để họ có thể kết nối lại và tiếp tục trận đấu mà không bị coi là rời trận, và trong khoảng thời gian đó trạng thái phòng/trận đấu vẫn giữ nguyên chờ họ, không tự động điền người khác vào chỗ đó ngay lập tức.
- Nếu người chơi kết nối lại đúng lúc hệ thống vừa quyết định coi họ là "rời trận" (do vượt quá grace period vài milisecond), phải xử lý rõ ràng: kết nối lại chỉ được phép vào lại nếu request đến trước khi transaction đánh dấu "rời trận" hoàn tất, nếu sau thì phải xử lý như 1 người chơi mới xin vào (không tự động phá vỡ quyết định đã chốt của trận đấu).
- Khi server game cần scale ngang (thêm instance xử lý nhiều phòng chơi), phải đảm bảo mỗi phòng chơi chỉ được xử lý bởi đúng 1 instance tại một thời điểm (tránh 2 instance cùng xử lý logic trận đấu của 1 phòng gây trạng thái phân nhánh), và presence của người chơi phải nhất quán xuyên suốt các instance (không lưu presence cục bộ trong memory của riêng từng instance mà không đồng bộ).

---

## Live chat hỗ trợ khách hàng với hàng đợi phân công nhân viên real-time

**Repository:** `realtime-support-chat-agent-queue`

**Hệ thống:** Một nền tảng hỗ trợ khách hàng qua live chat, khách kết nối WebSocket vào hàng đợi chờ, hệ thống tự động phân công nhân viên hỗ trợ đang online và có ít việc nhất.

**Vai trò của flow:** Flow phải phân công đúng 1 nhân viên cho 1 khách tại một thời điểm, xử lý đúng khi nhiều khách vào hàng đợi cùng lúc và khi nhân viên online/offline đột ngột (mất kết nối) trong lúc đang được phân công.

**Yêu cầu cụ thể:**
- Việc phân công khách cho nhân viên phải atomic: kiểm tra nhân viên đang online và số lượng chat hiện tại chưa đạt giới hạn tối đa, rồi gán ngay trong 1 transaction, để 2 khách vào hàng đợi gần như đồng thời không cùng được phân công cho 1 nhân viên vượt quá giới hạn tối đa (ví dụ mỗi nhân viên tối đa 3 chat cùng lúc).
- Mô tả cụ thể: nhân viên đang xử lý 2/3 chat, có 2 khách mới vào hàng đợi gần như đồng thời và cả 2 đều đang được xem xét phân công cho nhân viên này (do cùng đọc số liệu "còn 1 chỗ") — chỉ đúng 1 khách được phân công thành công theo update atomic có điều kiện, khách còn lại phải được matching engine tìm nhân viên khác hoặc giữ trong hàng đợi.
- Nếu nhân viên mất kết nối WebSocket đột ngột (rớt mạng, đóng tab) giữa lúc đang chat với khách, phải phát hiện qua heartbeat/ping-pong trong khoảng thời gian ngắn (ví dụ 15 giây không có phản hồi) và tự động: đưa khách đó về hàng đợi ưu tiên cao (vì đã chờ trước đó) để phân công lại nhân viên khác ngay, không để khách chờ vô thời hạn không biết đang xảy ra chuyện gì.
- Nếu nhân viên kết nối lại ngay sau khi hệ thống đã bắt đầu tiến trình phân công lại khách cho người khác, quy định rõ khách đó vẫn thuộc về nhân viên mới nếu việc phân công lại đã hoàn tất (atomic, tránh 2 nhân viên cùng nghĩ mình đang phụ trách 1 khách), nhân viên cũ nhận thông báo rõ "khách đã được chuyển cho đồng nghiệp khác do mất kết nối".
- Khi hệ thống có nhiều server WebSocket (scale ngang) và khách/nhân viên có thể kết nối tới các server khác nhau, việc gửi tin nhắn giữa 2 bên phải đi qua 1 lớp message broker trung tâm (pub/sub) để đảm bảo tin nhắn đến đúng người nhận dù họ đang kết nối ở server nào, không giả định khách và nhân viên luôn ở cùng 1 server instance.

---

## Bình luận real-time trong livestream với lượng comment cực lớn đổ về cùng lúc

**Repository:** `realtime-livestream-comment-firehose`

**Hệ thống:** Một platform livestream (bán hàng/giải trí) hiển thị bình luận real-time từ người xem qua WebSocket, có thể có hàng nghìn bình luận/giây trong lúc livestream cao điểm.

**Vai trò của flow:** Flow phải broadcast bình luận tới toàn bộ người xem đang kết nối với độ trễ thấp, xử lý đúng khi lượng bình luận vượt quá khả năng xử lý real-time mà không làm server WebSocket sập hoặc mất kết nối hàng loạt của người xem.

**Yêu cầu cụ thể:**
- Vì broadcast tới hàng chục nghìn người xem cùng lúc cho mỗi bình luận là tốn kém, phải thiết kế cơ chế fan-out hiệu quả (ví dụ qua các node WebSocket trung gian theo cụm người xem, không phải 1 server trung tâm gửi trực tiếp tới từng kết nối) để độ trễ không tăng tuyến tính theo số người xem.
- Mô tả cụ thể tình huống quá tải: lượng bình luận vượt ngưỡng xử lý real-time (ví dụ 5000 bình luận/giây trong khi hệ thống chỉ xử lý được 2000/giây) — quy định chính sách rõ ràng khi quá tải: có thể lấy mẫu (sampling) hiển thị 1 phần bình luận thay vì toàn bộ, hoặc dồn lại gửi theo batch ngắn (ví dụ mỗi 200ms gửi 1 lô), miễn là không làm nghẽn hoàn toàn kết nối WebSocket của người xem.
- Khi 1 node WebSocket trung gian bị quá tải và cần một số kết nối người xem chuyển sang node khác (rebalance), quá trình chuyển đổi phải trong suốt với người xem (không mất kết nối/không bị gián đoạn hiển thị bình luận quá lâu), và không có kết nối nào bị gán trùng vào 2 node cùng lúc gây nhận trùng bình luận.
- Nếu người xem gửi bình luận đúng lúc bị rate-limit (chống spam, ví dụ tối đa 1 bình luận/3 giây/người), phải phản hồi rõ ràng ngay cho client (không gửi bình luận đó đi) thay vì âm thầm drop khiến người xem không hiểu vì sao bình luận không hiển thị.
- Có cơ chế ưu tiên hiển thị cho một số loại bình luận đặc biệt (ví dụ câu hỏi được ghim, bình luận từ người mua đã xác nhận) không bị lẫn/trôi mất trong dòng bình luận tốc độ cao, đảm bảo các bình luận này luôn được broadcast dù đang trong chế độ giảm tải/sampling.
