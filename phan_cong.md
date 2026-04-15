# Kế hoạch triển khai dự án theo Sprint

| Giai đoạn | Sprint Owner | Nhiệm vụ chính (Owner làm Lead) | File trọng tâm | Người thực hiện |
| :--- | :--- | :--- | :--- | :--- |
| **Sprint 1** | P1 (Ingestion Lead) | Chốt Schema, Map nguồn dữ liệu, đảm bảo dữ liệu thô vào Log đúng chuẩn. | `etl_pipeline.py` (phần Ingest), `data_contract.md` | Hiển |
| **Sprint 2** | P2 (Logic Lead) | Chốt quy tắc Clean (≥ 3 rules) và Validate (≥ 2 expectations). Đảm bảo Embed không trùng. | `cleaning_rules.py`, `expectations.py` | Hậu |
| **Sprint 3** | P3 (Evaluation Lead) | Chốt kịch bản Inject rác, chạy Eval so sánh trước/sau fix và viết Quality Report. | `eval_retrieval.py`, `quality_report.md` | Dương |
| **Sprint 4** | P4 (Ops Lead) | Chốt Freshness check, viết Runbook hướng dẫn xử lý sự cố và hoàn thiện Docs. | `freshness_check.py`, `runbook.md` | Hiền |
| **Toàn bộ** | P5 (Architect & Auditor) | Setup môi trường, quản lý Git, chạy `grading_run.py` và tổng hợp Báo cáo nhóm. | `README.md`, `group_report.md`, `grading_run.py` | An |