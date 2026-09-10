# Sequence Diagram — Enhance: Search User Autocomplete

Đây là **enhance**, flow hoàn toàn mới: autocomplete tìm người dùng cá nhân hóa theo mối quan hệ follow của người tìm, ưu tiên bạn bè/người đang follow lên trước người lạ cùng tên thay vì chỉ xếp theo độ phổ biến chung.

```mermaid
sequenceDiagram
    actor Searcher as Người tìm kiếm
    participant WebApp as Social App
    participant Search as Search Service
    participant FollowGraph as Follow Graph Store
    participant Index as User Search Index

    Searcher->>WebApp: Gõ tên người dùng cần tìm
    WebApp->>Search: Autocomplete request (partial name, searcher_user_id)

    Search->>FollowGraph: Lấy danh sách đang follow của searcher
    FollowGraph-->>Search: Danh sách followed_user_id

    Search->>Index: Query theo tên khớp, kèm danh sách followed_user_id để boost điểm
    Index-->>Search: Kết quả, người trong danh sách follow được boost lên trước

    Search-->>WebApp: Gợi ý đã cá nhân hóa
    WebApp-->>Searcher: Hiển thị bạn bè/người đang follow cùng tên lên đầu
```
