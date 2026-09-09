# Enhance sequence — Báo cáo tổng hợp chi phí lương (chặn suy luận ngược khi phòng ban quá nhỏ)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chưa có báo cáo tổng hợp nào ở tầng công ty. Đáp ứng yêu cầu 3 của đề bài: báo cáo lương trung bình theo phòng ban cho CFO không được vô tình lộ đúng lương của một cá nhân khi phòng ban đó chỉ có 1 người.

```mermaid
sequenceDiagram
    actor CFO as CFO công ty A
    participant App as Payroll Reporting Service
    participant DB as SALARY_RECORD store
    participant Report as PAYROLL_SUMMARY_REPORT
    participant Log as SALARY_ACCESS_LOG

    CFO->>App: Xem báo cáo chi phí lương theo phòng ban
    App->>DB: Tính avg_salary theo department_id, đếm sample_size (số nhân viên có lương trong phòng)
    DB-->>App: Kết quả gộp theo từng phòng ban

    loop Với mỗi phòng ban
        alt sample_size >= min_sample_size
            App->>Report: Lưu PAYROLL_SUMMARY_REPORT(avg_salary, sample_size, suppressed=false)
            App-->>CFO: Hiển thị avg_salary cụ thể cho phòng ban này
        else sample_size < min_sample_size (vd phòng chỉ có 1 người)
            App->>Report: Lưu PAYROLL_SUMMARY_REPORT(sample_size, suppressed=true)
            App-->>CFO: Hiển thị "Không đủ dữ liệu để tổng hợp riêng phòng này", gộp chung vào nhóm phòng ban nhỏ khác thay vì lộ số cụ thể
        end
    end

    App->>Log: Ghi SALARY_ACCESS_LOG(decision=allowed, reason=hr_company_wide, target=toàn bộ phòng ban đã tổng hợp)
```
