# Enhance sequence — Theo dõi chi phí băng thông theo vùng

Đây là **enhance**, flow hoàn toàn mới so với base (base không theo dõi chi phí theo vùng vì chỉ có 1 điểm ingest trung tâm). Đáp ứng yêu cầu 5 trong đề bài: theo dõi chi phí băng thông theo từng khu vực địa lý để tối ưu vận hành, cảnh báo khi 1 khu vực có chi phí bất thường so với số viewer thực tế.

```mermaid
sequenceDiagram
    participant EndpointRegion as Ingest/Edge cua tung Region
    participant Collector as Bandwidth Metric Collector
    participant Metric as Region Bandwidth Metric Store
    participant Alert as Cost Alerting Service
    actor Ops as Doi van hanh

    loop Dinh ky (vd moi 5 phut)
        EndpointRegion->>Collector: Bao cao luu luong da truyen + so viewer hien tai theo Region
        Collector->>Metric: Ghi REGION_BANDWIDTH_METRIC (viewer_count, bandwidth_cost, recorded_at)
    end

    Collector->>Collector: Tinh ty le chi phi/viewer trung binh cho tung Region
    alt 1 Region co chi phi/viewer vuot nguong bat thuong so cac Region con lai
        Collector->>Metric: Danh dau is_anomalous = true cho ban ghi do
        Collector->>Alert: Gui canh bao kem Region, chi phi, so viewer thuc te
        Alert-->>Ops: Thong bao "Region X chi phi bat thuong so voi luong viewer"
    else Chi phi trong nguong binh thuong
        Collector->>Metric: Giu is_anomalous = false
    end
```
