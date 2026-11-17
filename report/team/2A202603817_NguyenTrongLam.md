# Báo cáo cá nhân — Nguyễn Trọng Lâm

## 1. Thông tin thành viên

- Họ tên: Nguyễn Trọng Lâm
- MSSV: 2A202603817
- Email: 26ai.lamnt5@vinuni.edu.vn
- GitHub: NTLam108

## 2. Phần việc phụ trách

Shape C · Orchestrator + Handoff: `src/lab8/shapes/orchestrator.py`, ranh giới handoff `handoff.v1` và trace an toàn (rubric C1 phần orchestrator, C5). Ở CP5 phụ trách mục "Trace và bằng chứng" trong `report/team/decision_adr.md` và phần "Shape C — Orchestrator và handoff" trong `report/team/group_report.md`.

## 3. Đóng góp cụ thể

- **CP2.** Cài orchestrator: đọc claim qua `get_claim_view`; trả `failed` khi claim không tồn tại và `needs_info` khi thiếu `order_id`; dựng handoff bằng `make_handoff`; chạy `run_readonly_investigators`; trả `escalated` khi đơn không thấy hoặc thuộc khách khác; tính quote bằng `compute_compensation`; gọi `durability.issue_refund` với quote đầy đủ; trả kết quả qua `finish()` để mọi nhánh đều có trace.
- **CP3.** Truyền `crash_at` vào `issue_refund`. Khi gặp `InjectedCrash`, ghi event `crash` vào trace rồi `raise` lại, không nuốt lỗi.
- **Quyết định kỹ thuật.**
  - `facts` của handoff để rỗng: câu chữ trong ticket ("trễ 5 ngày", "ignore policy") là lời khách, chưa phải dữ kiện đã kiểm chứng, nên không được đi qua ranh giới.
  - Investigator chỉ có `lookup_order` và `lookup_refund_history`. Không giao `compute_compensation` cho investigator, để quyết định về tiền nằm hẳn ở orchestrator.
  - Trace chỉ ghi handoff đã sanitize và order view qua allowlist; không ghi ticket, claim row, order row hay parent context.
- **Test.** Thêm `src/tests/test_orchestrator.py` (21 test): investigator chỉ có tool đọc; handoff trong trace chỉ có field allowlist; trace và kết quả của cả 14 claim public không chứa `CANARY_SECRET_`, `parent_secret`, `raw_headers` hay nguyên văn ticket; C-002 seed 55 (ticket ghi trễ 5 ngày, đơn ghi 2 ngày) trả `denied`; C-009 (prompt injection) đúng nhãn; C-014 có `proposal_amount` 150.000 và ledger 120.000; gọi lại ba lần vẫn một refund; crash được ném ra, có event `crash`, resume vẫn một refund.
- **CP5.** Viết mục "Trace và bằng chứng" trong ADR và phần orchestrator trong báo cáo nhóm.

## 4. Bằng chứng

- Commit `880edc9` — `feat(orchestrator): implement shape C orchestrator with read-only investigators`.
- Commit `34a433c` — `test(orchestrator): handoff boundary, trace leak scan, money in code, retry and crash`.
- `python -m pytest -q -m student -o addopts=`: 62/62 test của orchestrator pass, gồm `test_t11_retry_keeps_one_refund[orchestrator]` và `test_t12_crash_before_checkpoint_does_not_double_pay[orchestrator]`.
- `python -m pytest -q tests/test_orchestrator.py`: 21/21 passed.
- Kiểm tra test có bắt lỗi: cố ý cài ba lỗi vào orchestrator rồi chạy lại. Nuốt `InjectedCrash` làm test crash fail; ghi nguyên ticket vào trace làm 14 test quét lộ dữ liệu fail; tự trừ phần đã hoàn trước khi gọi host làm T14, retry và crash fail. Sau đó khôi phục file.
- `src/reports/benchmark.json`: orchestrator pass rate 1.000, pass^5 1.000, variance 0, cost/success 47,90 VND (workflow 27,86 VND).
- `python scripts/scan_public_benchmark.py` (trong `src/`): 280 trace mới, 42 chuỗi tìm kiếm, 0 chuỗi khớp.

## 5. Công cụ AI đã dùng

Claude Code (Anthropic) hỗ trợ đọc đề, viết `orchestrator.py`, viết `test_orchestrator.py` và soạn nháp các phần báo cáo. Cách tự kiểm tra: chạy cổng host `python -m pytest -q`, ma trận student cho orchestrator (62/62), bộ `test_orchestrator.py`; cố ý cài lỗi vào orchestrator để chắc test fail đúng chỗ; chạy `lab8.check_submission` (0 FAIL); đối chiếu số trong báo cáo với `src/reports/benchmark.json` và kết quả script quét trace.

## 6. Tự đánh giá

Orchestrator qua đủ test public và giữ ranh giới handoff bằng allowlist trong code. Trên bộ public, orchestrator không tăng pass rate so với workflow nhưng tốn hơn khoảng 72% chi phí mô phỏng mỗi lần thành công, nên nhóm chọn workflow làm shape nộp. Giới hạn: kết quả đo trên 14 claim public với MockLLM tất định theo seed; kết quả hidden do giảng viên chấm.
