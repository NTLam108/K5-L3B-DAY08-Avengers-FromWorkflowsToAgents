# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm: Avengers
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`): 773a1fa

## 2. Mục tiêu và cách tiếp cận

Nhóm xem bài toán là một câu hỏi về **mức tự chủ**: cùng một quy trình xử lý claim hoàn phí ship, cài theo bốn kiến trúc từ ít đến nhiều tự chủ (workflow, single ReAct, orchestrator, swarm), đo trên cùng điều kiện, rồi chọn kiến trúc ít tự chủ nhất mà vẫn làm đúng việc. Ranh giới nhóm giữ ở cả ba shape: số tiền do `compute_compensation` tính, approval, trần theo đơn và chống ghi trùng do host (`durability.issue_refund`) quyết; model chỉ được đọc và soạn chữ; câu chữ trong ticket là dữ liệu chứ không phải lệnh.

**Chia việc.** Năm vai, mỗi người một phần có file riêng để tránh xung đột khi merge: Quân phụ trách workflow và tích hợp; Thanh phụ trách single; Lâm phụ trách orchestrator, handoff và trace; Vũ Hiếu phụ trách durability, policy và quét bảo mật; Đức Hiếu phụ trách benchmark, đánh giá và báo cáo. Mỗi người làm trên nhánh riêng rồi gộp vào `main`; trước mỗi lần gộp nhóm chạy cổng host `python -m pytest -q` và ma trận student.

**Thứ tự làm.** Theo năm checkpoint. CP1: cả nhóm chạy smoke, đọc `business_rules.md`, `contracts.md` và swarm tham chiếu, ghi lại baseline 186 failed / 62 passed. CP2: ba shape được viết song song theo cùng chữ ký `run_claim`, mỗi shape qua 62 test student. CP3: truyền `crash_at`, không nuốt `InjectedCrash`, dựng handoff bằng allowlist (T11, T12 xanh). CP4: crash suite 20/20, rà các nhánh approval hỏng và quét trace. CP5: benchmark 4 shape × 5 seed, ADR, research card, báo cáo và bonus B4.

## 3. Giải pháp

Mô tả ba shape nhóm viết (workflow, single, orchestrator): mỗi shape đọc gì, quyết định gì, gọi host ra sao. Chỉ rõ ranh giới code / model (tham khảo [docs/architecture.md](../../docs/architecture.md)), cách handoff được sanitize và trace ghi những event nào.

### Shape A — Workflow (shape chọn nộp)

[`workflow.py`](../../src/lab8/shapes/workflow.py) là các bước Python cố định, không gọi model (mean model calls 0).

1. Đọc claim qua `get_claim_view`. Claim không tồn tại → `failed` (`unknown_claim`).
2. Thiếu `order_id` → `needs_info`, không ghi ledger, không lấy mã đơn từ ticket.
3. `lookup_order` trong phạm vi principal. Đơn không thấy hoặc thuộc khách khác → `escalated`; trace chỉ ghi `visible` và `reason`.
4. Quote bằng `compute_compensation` trên dòng order, bỏ qua số ngày trễ khách tự khai trong ticket.
5. Quote 0 → `denied`, không gọi host.
6. Quote > 0 → `durability.issue_refund(conn, claim_id, quote đầy đủ, crash_at=crash_at)`. Host từ chối (approval hỏng, đơn đã hoàn đủ) → `denied` với `proposal_amount` vẫn là quote; xung đột số tiền → `failed`; thành công → `resolved`. Khi gặp `InjectedCrash`, workflow ghi event `crash` rồi `raise` lại.

Trace ghi `lookup_order`, `quote` (`amount_vnd`, `reason`, `clause_ids`) và `decision`. Workflow có ít bước nhất (mean 2.79 event) và rẻ nhất (27.86 VND/success).

### Shape B — Single agent

[`single.py`](../../src/lab8/shapes/single.py) dùng `lab8.runtime.run_react` với `max_steps=8` và `worker=True`, nên vòng lặp chỉ dispatch được tool đọc. Callback `decide` chọn tool theo kết quả tool trước: `get_claim_view` → `lookup_order` → `compute_compensation` → trả `final`. Mỗi lượt `decide` gọi MockLLM một lần và tính vào Budget, nên single có mean 3.79 model call. Các nhánh dừng sớm giống workflow: claim không tồn tại → `failed`, thiếu `order_id` → `needs_info`, đơn không thấy → `escalated`.

Model không chọn số tiền và không ghi tiền: `proposal_amount` lấy từ output thật của `compute_compensation`. Chỉ **sau** khi vòng lặp dừng ở `final` với quote > 0, code của shape mới gọi `durability.issue_refund` với quote đầy đủ và `crash_at`. Trace ghi `observation` (riêng `get_claim_view` chỉ giữ `found`, `claim_id`, `order_id`, không giữ ticket), `action`, `refund` và `decision`. Quy ước dùng chung về `decide`, tool và Budget nằm ở [`docs/decide_tools.md`](../../docs/decide_tools.md).

### Shape C — Orchestrator và handoff

[`orchestrator.py`](../../src/lab8/shapes/orchestrator.py) chia việc thành hai vai. Orchestrator giữ mọi quyết định có hậu quả: phân loại claim, tính tiền và gọi host. Investigator chỉ đọc dữ liệu được giao.

1. **Đọc claim.** Orchestrator đọc claim qua `get_claim_view`, không `SELECT *`, nên không chạm tới `parent_secret`. Claim không tồn tại thì trả `failed` (`unknown_claim`); thiếu `order_id` thì trả `needs_info` và không đoán mã đơn từ câu chữ trong ticket.
2. **Dựng handoff.** Handoff được tạo bằng `make_handoff` theo contract `handoff.v1`. Chỉ các field trong allowlist (`contract_version`, `request_id`, `claim_id`, `principal_scope`, `task`, `facts`, `allowed_tools`, `max_output_tokens`) được giữ lại; `make_handoff` từ chối mọi tool ghi và mọi payload còn chứa `CANARY_SECRET_`, `parent_secret` hay `raw_headers`. Nhóm để `facts` rỗng: orchestrator chưa có dữ kiện đã kiểm chứng, còn "trễ 5 ngày" hay "ignore policy" trong ticket chỉ là lời khách.
3. **Investigator chỉ đọc.** `run_readonly_investigators` chạy hai tool `lookup_order` và `lookup_refund_history` trong phạm vi principal của claim. Đơn không tồn tại hoặc thuộc khách khác thì orchestrator trả `escalated`, không chép SĐT, địa chỉ, tên hàng hay internal note. Investigator không có write registry, nên dù muốn cũng không gọi được `issue_refund`.
4. **Tiền do code tính.** Nhóm cố ý không giao `compute_compensation` cho investigator. Orchestrator tự gọi `compute_compensation` trên order view đã kiểm chứng, để quyết định về tiền không đi qua ranh giới handoff. MockLLM chỉ được gọi để lập kế hoạch và soạn câu trả lời.
5. **Gọi host.** Quote bằng 0 thì trả `denied` mà không gọi host. Quote lớn hơn 0 thì orchestrator truyền **quote đầy đủ** vào `durability.issue_refund` để host tự kiểm approval, trừ phần đơn đã được hoàn và chống ghi trùng. Ở T14, quote là **150.000 VND** và host chỉ ghi thêm **120.000 VND**. Orchestrator truyền `crash_at` cho host; khi gặp `InjectedCrash`, orchestrator ghi event `crash` vào trace rồi `raise` lại để crash suite thấy crash thật.

Trace orchestrator ghi `plan`, `handoff` (bản đã sanitize), `investigator` (order view qua allowlist và số tiền đã hoàn), `quote`, `decision`, và `crash` khi có crash; không ghi ticket, claim row, order row hay parent context. Chi tiết và bằng chứng quét trace nằm ở mục "Trace và bằng chứng" của [`decision_adr.md`](decision_adr.md).

Kết quả trên bộ public: **62/62** test student; benchmark **70/70** trial pass, pass^5 1.000, variance 0, cost/success **47,90 VND** so với **27,86 VND** của workflow (`src/reports/benchmark.json`). [`test_orchestrator.py`](../../src/tests/test_orchestrator.py) có 21 test cho ranh giới handoff, quét lộ dữ liệu trên 14 claim, tiền do code tính (C-002, C-009, C-014), gọi lại ba lần và crash rồi resume.

## 4. Kết quả

Bảng so sánh 4 shape trên bộ public, lấy từ [`src/reports/benchmark.json`](../../src/reports/benchmark.json): 14 claim × seed `11,22,33,44,55` = 70 trial mỗi shape, MockLLM, cùng bảng giá, SQLite mới mỗi trial. Cost là VND mô phỏng; latency `unavailable`.

| Shape | Trial pass | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success | Mean / p95 steps | Model calls TB |
| --- | ---: | ---: | ---: | --- | ---: | ---: | --- | ---: |
| workflow | 70/70 | 1.000 | 1.000 | 1.000–1.000 | 0 | 27.86 | 2.79 / 3 | 0.00 |
| single | 70/70 | 1.000 | 1.000 | 1.000–1.000 | 0 | 29.46 | 8.21 / 9 | 3.79 |
| orchestrator | 70/70 | 1.000 | 1.000 | 1.000–1.000 | 0 | 47.90 | 4.71 / 5 | 2.00 |
| swarm | 69/70 | 0.986 | 0.929 | 0.929–1.000 | 0.000816 | 49.57 | 2.80 / 3 | 3.00 |

Trial fail duy nhất là `R-swarm-C-002-s55`: swarm tin số ngày trễ trong ticket (5) thay cho `delay_days` của đơn (2) và hoàn 20.000 VND cho một claim phải bị từ chối. [Crash suite](../../src/reports/crash.json) đạt **20/20**. Tại commit chốt `773a1fa`, chạy lại benchmark và crash test cho kết quả trùng khớp hai file report.

Tự kiểm trên `student_drop/` (bộ hidden không nhãn): nhóm không có thư mục này trong lần chạy, nên chưa có `hidden_selfcheck.json`; con số hidden do giảng viên chấm.

**Bonus B4** ([`bonus_report.md`](bonus_report.md), số liệu ở `src/reports/bonus_ablation.json`): baseline rules-only trùng workflow ở mọi trial; swarm cắt ngẫu nhiên lời gọi `peer`/`parse` giảm 30% model call mà pass rate không đổi. Chênh lệch workflow so với swarm **không có ý nghĩa thống kê** trên 14 claim (McNemar p = 1.000, sign test theo claim p = 1.000, CI95 của chênh lệch [0.000, 0.043]). Ablation cho thấy guard `needs_info` và guard phạm vi đơn là cần thiết; guard "quote 0 thì không gọi host" bỏ đi không đổi metric nhưng làm host ghi thêm audit `policy_denied` thừa.

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

### C3 — Idempotency và khôi phục sau crash

[Crash suite của lab](../../src/reports/crash.json) đạt **20/20** trên host và Swarm tham chiếu. Lần chạy lại cho cùng kết quả ở cả 20 kịch bản.

Demo `python -m lab8.demo.buggy_workflow` in `unsafe refund rows: 2`. Refund đầu tiên đã được ghi khi checkpoint còn thiếu. Khi tiếp tục chỉ dựa vào checkpoint, demo tưởng chưa xử lý và ghi refund lần hai. Host thật ghi refund và audit trong cùng giao dịch SQLite, dùng khóa `refund:{claim_id}` để nhận ra lần ghi trước và bổ sung checkpoint khi chạy lại. Test gọi cùng claim ba lần xác nhận mỗi shape giữ đúng một dòng ledger và một audit hoàn tiền.

Trên Workflow, nhóm dùng claim T14 để thử cả ba điểm crash `before_effect`, `after_effect_before_checkpoint` và `after_checkpoint`. Đơn đã được hoàn **30.000 VND** và Workflow gửi quote đầy đủ **150.000 VND**. Sau khi khôi phục và gọi lại, SQLite giữ đúng một refund **120.000 VND** cho claim, một audit `refund_committed` và checkpoint đã được ghi. Tổng tiền hoàn của đơn là **150.000 VND** ở cả ba kịch bản.

### C4 — Approval, policy hiện hành và trần theo đơn

Workflow từ chối cả bốn ca approval bị thiếu, hết hạn, bị thu hồi và thuộc claim khác. Hai lần gọi mỗi ca đều trả quote **35.000 VND** với trạng thái `denied`. Ledger có **0 refund**, audit có **0 `refund_committed`** và đúng một `policy_denied` với lý do tương ứng. Grader kiểm tra cả trace và kết quả trong các ca này.

Mỗi shape nhóm viết đạt **62/62** test public. [Test bổ sung trên cả ba shape](../../src/tests/test_refund_safety_matrix.py) kiểm tra T13 từ chối hoàn thêm và ghi `policy_denied` với lý do `order_already_refunded`, còn T14 giữ quote **150.000 VND** và chỉ ghi thêm **120.000 VND**. [Test tích hợp trên Workflow](../../src/tests/test_workflow_refund_safety.py) kiểm tra thêm crash và các approval không hợp lệ. Chạy hai bộ từ `src/` bằng `python -m pytest -q tests/test_refund_safety_matrix.py tests/test_workflow_refund_safety.py`.

### Kiểm tra trace và đầu ra

Chạy lại phép kiểm từ thư mục `src/` sau khi kích hoạt môi trường Python của lab.

```bash
python scripts/scan_public_benchmark.py
```

[Script quét](../../src/scripts/scan_public_benchmark.py) chạy benchmark public với 4 shape, 14 claim và các seed `11,22,33,44,55` trong thư mục tạm. Script quét **280 trace mới** cùng stdout và stderr với **42 chuỗi tìm kiếm**, gồm hai tiền tố `CANARY_SECRET_`, `CANARY_INTERNAL_` và 40 giá trị phân biệt của số điện thoại, địa chỉ, ghi chú nội bộ và tên hàng trong `data/public_world_state.json`. Kết quả mong đợi là **0 chuỗi khớp**. Thiếu trace hoặc có chuỗi khớp làm lệnh trả mã lỗi.

Git index có **0** tệp thuộc `traces/`, tệp SQLite hoặc tệp cấu hình `.env`.

## 5. Khó khăn và cách xử lý

Lỗi gặp phải (kèm lệnh, output hoặc ảnh chụp) và cách nhóm xử lý.

| Vấn đề | Lệnh và output | Cách xử lý |
| --- | --- | --- |
| Shape xong CP2 nhưng chưa xử lý crash | `python -m pytest -q -m student -o addopts= -k orchestrator` → `1 failed, 61 passed`, fail ở `test_t12_crash_before_checkpoint_does_not_double_pay[orchestrator]` | Truyền `crash_at` vào `issue_refund`; khi gặp `InjectedCrash` thì ghi trace rồi `raise`, không nuốt lỗi. Chạy lại: 62/62 |
| Hiểu sai demo crash là lỗi của nhóm | `python -m lab8.demo.buggy_workflow` → `unsafe refund rows: 2 (the host ledger must stay at 1)` | Đây là demo cố ý sai: checkpoint đơn lẻ không thay được ledger. Nhóm đối chiếu với host thật (khóa `refund:{claim_id}`, ghi refund và audit trong một transaction) và thêm test gọi lại ba lần, crash rồi resume |
| File report bị Git đánh dấu đã sửa sau khi chạy test trên Windows | `git status` → ` M src/reports/crash.json`; `git diff --ignore-cr-at-eol` trống | Chỉ khác ký tự xuống dòng (CRLF). Khôi phục bằng `git checkout -- <file>`, không commit, không sửa tay JSON |
| Hash commit đổi khi gộp nhánh | Commit test trên nhánh cá nhân là `ddc9ad3`, sau khi rebase lên `main` thành `34a433c` | Bảng đóng góp trong `TEAM.md` ghi hash trên `main`, là nhánh nộp |
| Số `steps` của biến thể bonus bị lệch | Biến thể ablation có mean steps 3.79 thay vì 2.79 dù cùng logic | `steps` đếm event trong trace; event phụ `ablation`/`pruning` làm lệch metric. Bỏ event phụ, chạy lại benchmark |
| Topic bonus B7 không qua checker | `python -m lab8.check_submission` → `[FAIL] report/team/bonus_report.md: tiêu đề phải là '# Bonus report: B1' .. 'B6'` | RUBRIC cho chọn B1–B7 nhưng checker chỉ nhận B1–B6. Nhóm không sửa checker, chuyển sang B4 |

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

Benchmark dùng MockLLM tất định theo seed và giá token mô phỏng. Lab ghi latency là `unavailable`. Kết quả trên 14 claim public đo tính đúng, số bước và chi phí mô phỏng trong cấu hình này. Chất lượng và thời gian đáp ứng khi dùng model sản xuất cần được đo riêng.

Single dùng vòng ReAct tối đa tám bước, với action được chọn bằng nhánh Python dựa trên kết quả tool. Lời gọi MockLLM tính vào chi phí nhưng output chưa điều khiển action. Khi so sánh kiến trúc, cần đọc kết quả Single theo cách triển khai này.

## 8. Tài liệu tham khảo

- Tài liệu của lab: [`README.md`](../../README.md), [`docs/business_rules.md`](../../docs/business_rules.md), [`docs/contracts.md`](../../docs/contracts.md), [`docs/architecture.md`](../../docs/architecture.md), [`docs/metrics.md`](../../docs/metrics.md), [`docs/bonus_topics.md`](../../docs/bonus_topics.md).
- Shape tham chiếu: [`src/lab8/shapes/swarm_reference.py`](../../src/lab8/shapes/swarm_reference.py).
- Google Research (2026). *Towards a Science of Scaling Agent Systems*. Slide 15, 33, 41 của bài N8 (dùng cho research card R1).
- *Rethinking Evaluation of MAS Efficiency*, EMNLP 2026. Slide 44, 59 (bonus B4).
- *AgentArch*, AACL 2026. Slide 43 (bonus B4).
- *Why Do Multi-Agent LLM Systems Fail?* (MAST), NeurIPS 2025. Slide 35, 45 (tham khảo khi phân tích lỗi seed 55 của swarm).
- DBOS, *Durable Workflows and Steps*. Slide 36–37 (idempotency và ledger).
