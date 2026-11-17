# Team

## Thông tin nhóm

- Tên nhóm: Avengers
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-Avengers-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: https://github.com/NTLam108/K5-L3B-DAY08-Avengers-FromWorkflowsToAgents, main

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Nguyễn Mạnh Quân | 2A202603776 | 26ai.quannm9@vinuni.edu.vn | osgi23 | Shape A · Workflow + Tích hợp |
|   2 | Đỗ Minh Thanh | 2A202603793 | thd050810@gmail.com | thd050810 | Shape B · Single agent |
|   3 | Nguyễn Trọng Lâm | 2A202603817 | 26ai.lamnt5@vinuni.edu.vn | NTLam108 | Shape C · Orchestrator + Handoff |
|   4 | Vũ Xuân Hiếu | 2A202603860 | [26ai.hieuvx2@vinuni.edu.vn](mailto:26ai.hieuvx2@vinuni.edu.vn) | vuhieuvh | Durability · Policy · Bảo mật |
|   5 | Nguyễn Đức Hiếu | 2A202603772 | 26ai.hieund6@vinuni.edu.vn | Hie24 | Benchmark · Đánh giá · Báo cáo |

Gợi ý chia vai:

- Nhóm 6 người: integration; workflow; single agent; orchestrator; ledger/policy/crash; eval/report.
- Nhóm 4 người: workflow + integration; single; orchestrator + handoff; ledger + evaluation.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Nguyễn Mạnh Quân | Shape A · Workflow + Tích hợp (Shape chọn nộp) | `da275cb` | `src/tests/test_workflow.py` |
| Đỗ Minh Thanh | Shape B · Single agent · ReAct/Budget · phân tích seed variance trong ADR | `bbd7c4e`, `e30eb01`, `018ffe9`, `85f981d` | `src/tests/test_single.py` (8 test: unknown claim, Budget, 3 điểm crash, conflict, tool chỉ đọc/thời điểm ghi refund, step cap) |
| Nguyễn Trọng Lâm | Shape C · Orchestrator + Handoff · report benchmark/crash · ADR, research card, báo cáo nhóm · bonus B4 | `880edc9`, `34a433c`, `a152042`, `30accf1`, `95b63db`, `773a1fa`, `ca41f8e` | `src/tests/test_orchestrator.py` (21 test), `src/tests/test_bonus_b4.py` (24 test) |
| Vũ Xuân Hiếu | C3 Durability · C4 Policy · quét trace, stdout và stderr | `f4221af`, `e8bf872` | `src/tests/test_refund_safety_matrix.py`, `src/tests/test_workflow_refund_safety.py`, `src/tests/test_order_cap_and_approval_binding.py` |
| Nguyễn Đức Hiếu | Eval · baseline Swarm · kiểm chứng benchmark · báo cáo cá nhân | `4c947f9`, `976e2ef` | `src/tests/test_eval_fairness.py` (3 test: DB riêng từng trial, đảo thứ tự seed, tái lập kết quả và chi phí) |

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

Ví dụ: `report/team/20241234_NguyenVanAn.md`.

Sao chép mục dưới đây cho từng thành viên.

## 2A202603776_NguyenManhQuan

### Phần việc phụ trách

- Shape A (Deterministic Workflow) trong `src/lab8/shapes/workflow.py`: shape được cả nhóm thống nhất chọn làm phương án nộp chính thức trong `report/team/decision_adr.md`.
- Tích hợp và kiểm tra toàn bộ suite test cổng host và bộ test student.

### Đóng góp cụ thể

- Hoàn thiện toàn bộ logic workflow theo đúng contracts và business rules (T01–T14): đọc qua `get_claim_view`, `lookup_order`, tính tiền qua `compute_compensation`, gọi `durability.issue_refund` với quote đầy đủ, truyền `crash_at` và bung lỗi `InjectedCrash`.
- Tuân thủ nghiêm ngặt Trace Contract v1, chỉ log metadata đã allowlist, không lộ bất kỳ secret hay PII nào trong 70 trial benchmark.
- Thêm bộ test `src/tests/test_workflow.py` (4 test kiểm tra claim chuẩn, thiếu order_id, ngoài scope principal, và claim không tồn tại).

### Bằng chứng

- Commit `da275cb` — `feat(workflow): implement shape A deterministic workflow and unit tests`.
- `python -m pytest -q -m student -o addopts= -k workflow`: 62/62 passed.
- `python -m pytest -q tests/test_workflow.py`: 4/4 passed.
- `src/reports/benchmark.json`: Shape workflow đạt 70/70 trial pass (100%), pass^5 = 1.000, phương sai seed = 0.0, chi phí mô phỏng tối ưu nhất nhóm: 27.86 VND/success (tổng 1950 VND).
- `python -m lab8.check_submission`: 0 FAIL, 0 WARN.

## 2A202603817_NguyenTrongLam

### Phần việc phụ trách

- Shape C (Orchestrator + read-only investigators) trong `src/lab8/shapes/orchestrator.py`.
- Ranh giới handoff `handoff.v1` và trace an toàn (C5); mục "Trace và bằng chứng" trong `report/team/decision_adr.md`.
- Sinh report benchmark và crash, điền ADR và research card, hoàn thiện báo cáo nhóm, chốt commit nộp.
- Bonus B4: baseline công bằng, ablation và thống kê.

### Đóng góp cụ thể

- Cài orchestrator theo CP2 và CP3: handoff allowlist với `facts` rỗng, investigator chỉ có `lookup_order` và `lookup_refund_history`, orchestrator tự tính quote và truyền quote đầy đủ cùng `crash_at` vào `durability.issue_refund`.
- Thêm `src/tests/test_orchestrator.py` (21 test): ranh giới handoff, quét lộ dữ liệu trên 14 claim, tiền do code tính, gọi lại ba lần, crash rồi resume.
- Sinh `src/reports/benchmark.json` (4 shape × 14 claim × 5 seed) và `src/reports/crash.json` (20/20); điền `decision_adr.md` (shape chọn workflow) và `research_card.md` (R1 · Scaling).
- Viết mục "Trace và bằng chứng" trong ADR; viết phần orchestrator, workflow, single, kết quả, khó khăn và tài liệu tham khảo trong `report/team/group_report.md`; chốt commit `773a1fa` sau khi chạy lại benchmark và crash cho kết quả trùng khớp.
- Bonus B4: baseline `rules_only` và `swarm_pruned`, ba ablation trên workflow qua `--extra-shape`, `src/bonus/stats.py` (bootstrap CI theo claim, McNemar, sign test) và `report/team/bonus_report.md`; thêm `src/tests/test_bonus_b4.py` (24 test).

### Bằng chứng

- Commit `880edc9` (orchestrator), `34a433c` (test orchestrator), `a152042` (report benchmark và crash), `30accf1` (ADR và research card), `95b63db` (CP5: trace evidence, báo cáo cá nhân), `773a1fa` (bonus B4), `ca41f8e` (chốt commit, hoàn thiện báo cáo nhóm).
- `python -m pytest -q -m student -o addopts= -k orchestrator`: 62/62 passed; `python -m pytest -q tests/test_orchestrator.py`: 21/21 passed; `python -m pytest -q tests/test_bonus_b4.py`: 24/24 passed.
- Benchmark public: orchestrator 70/70 trial pass (`src/reports/benchmark.json`); bonus: `src/reports/bonus_benchmark.json` (490 trial) và `src/reports/bonus_ablation.json`.

## 2A202603860_VuXuanHieu

### Phần việc phụ trách

- Kiểm thử tính idempotent, khôi phục sau crash, approval và trần hoàn tiền theo đơn. Quét dữ liệu nhạy cảm trong trace và đầu ra console của benchmark public.

### Đóng góp cụ thể

- Thêm test T13/T14 và test gọi cùng claim ba lần trên cả ba shape. Với Workflow, kiểm tra ba điểm crash và bốn dạng approval không hợp lệ.
- Viết script tái lập phép quét benchmark public và ghi phạm vi kiểm chứng trong [báo cáo cá nhân](report/team/2A202603860_VuXuanHieu.md).

### Bằng chứng

- Commit `f4221af` cho các test refund, `e8bf872` cho script quét.
- Crash suite chạy lại đạt **20/20**, khớp [báo cáo crash](src/reports/crash.json).
- Từ `src/`, chạy `python scripts/scan_public_benchmark.py` để quét 280 trace mới, stdout và stderr với 42 chuỗi tìm kiếm. Kết quả trên bộ public có 0 chuỗi khớp.

## 2A202603772_NguyenDucHieu

### Phần việc phụ trách

- Benchmark và đánh giá (Eval + Report): baseline Swarm, các metric trong `docs/metrics.md`, kiểm tra tính công bằng và khả năng tái lập của phép đo.
- Kiểm chứng benchmark bốn shape sau tích hợp; sử dụng R1 và ADR nhóm đã chốt để đối chiếu số liệu.

### Đóng góp cụ thể

- Chạy baseline Swarm trên 14 claim public với seed `11,22,33,44,55`, tạo `src/reports/benchmark_swarm_baseline.json` bằng công cụ của lab.
- Thêm ba test trong `src/tests/test_eval_fairness.py`: mỗi trial có DB riêng và chỉ chứa refund lịch sử đã seed lúc bắt đầu; đảo thứ tự seed không đổi kết quả; chạy lại cùng cấu hình tái lập được kết quả và chi phí mô phỏng.
- Phân tích trial `C-002`, seed 55: Swarm tin số ngày trễ trong ticket (5 ngày) thay vì order (2 ngày), dẫn đến quote sai 20.000 VND và trạng thái `resolved` thay vì `denied`.
- Chạy lại benchmark 4 shape × 14 claim × 5 seed và đối chiếu toàn bộ nội dung JSON với kết quả nhóm: khớp biến kiểm soát, thống kê và 280 trial. File kiểm tra trùng kết quả được giữ local; bằng chứng nộp dùng `src/reports/benchmark.json`.
- Hoàn thiện [báo cáo cá nhân](report/team/2A202603772_NguyenDucHieu.md) theo template.

### Bằng chứng

- Commit `4c947f9` — `test(eval): add benchmark fairness checks and swarm baseline` (đã có trên main).
- Commit `976e2ef` — `docs(report): add Nguyen Duc Hieu personal report`.
- Baseline Swarm: 69/70 trial pass, pass rate 98,5714%, pass^5 92,8571%, seed variance 0,0008163265, cost/success 49,5669565 VND mô phỏng.
- `src/reports/benchmark.json`: workflow, single và orchestrator cùng đạt 70/70 trial pass; Swarm đạt 69/70. Chi phí mỗi trial thành công lần lượt là 27,86; 29,46; 47,90; 49,57 VND mô phỏng.
- Ba test Eval đã pass trong lần chạy toàn bộ suite gồm 347 test tại cây mã tương đương main `4c947f9`; số này là tổng test tích hợp ở thời điểm kiểm chứng, không phải số test do cá nhân thêm.

## 2A202603793_DoMinhThanh

### Phần việc phụ trách

- Shape B (Single agent) trong `src/lab8/shapes/single.py`: vòng ReAct, tool chỉ đọc, Budget và host ghi refund sau vòng lặp.
- Tài liệu dùng chung `docs/decide_tools.md`, test `src/tests/test_single.py`, phân tích seed variance và mục “Có đáng đổi lấy multi-agent không” trong ADR.

### Đóng góp cụ thể

- Đọc `run_react`, `dispatch`, Budget và swarm tham chiếu; ghi lại action tool/args hoặc final, giới hạn tám lượt có tính lượt final, cách giữ observation và tránh đếm trùng model/tool call.
- Thay stub single bằng `run_react(max_steps=8, worker=True)`. Callback chọn `get_claim_view → lookup_order → compute_compensation → final`; dừng sớm với failed/needs_info/escalated khi thiếu dữ kiện hoặc order ngoài scope.
- Gọi MockLLM qua `complete(budget, ...)` để tính model call, token và chi phí. Action hiện được chọn bằng nhánh Python; output MockLLM chưa trực tiếp điều khiển action.
- Giữ tiền trong code: tính quote từ order và policy hiện hành; không tin số ngày ticket hay clause SHIP-LATE-OLD. Chỉ sau vòng ReAct mới gọi `durability.issue_refund`, truyền quote đầy đủ cùng `crash_at`, không bắt `InjectedCrash`.
- Thêm tám test riêng: claim không tồn tại và trace; Budget; ba cửa sổ crash/resume; amount conflict; callback chỉ chọn tool đọc và chưa ghi tiền trong vòng; step cap không refund.
- Phân tích lỗi swarm C-002 seed 55: ghi đè delay từ 2 thành 5 theo ticket, dẫn đến hoàn sai 20.000 VND. Tính variance 0.0008163265 và đối chiếu cost/success của bốn shape trong ADR.
- Hoàn thiện [báo cáo cá nhân](report/team/2A202603793_DoMinhThanh.md), ghi rõ phạm vi kiểm chứng public và giới hạn MockLLM.

### Bằng chứng

- Commit `bbd7c4e` — tài liệu chọn tool và Budget; `e30eb01` — triển khai single; `018ffe9` — tám test single; `85f981d` — phân tích ADR.
- Từ `src/`, chạy `python -m pytest -q tests/test_single.py`: **8/8 passed**.
- `python -m pytest -q -m student -o addopts= -k 'single and not workflow and not orchestrator and not swarm'`: **62/62 passed**, gồm T11 retry và T12 crash/resume.
- `src/reports/benchmark.json`: single **70/70 trial pass**, pass rate/pass^5 **1.000/1.000**, seed min–max **1.000–1.000**, variance **0**, cost/success **29,46 VND mô phỏng**; mean model calls **3,79**, mean tool calls **2,79**.
- Run id đối chiếu: `R-single-C-002-s55` trả denied đúng nhãn, còn `R-swarm-C-002-s55` trả resolved sai nhãn. Single không tự trừ tiền đã hoàn trước khi gọi host.
