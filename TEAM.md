# Team

## Thông tin nhóm

- Tên nhóm: Avengers
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-Avengers-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: https://github.com/NTLam108/K5-L3B-DAY08-Avengers-FromWorkflowsToAgents, main

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Nguyễn Mạnh Quân | 2A202603776 | manhquan03122005@gmail.com | osgi23 | Shape A · Workflow + Tích hợp |
|   2 | Thanh | | | | Shape B · Single agent |
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
| Nguyễn Trọng Lâm | Shape C · Orchestrator + Handoff · mục Trace và bằng chứng (ADR) | `880edc9`, `34a433c` | `src/tests/test_orchestrator.py` |
| Vũ Xuân Hiếu | C3 Durability · C4 Policy · quét trace và stdout | `f4221af` | `src/tests/test_refund_safety_matrix.py`, `src/tests/test_workflow_refund_safety.py`, `src/tests/test_order_cap_and_approval_binding.py` |
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

### Đóng góp cụ thể

- Cài orchestrator theo CP2 và CP3: handoff allowlist với `facts` rỗng, investigator chỉ có `lookup_order` và `lookup_refund_history`, orchestrator tự tính quote và truyền quote đầy đủ cùng `crash_at` vào `durability.issue_refund`.
- Thêm `src/tests/test_orchestrator.py` (21 test).
- Viết phần "Shape C — Orchestrator và handoff" trong `report/team/group_report.md`.

### Bằng chứng

- Commit `880edc9` (orchestrator) và `34a433c` (test).
- `python -m pytest -q -m student -o addopts= -k orchestrator`: 62/62 passed; `python -m pytest -q tests/test_orchestrator.py`: 21/21 passed.
- Benchmark public: orchestrator 70/70 trial pass (`src/reports/benchmark.json`).

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

## MSSV_HoTen

### Phần việc phụ trách

-

### Đóng góp cụ thể

-

### Bằng chứng

-
