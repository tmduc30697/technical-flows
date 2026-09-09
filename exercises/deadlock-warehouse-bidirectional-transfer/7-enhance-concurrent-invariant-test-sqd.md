# Enhance sequence — Test hàng loạt phiếu chuyển ngẫu nhiên, assert invariant tổng tồn kho và timeout

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 5 của đề bài: dựng nhiều phiếu chuyển ngẫu nhiên giữa các cặp kho khác nhau với danh sách SKU chồng lấn một phần, chạy song song hàng loạt, assert tổng tồn kho từng SKU trên toàn hệ thống bất biến trước/sau, và không có phiếu nào bị treo vượt quá timeout quy định dù đã tính cả thời gian retry.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant Gen as Random Order Generator
    participant Pool as N phiếu chuyển (cặp kho + SKU chồng lấn ngẫu nhiên)
    participant DB as Database (lock toàn cục + retry đã áp dụng)
    participant Inv as Bảng INVENTORY

    Test->>Inv: Đọc snapshot tồn kho toàn hệ thống theo từng SKU trước test
    Test->>Test: Tính total_before[sku] = tổng quantity của sku đó trên mọi kho

    Test->>Gen: Sinh N phiếu chuyển ngẫu nhiên (cặp kho khác nhau, SKU list chồng lấn một phần)
    Gen-->>Pool: Danh sách N phiếu đã sinh, kèm timeout quy định cho mỗi phiếu

    Test->>Pool: Khởi chạy đồng thời toàn bộ N phiếu
    par Chạy song song
        Pool->>DB: Phiếu i: BEGIN, lock theo composite key toàn cục, có thể retry tối đa max_retry lần
    and
        Pool->>DB: Phiếu j (SKU chồng lấn 1 phần với phiếu i): BEGIN, lock theo cùng thứ tự toàn cục
    end

    DB-->>Pool: Từng phiếu commit thành công (một số phiếu tốn thêm thời gian do phải retry)
    Test->>Pool: Đo thời gian hoàn tất từng phiếu (bao gồm cả thời gian retry)
    Test->>Test: assert mọi phiếu hoàn tất trong ngưỡng timeout quy định, không phiếu nào bị treo

    Test->>Inv: Đọc snapshot tồn kho toàn hệ thống theo từng SKU sau khi toàn bộ N phiếu hoàn tất
    Test->>Test: Tính total_after[sku] = tổng quantity của sku đó trên mọi kho
    Test->>Test: assert total_after[sku] == total_before[sku] với mọi SKU

    Note over Test,DB: Invariant tổng tồn kho từng SKU phải đúng bất kể có bao nhiêu phiếu phải retry do deadlock, vì retry chỉ làm chậm chứ không làm sai lệch số lượng chuyển
```
