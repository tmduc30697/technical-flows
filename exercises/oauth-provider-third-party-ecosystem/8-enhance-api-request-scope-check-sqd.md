# Enhance sequence — API request scope check

Đây là **enhance**, flow hoàn toàn mới so với base — app thứ ba dùng access token gọi vào cùng Resource API mà trước đây chỉ chủ shop truy cập qua session (xem `3-base-view-shop-data-sqd.md`). Resource server giờ phải kiểm tra access token có đủ scope cần thiết cho từng endpoint, chặn request nếu thiếu scope, dù token vẫn còn hạn.

```mermaid
sequenceDiagram
    participant App as Third-party App
    participant API as Resource API
    participant DB as Database

    App->>API: POST /api/inventory/123 { quantity: 50 } (Authorization Bearer access_token)
    API->>DB: Tìm ACCESS_TOKEN theo token_hash, kiểm tra chưa hết hạn và chưa bị revoke
    alt token không hợp lệ hoặc đã hết hạn
        API-->>App: 401 invalid_token
    else token hợp lệ
        API->>API: Kiểm tra scopes của token có write_inventory không
        alt thiếu scope write_inventory
            API-->>App: 403 insufficient_scope
        else đủ scope
            API->>DB: UPDATE inventory_item SET quantity = 50 WHERE shop_id = :shop_id_cua_token
            DB-->>API: xác nhận cập nhật
            API-->>App: 200 cập nhật thành công
        end
    end
    Note over API,DB: shop_id luôn lấy từ APP_GRANT gắn với token, app không thể tự truyền shop_id khác để truy cập chéo shop
```
