# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Tạ Thị Thùy Dương  
**Vai trò:** Evaluation Lead (Sprint 3 — P3)  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `artifacts/eval/before_after_eval.csv` — eval trạng thái tốt sau `run_id=sprint3-clean`
- `artifacts/eval/after_inject_bad.csv` — eval trạng thái xấu sau `run_id=inject-bad`
- `artifacts/eval/restored_eval.csv` — eval xác nhận recovery sau `run_id=sprint3-restored`
- `docs/quality_report.md` — báo cáo chất lượng tổng hợp toàn bộ Sprint 3

**Kết nối với thành viên khác:**

Tôi dùng `etl_pipeline.py` (Hiển — Sprint 1) và `eval_retrieval.py` làm entrypoint để chạy các kịch bản. Kết quả eval phụ thuộc trực tiếp vào cleaning rules và expectations (Hậu — Sprint 2): nếu `fix_refund_window` không được viết đúng, kịch bản inject sẽ không tạo ra sự khác biệt đo được. Tôi cũng phát hiện lỗi encoding Windows trong `etl_pipeline.py` khi chạy flag `--skip-validate` và phối hợp fix (`sys.stdout.reconfigure(encoding="utf-8")`).

**Bằng chứng:**

- `artifacts/manifests/manifest_sprint3-clean.json`, `manifest_inject-bad.json`, `manifest_sprint3-restored.json` — 3 run do tôi khởi tạo
- `artifacts/logs/run_sprint3-clean.log`, `run_inject-bad.log`, `run_sprint3-restored.log`

---

## 2. Một quyết định kỹ thuật

Tôi chọn **chạy 3 lần pipeline theo thứ tự clean → inject → restore** thay vì chỉ 2 lần (inject rồi fix). Lý do: nếu bắt đầu bằng inject thì không có baseline để so sánh — người đọc không biết trạng thái "đúng" trông như thế nào trước khi bị phá. Bằng cách chạy clean trước, tôi tạo được bảng 3 cột rõ ràng:

| Scenario | hits_forbidden (q_refund_window) |
|---|---|
| sprint3-clean | no |
| inject-bad | **yes** |
| sprint3-restored | no |

Quyết định thứ hai là **không sửa `eval_retrieval.py`** dù tôi có thể thêm cột `scenario` vào output. Tôi giữ nguyên file gốc và dùng 3 file CSV riêng biệt, vì sửa eval script của người khác giữa chừng có thể làm thay đổi kết quả grading theo cách không kiểm soát được. Tách file an toàn hơn cho audit.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` → pipeline crash ngay lập tức với:

```
UnicodeEncodeError: 'charmap' codec can't encode character '\u2192'
```

**Check nào phát hiện:** Lỗi xảy ra tại `print(msg)` trong hàm `log()` — Windows dùng cp1252 làm encoding stdout mặc định, ký tự `→` (U+2192) không tồn tại trong bảng này.

**Điều đáng chú ý:** Lỗi chỉ xuất hiện khi chạy `--skip-validate` vì dòng log chứa `→` chỉ được thực thi ở nhánh `halt=True AND skip_validate=True`. Các run thông thường của Hiển không đi qua nhánh này nên không phát hiện được trong Sprint 1.

**Fix:** Thêm vào `etl_pipeline.py` ngay sau phần import:

```python
if hasattr(sys.stdout, "reconfigure"):
    sys.stdout.reconfigure(encoding="utf-8")
if hasattr(sys.stderr, "reconfigure"):
    sys.stderr.reconfigure(encoding="utf-8")
```

Sau fix, pipeline chạy đúng mà không cần workaround `PYTHONIOENCODING`.

---

## 4. Bằng chứng trước / sau

`run_id=sprint3-clean` (good state) — `before_after_eval.csv`, dòng `q_refund_window`:

```
policy_refund_v4,"Yêu cầu được gửi trong vòng 7 ngày làm việc...",yes,no,,3
```

`run_id=inject-bad` — `after_inject_bad.csv`, dòng `q_refund_window`:

```
policy_refund_v4,"Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc...",yes,yes,,3
```

Cột `hits_forbidden` thay đổi `no → yes`: vector store đang trả về context sai cho agent. Expectation `refund_no_stale_14d_window` đã FAIL ở bước validate (`violations=1`) nhưng bị bypass bởi `--skip-validate` — đúng kịch bản demo có chủ đích. Sau `run_id=sprint3-restored`, `hits_forbidden` trở về `no`, `embed_prune_removed=1` xác nhận chunk stale đã bị xóa khỏi index.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ viết thêm **kịch bản inject HR version conflict**: thêm một dòng vào `policy_export_dirty.csv` với `doc_id=hr_leave_policy`, `effective_date=2025-06-01`, `chunk_text` chứa "10 ngày phép năm", rồi tạm thời tắt rule quarantine stale HR date. Hiện tại `q_leave_version` pass nhất quán qua cả 3 scenario vì inject chỉ ảnh hưởng refund chunk — chưa có bằng chứng thực nghiệm rằng expectation `hr_leave_no_stale_10d_annual` bảo vệ được version conflict HR khi bị tấn công trực tiếp.
