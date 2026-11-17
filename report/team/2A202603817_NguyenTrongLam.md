# Báo cáo cá nhân — Nguyễn Trọng Lâm

## 1. Thông tin thành viên

- Họ tên: Nguyễn Trọng Lâm
- MSSV: 2A202603817
- Email: 26ai.lamnt5@vinuni.edu.vn
- GitHub: NTLam108

## 2. Phần việc phụ trách

- Shape C · Orchestrator + Handoff: `src/lab8/shapes/orchestrator.py`, ranh giới handoff `handoff.v1` và trace an toàn (rubric C1 phần orchestrator, C5).
- Bằng chứng benchmark và crash, ADR, research card, báo cáo nhóm và chốt commit nộp (C6, C7).
- Bonus B4: baseline công bằng, ablation và thống kê.

## 3. Đóng góp cụ thể

- **CP2.** Cài orchestrator: đọc claim qua `get_claim_view`; trả `failed` khi claim không tồn tại và `needs_info` khi thiếu `order_id`; dựng handoff bằng `make_handoff`; chạy `run_readonly_investigators`; trả `escalated` khi đơn không thấy hoặc thuộc khách khác; tính quote bằng `compute_compensation`; gọi `durability.issue_refund` với quote đầy đủ; trả kết quả qua `finish()` để mọi nhánh đều có trace.
- **CP3.** Truyền `crash_at` vào `issue_refund`. Khi gặp `InjectedCrash`, ghi event `crash` vào trace rồi `raise` lại, không nuốt lỗi.
- **Quyết định kỹ thuật.**
  - `facts` của handoff để rỗng: câu chữ trong ticket ("trễ 5 ngày", "ignore policy") là lời khách, chưa phải dữ kiện đã kiểm chứng, nên không được đi qua ranh giới.
  - Investigator chỉ có `lookup_order` và `lookup_refund_history`. Không giao `compute_compensation` cho investigator, để quyết định về tiền nằm hẳn ở orchestrator.
  - Trace chỉ ghi handoff đã sanitize và order view qua allowlist; không ghi ticket, claim row, order row hay parent context.
- **Test.** Thêm `src/tests/test_orchestrator.py` (21 test): investigator chỉ có tool đọc; handoff trong trace chỉ có field allowlist; trace và kết quả của cả 14 claim public không chứa `CANARY_SECRET_`, `parent_secret`, `raw_headers` hay nguyên văn ticket; C-002 seed 55 (ticket ghi trễ 5 ngày, đơn ghi 2 ngày) trả `denied`; C-009 (prompt injection) đúng nhãn; C-014 có `proposal_amount` 150.000 và ledger 120.000; gọi lại ba lần vẫn một refund; crash được ném ra, có event `crash`, resume vẫn một refund.
- **CP5 — report và quyết định kiến trúc.** Sinh `src/reports/benchmark.json` (4 shape × 14 claim × 5 seed) và `src/reports/crash.json` (20/20) bằng công cụ của lab, không sửa tay. Điền `decision_adr.md` với shape chọn `workflow`: cùng pass rate 1.000 với single và orchestrator nhưng rẻ nhất (27,86 VND/success) và ít tự chủ nhất. Điền `research_card.md` câu R1 · Scaling, ablation workflow → orchestrator: pass rate không đổi, cost/success tăng 72%. Viết mục "Trace và bằng chứng" trong ADR.
- **CP5 — báo cáo nhóm và chốt commit.** Viết các mục còn trống của `group_report.md`: mục tiêu và cách tiếp cận, phần workflow, single, orchestrator, bảng kết quả 4 shape, bảng khó khăn và tài liệu tham khảo. Trước khi chốt commit `773a1fa`, chạy lại benchmark và crash test tại commit đó và đối chiếu: 280 trial và 20/20 crash trùng khớp file đã nộp.
- **Bonus B4.** Viết hai baseline: `rules_only` (không gọi model) và `swarm_pruned` (cắt ngẫu nhiên lời gọi `peer`/`parse` theo seed). Viết ba ablation trên workflow, mỗi lần bỏ một guard, chạy qua `--extra-shape` để dùng cùng claim, seed, giá và evaluator. Viết `src/bonus/stats.py` chỉ dùng thư viện chuẩn: bootstrap 95% CI lấy mẫu lại theo claim, McNemar chính xác trên cặp (claim, seed) và sign test theo claim. Viết `report/team/bonus_report.md`.
- **Quyết định kỹ thuật ở bonus.**
  - Lấy mẫu lại theo claim chứ không theo trial: 5 seed của cùng một claim không độc lập vì MockLLM tất định. McNemar theo trial cho p = 0,0625 khi bỏ guard, nhưng sign test theo claim cho p = 1,000; báo cáo nêu cả hai.
  - Bỏ các event trace phụ (`ablation`, `pruning`) vì `steps` đếm event trong trace; để lại thì metric `steps` bị lệch (2,79 → 3,79) dù logic không đổi.
  - Ban đầu thử B7 (kiểm toán grader), nhưng `check_submission` chỉ chấp nhận tiêu đề B1–B6 dù RUBRIC cho chọn B1–B7. Nhóm không sửa checker và chuyển sang B4.
- **Test bonus.** Thêm `src/tests/test_bonus_b4.py` (24 test): `rules_only` trùng workflow trên 14 claim; mỗi ablation chỉ hỏng đúng claim của nó và không tạo tiền; bỏ `zero_quote_guard` giữ nguyên metric nhưng sinh audit `policy_denied` thừa; đồng xu cắt tỉa tái lập được; giá trị p-value và bootstrap đúng.

## 4. Bằng chứng

- Commit `880edc9` — `feat(orchestrator): implement shape C orchestrator with read-only investigators`.
- Commit `34a433c` — `test(orchestrator): handoff boundary, trace leak scan, money in code, retry and crash`.
- `python -m pytest -q -m student -o addopts=`: 62/62 test của orchestrator pass, gồm `test_t11_retry_keeps_one_refund[orchestrator]` và `test_t12_crash_before_checkpoint_does_not_double_pay[orchestrator]`.
- `python -m pytest -q tests/test_orchestrator.py`: 21/21 passed.
- Kiểm tra test có bắt lỗi: cố ý cài ba lỗi vào orchestrator rồi chạy lại. Nuốt `InjectedCrash` làm test crash fail; ghi nguyên ticket vào trace làm 14 test quét lộ dữ liệu fail; tự trừ phần đã hoàn trước khi gọi host làm T14, retry và crash fail. Sau đó khôi phục file.
- `src/reports/benchmark.json`: orchestrator pass rate 1.000, pass^5 1.000, variance 0, cost/success 47,90 VND (workflow 27,86 VND).
- `python scripts/scan_public_benchmark.py` (trong `src/`): 280 trace mới, 42 chuỗi tìm kiếm, 0 chuỗi khớp.
- Commit `a152042` (report benchmark và crash), `30accf1` (ADR và research card), `95b63db` (trace evidence, báo cáo cá nhân), `773a1fa` (bonus B4, commit chốt), `ca41f8e` (chốt commit trong ADR, hoàn thiện báo cáo nhóm).
- `python -m lab8.check_submission`: 0 FAIL, 0 WARN tại `ca41f8e`.
- Bonus: `src/reports/bonus_benchmark.json` (490 trial = 7 shape × 14 claim × 5 seed) và `src/reports/bonus_ablation.json`. Chênh workflow so với swarm 0,014, CI95 [0,000; 0,043], McNemar p = 1,000, sign test p = 1,000. `python -m pytest -q tests/test_bonus_b4.py`: 24/24 passed.

## 5. Công cụ AI đã dùng

Claude Code (Anthropic) hỗ trợ đọc đề, viết `orchestrator.py`, `test_orchestrator.py`, code và test bonus B4, và soạn nháp ADR, research card, báo cáo nhóm, báo cáo cá nhân. Cách tự kiểm tra: chạy cổng host `python -m pytest -q`, ma trận student cho orchestrator (62/62), bộ `test_orchestrator.py` và `test_bonus_b4.py`; cố ý cài lỗi vào orchestrator để chắc test fail đúng chỗ; chạy lại benchmark và crash tại commit chốt để đối chiếu với file report; chạy `lab8.check_submission` (0 FAIL); đối chiếu mọi con số trong báo cáo với file JSON trong `src/reports/` và kết quả script quét trace.

## 6. Tự đánh giá

Orchestrator qua đủ test public và giữ ranh giới handoff bằng allowlist trong code. Trên bộ public, orchestrator không tăng pass rate so với workflow nhưng tốn hơn khoảng 72% chi phí mô phỏng mỗi lần thành công, nên nhóm chọn workflow làm shape nộp. Bonus B4 cho thấy với 14 claim, chênh lệch giữa workflow và swarm không có ý nghĩa thống kê; lý do chọn workflow vì thế dựa vào chi phí và rủi ro hoàn tiền sai, không dựa vào pass rate. Giới hạn: kết quả đo trên 14 claim public với MockLLM tất định theo seed; kết quả hidden do giảng viên chấm và quyết định bonus có được tính hay không.
