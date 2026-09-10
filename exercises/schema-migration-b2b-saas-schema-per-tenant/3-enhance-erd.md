# ERD — Enhance (sau khi có schema-per-tenant migration)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — routing table theo tenant, dual-write có xử lý thất bại một bên, đọc luôn từ schema nguồn sự thật, và rollback nhanh sau cutover. So với base, dữ liệu nghiệp vụ (`EMPLOYEE`, `SALARY_RECORD`) nay tồn tại tiềm năng ở CẢ HAI schema (`old_schema` dùng chung, `new_schema` riêng theo tenant) trong lúc migrate, và `TENANT_ROUTING` là nguồn quyết định duy nhất schema nào đang là "nguồn sự thật".

```mermaid
erDiagram
    TENANT ||--|| TENANT_ROUTING : "routed via"
    TENANT ||--o{ EMPLOYEE : "owns (in current source-of-truth schema)"
    TENANT ||--o{ SALARY_RECORD : "owns (in current source-of-truth schema)"
    EMPLOYEE ||--o{ SALARY_RECORD : has

    TENANT_ROUTING ||--o{ DUAL_WRITE_LOG : "generates during migrating state"
    TENANT_ROUTING ||--o{ MIGRATION_JOB : "tracked by"
    MIGRATION_JOB ||--o| CUTOVER_LOCK : "acquires during cutover"

    TENANT {
        string tenant_id PK
        string company_name
        string status
    }

    TENANT_ROUTING {
        string tenant_id PK "FK to TENANT"
        string schema_state
        string old_schema_ref
        string new_schema_ref
        datetime updated_at
    }

    EMPLOYEE {
        string employee_id PK
        string tenant_id FK
        string full_name
        string position
        string status
    }

    SALARY_RECORD {
        string salary_record_id PK
        string tenant_id FK
        string employee_id FK
        decimal amount
        date effective_date
    }

    MIGRATION_JOB {
        string job_id PK
        string tenant_id FK
        string status
        datetime started_at
        datetime cutover_at
        datetime completed_at
    }

    CUTOVER_LOCK {
        string lock_id PK
        string tenant_id FK
        string job_id FK
        datetime acquired_at
        datetime released_at
    }

    DUAL_WRITE_LOG {
        string log_id PK
        string tenant_id FK
        string operation
        string old_schema_status
        string new_schema_status
        string resolution
        datetime created_at
    }
```
