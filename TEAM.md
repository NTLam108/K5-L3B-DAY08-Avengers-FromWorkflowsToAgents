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
|   4 | Vũ Xuân Hiếu | 2A202603860 | | | Durability · Policy · Bảo mật |
|   5 | Đức Hiếu | | | | Benchmark · Đánh giá · Báo cáo |

Gợi ý chia vai:

- Nhóm 6 người: integration; workflow; single agent; orchestrator; ledger/policy/crash; eval/report.
- Nhóm 4 người: workflow + integration; single; orchestrator + handoff; ledger + evaluation.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Nguyễn Mạnh Quân | Shape A · Workflow + Tích hợp | (pending commit) | tests/test_workflow.py |
| Nguyễn Trọng Lâm | Shape C · Orchestrator + Handoff · mục Trace và bằng chứng (ADR) | `880edc9`, `34a433c` | `src/tests/test_orchestrator.py` |
| Vũ Xuân Hiếu | C3 Durability · C4 Policy · quét trace và stdout | `f4221af` | `src/tests/test_refund_safety_matrix.py`, `src/tests/test_workflow_refund_safety.py`, `src/tests/test_order_cap_and_approval_binding.py` |

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

Ví dụ: `report/team/20241234_NguyenVanAn.md`.

Sao chép mục dưới đây cho từng thành viên.

## 2A202603776_NguyenManhQuan

### Phần việc phụ trách

- Shape A (Deterministic Workflow) trong `src/lab8/shapes/workflow.py`.
- Tích hợp và kiểm tra toàn bộ suite test cổng host.

### Đóng góp cụ thể

- Hoàn thiện toàn bộ logic workflow theo đúng contracts và business rules (T01–T14).
- Thêm bộ test `src/tests/test_workflow.py`.

### Bằng chứng

- `pytest -q -m student -o addopts= -k workflow`: 62/62 passed.
- `pytest -q`: 58/58 passed.
- `python -m lab8.check_submission`: shape workflow OK.

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

## MSSV_HoTen

### Phần việc phụ trách

-

### Đóng góp cụ thể

-

### Bằng chứng

-
