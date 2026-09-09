# Sequence Diagram - Base: trigger-deploy

Đây là flow **base**: một pipeline CI/CD ở 1 region được trigger (do commit hoặc thao tác thủ công) và deploy thẳng lên hạ tầng của region đó, không hề kiểm tra hay xin phép bất kỳ ai. Flow này là nền để so sánh với enhance, vì enhance sẽ chèn bước xin lock trước khi cho phép bước deploy thật sự chạy.

```mermaid
sequenceDiagram
    actor Dev as Developer / CI trigger
    participant CD as Pipeline CI/CD (Region A)
    participant Infra as Hạ tầng Region A

    Dev->>CD: Push code, trigger deploy
    CD->>CD: Build, test, đóng gói artifact
    CD->>Infra: Deploy artifact lên Region A
    Infra-->>CD: Deploy thành công
    CD-->>Dev: Thông báo deploy Region A hoàn tất
```
