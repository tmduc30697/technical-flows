# Enhance sequence — Xem audit log (cách ly theo tenant, kể cả với system admin)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — ở base audit log được ghi nhưng chưa có flow/cơ chế nào giới hạn việc xem log theo tenant. Đáp ứng yêu cầu 4 của đề bài: audit log chi tiết, tách biệt theo tenant, và không cho tenant khác — kể cả admin hệ thống ở mức thông thường — xem được audit log của tenant khác.

```mermaid
sequenceDiagram
    actor TenantAdmin as Admin phòng khám (tenant A)
    actor SysAdmin as System Admin (mức thông thường)
    participant App as Audit Log Service
    participant Log as AUDIT_LOG (partition riêng theo tenant)

    TenantAdmin->>App: Xem audit log của tenant A
    App->>App: Kiểm tra scope, tenant_id trong token = A
    App->>Log: Truy vấn partition riêng của tenant A
    Log-->>App: Danh sách log (ai xem, khi nào, hồ sơ nào)
    App-->>TenantAdmin: Trả về audit log của chính tenant A

    SysAdmin->>App: Yêu cầu xem audit log của tenant B (không phải tenant mình quản lý)
    App->>App: Kiểm tra quyền, system_admin mức thông thường không có scope cross-tenant
    App-->>SysAdmin: Từ chối truy cập, ghi nhận đây là một lần thử truy cập trái phép
    Note over App: Chỉ vai trò đặc quyền, có phê duyệt riêng và tự nó cũng bị audit mới được xem log xuyên tenant, không áp dụng cho system admin thông thường
```
