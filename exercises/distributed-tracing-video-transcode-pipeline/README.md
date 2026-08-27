# Pipeline xử lý media (transcode video) nhiều bước bất đồng bộ

**Hệ thống:** Một dịch vụ cho phép người dùng upload video, sau đó video được xử lý qua nhiều bước bất đồng bộ: kiểm duyệt nội dung, transcode nhiều độ phân giải, tạo thumbnail, rồi publish.

**Vai trò của flow:** Vì các bước chạy trên nhiều worker khác nhau và có thể mất từ vài giây tới vài phút, tracing giúp biết một video đang ở bước nào và bước nào đang là nút cổ chai chung của toàn hệ thống.

**Yêu cầu cụ thể:**
- Toàn bộ các job xử lý (kiểm duyệt, transcode từng độ phân giải, tạo thumbnail) của một video phải liên kết vào cùng một trace gốc dù chạy trên các worker pool khác nhau và có thể chạy song song.
- Với các bước chạy song song (ví dụ transcode 3 độ phân giải cùng lúc), trace phải thể hiện đúng quan hệ song song (sibling span), không xếp chúng thành tuần tự sai lệch thời gian thực tế.
- Khi một bước bị lỗi và job được retry tự động sau một khoảng thời gian (backoff), span của lần retry phải ghi rõ thời gian chờ (đã cố ý delay) tách biệt với thời gian xử lý thật, để không tính sai vào "thời gian xử lý trung bình".
- Cung cấp khả năng tổng hợp theo từng bước trên toàn hệ thống (không chỉ theo một video) để trả lời "bước transcode 1080p đang chậm dần theo thời gian, có phải do tài nguyên worker đang thiếu".
- Xử lý trường hợp một video bị người dùng xóa giữa lúc đang xử lý — các span phát sinh sau đó (ví dụ job vẫn đang chạy trong queue) phải được đóng lại rõ ràng với trạng thái "cancelled", không để trace treo ở trạng thái "đang xử lý" mãi mãi trên dashboard giám sát.
