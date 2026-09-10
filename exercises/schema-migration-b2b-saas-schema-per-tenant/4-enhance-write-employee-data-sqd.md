# Sequence Diagram — Enhance: Write Employee Data (Routed)

Đây là **enhance**, flow ghi dữ liệu nhân sự đã có ở base ([2-base-write-employee-data-sqd.md](2-base-write-employee-data-sqd.md)) nhưng nay thay đổi căn bản: mọi request phải tra `TENANT_ROUTING` trước khi ghi, không hardcode schema trong code app. Flow này cho thấy 3 nhánh khác nhau tùy `schema_state` của tenant.

```mermaid
sequenceDiagram
    actor HRUser as HR User
    participant App as HR App Service
    participant Routing as Tenant Routing Table
    participant OldDB as Old Schema (shared)
    participant NewDB as New Schema (per-tenant)

    HRUser->>App: Update employee salary (tenant_id, employee_id, new_amount)
    App->>Routing: Look up schema_state for tenant_id

    alt schema_state = old
        Routing-->>App: Route to old_schema
        App->>OldDB: UPDATE SALARY_RECORD WHERE tenant_id=? AND employee_id=?
        OldDB-->>App: Row updated
    else schema_state = migrating
        Routing-->>App: Route to dual-write, old_schema is still source of truth
        App->>OldDB: Write update
        App->>NewDB: Write same update
        Note over App: See dual-write flow for failure handling detail
    else schema_state = new
        Routing-->>App: Route to new_schema
        App->>NewDB: UPDATE SALARY_RECORD WHERE employee_id=?
        NewDB-->>App: Row updated
    end

    App-->>HRUser: Salary updated
```
