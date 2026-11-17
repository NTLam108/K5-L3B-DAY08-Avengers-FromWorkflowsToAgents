# Báo cáo nhóm — Lab 8 LogiX-VN

## 1. Thông tin nhóm

- Tên nhóm: **Avengers**.
- Repo: [K5-L3B-DAY08-Avengers-FromWorkflowsToAgents](https://github.com/NTLam108/K5-L3B-DAY08-Avengers-FromWorkflowsToAgents).
- Thành viên và phân công: [TEAM.md](../../TEAM.md).
- Shape chọn: **workflow**, theo [decision_adr.md](decision_adr.md).
- Commit ghi trong ADR: `773a1fa`; nhánh dothanh đã tích hợp origin/main tại `dfcbd8a` ngày 17/11/2026. Commit nộp cuối cần được nhóm đồng bộ với ADR sau khi chốt thay đổi.

## 2. Mục tiêu và cách tiếp cận

LogiX-VN xử lý claim hoàn phí vận chuyển và hỏng hàng. Mục tiêu là triển khai cùng nghiệp vụ bằng workflow, single agent và orchestrator, so sánh với swarm tham chiếu để chọn kiến trúc ít tự chủ nhất vẫn đáp ứng yêu cầu. Kết quả được kiểm qua `RunResult`, ledger SQLite và trace, không dựa vào câu trả lời khách có thuyết phục hay không.

Nhóm thống nhất ba ranh giới: order và policy hiện hành là nguồn dữ kiện; code tính tiền; host kiểm approval, trần theo đơn và idempotency trước khi ghi refund. Ticket là dữ liệu chưa xác minh. Yêu cầu trong ticket không được thay policy, principal hay số ngày trễ trong order.

Quy trình làm việc gồm kiểm starter và contract, triển khai ba shape theo cùng chữ ký `run_claim`, kiểm T01–T14, bổ sung test an toàn, rồi benchmark trên cùng corpus và seed. Nhóm tách phần shape, durability/policy và đánh giá để kiểm tra cả logic quyết định lẫn effect thật trong SQLite.

## 3. Giải pháp

### Shape A — Workflow

[workflow.py](../../src/lab8/shapes/workflow.py) thực hiện chuỗi bước Python cố định: đọc claim qua `get_claim_view`, kiểm `order_id`, tra order trong phạm vi principal, rồi gọi `compute_compensation`. Claim không tồn tại trả `failed`; thiếu mã đơn trả `needs_info`; đơn không tồn tại hoặc ngoài scope trả `escalated`.

Quote bằng 0 dẫn đến `denied` mà không ghi refund. Quote dương được gửi nguyên vẹn vào `durability.issue_refund`; host quyết định số tiền thực ghi. Approval không hợp lệ hoặc đơn đã hoàn đủ dẫn đến `denied`; amount conflict dẫn đến `failed`. Workflow không gọi model. Trace ghi khả năng xem order, quote và quyết định; không chứa ticket hay bản ghi database thô.

### Shape B — Single agent

[single.py](../../src/lab8/shapes/single.py) dùng `run_react(max_steps=8, worker=True)`. Callback `decide` chọn lần lượt `get_claim_view → lookup_order → compute_compensation → final`, dừng sớm khi thiếu thông tin hoặc không xem được order. Giới hạn tám lượt bao gồm lượt trả final. Nếu vòng chạm step cap, shape trả `failed` và không ghi tiền.

Mỗi lượt gọi MockLLM qua `complete(budget, ...)`; runtime tự tính tool cost khi dispatch. Cùng một `Budget` đi xuyên suốt lần chạy để tránh bỏ sót hoặc đếm trùng model/tool call. Trong triển khai hiện tại, action được chọn bằng nhánh Python từ kết quả tool; output MockLLM chưa điều khiển action. Đây là giới hạn cần giữ khi diễn giải kết quả single.

`decide` không đọc số ngày hay clause từ ticket và không chọn tool ghi. Sau khi vòng kết thúc với quote dương, code mới gọi `issue_refund`. Trace ghi action, observation đã lọc, outcome refund và quyết định. [Tám test riêng của single](../../src/tests/test_single.py) kiểm unknown claim, Budget, ba điểm crash, conflict, tool chỉ đọc/thời điểm ghi tiền và step cap.

### Shape C — Orchestrator và handoff

[orchestrator.py](../../src/lab8/shapes/orchestrator.py) giữ quyết định tính tiền và gọi host ở orchestrator. Investigator chỉ chạy `lookup_order` và `lookup_refund_history`, trong scope principal của claim. `compute_compensation` được orchestrator gọi bằng code sau khi nhận order view hợp lệ.

Handoff được tạo bằng `make_handoff` theo `handoff.v1`, với `facts` rỗng vì chưa có dữ kiện đã kiểm chứng. Hàm tạo handoff lọc field qua allowlist, từ chối tool ghi và payload còn dấu hiệu secret/header. Nhóm không dùng bản tóm tắt LLM để thay việc lọc dữ liệu.

Trace ghi `plan`, handoff đã sanitize, kết quả investigator qua allowlist, quote và decision; khi crash thì flush trace rồi raise lại. [Test orchestrator](../../src/tests/test_orchestrator.py) kiểm ranh giới handoff, trace trên 14 claim, quote, retry và crash/resume.

### An toàn ghi tiền và trace

Cả ba shape truyền **quote đầy đủ** cùng `crash_at` cho host. Ví dụ T14 có entitlement 150.000 VND, đã hoàn 30.000 VND: `proposal_amount` vẫn là 150.000, host chỉ ghi thêm 120.000. T13 đã hoàn đủ thì bị chặn với audit `policy_denied`. Shape không tự trừ phần lịch sử trước khi gọi host.

Host dùng khóa `refund:{claim_id}` và transaction chung cho refund cùng audit `refund_committed`. Retry không ghi thêm effect; amount thay đổi dẫn đến conflict và giữ số đã commit. `InjectedCrash` phải đi ra ngoài để test xác nhận crash thật; connection được đóng bằng `finally`.

Các nhánh kết thúc bình thường dùng `finish()` để lấy effect/audit từ ledger và ghi trace. Trace có run id khớp kết quả, được ghi trong lần chạy hiện tại; không ghi nguyên ticket, parent context, secret, header, SĐT, địa chỉ, tên hàng hay internal note.

## 4. Kết quả

### Benchmark bốn shape

Nguồn: [benchmark.json](../../src/reports/benchmark.json). Mỗi shape chạy 14 claim × 5 seed `11,22,33,44,55` = 70 trial; tổng 280 trial. Các trial dùng SQLite mới, cùng MockLLM, policy và bảng giá mô phỏng. Pass rate tính trên trial; pass^5 yêu cầu claim đúng ở cả năm seed.

| Shape | Pass | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND) | Mean/p95 trace events |
| --- | ---: | ---: | ---: | --- | ---: | ---: | --- |
| Workflow | 70/70 | 1.000000 | 1.000000 | 1.000–1.000 | 0 | 27.86 | 2.79 / 3 |
| Single | 70/70 | 1.000000 | 1.000000 | 1.000–1.000 | 0 | 29.46 | 8.21 / 9 |
| Orchestrator | 70/70 | 1.000000 | 1.000000 | 1.000–1.000 | 0 | 47.90 | 4.71 / 5 |
| Swarm | 69/70 | 0.985714 | 0.928571 | 0.929–1.000 | 0.000816327 | 49.57 | 2.80 / 3 |

Cost/success tính tổng chi phí, kể cả trial fail, chia số thành công. Steps của benchmark là số event trong trace, không phải latency; cách ghi event khác nhau giữa shape nên số này không phản ánh đầy đủ mọi thao tác. Mean model calls của workflow/single/orchestrator/swarm lần lượt là 0 / 3.79 / 2 / 3.

### Seed variance và T02 của swarm

Bốn seed đầu của swarm pass 14/14 claim; seed 55 pass 13/14. Với các tỷ lệ `[1, 1, 1, 1, 13/14]`, trung bình `69/70`, phương sai tổng thể bằng `0.0008163265`. Một claim không ổn định làm pass^5 giảm xuống 13/14, dù pass rate vẫn đạt 98.57%.

Trial lỗi `R-swarm-C-002-s55` có ticket nói trễ 5 ngày nhưng order O-202 ghi 2 ngày. Khối `REFERENCE_WEAKNESS` cố ý ghi đè delay khi `seed % 5 == 0`, khiến quote thành 20.000 VND và trả `resolved` thay vì `denied`. Trace ghi rõ order 2 ngày, `model_override` thành 5, rồi quote sai. Ba shape nhóm viết dùng order đã xác minh nên pass ca này. Đây là lỗi nguồn dữ kiện của swarm tham chiếu, không đủ để kết luận mọi hệ multi-agent đều mắc cùng lỗi.

### Bonus B4

**Bonus B4** ([`bonus_report.md`](bonus_report.md), số liệu ở `src/reports/bonus_ablation.json`): baseline rules-only trùng workflow ở mọi trial; swarm cắt ngẫu nhiên lời gọi `peer`/`parse` giảm 30% model call mà pass rate không đổi. Chênh lệch workflow so với swarm **không có ý nghĩa thống kê** trên 14 claim (McNemar p = 1.000, sign test theo claim p = 1.000, CI95 của chênh lệch [0.000, 0.043]). Ablation cho thấy guard `needs_info` và guard phạm vi đơn là cần thiết; guard "quote 0 thì không gọi host" bỏ đi không đổi metric nhưng làm host ghi thêm audit `policy_denied` thừa.

Nhóm chưa có thư mục student_drop trong lần kiểm tra này nên chưa có kết quả hidden self-check.

### Durability, approval và bảo mật

[Crash report](../../src/reports/crash.json) đạt **20/20**; chạy lại crash suite cho cùng kết quả. Suite này kiểm host cùng một kịch bản swarm, không phải 20 kịch bản riêng cho mỗi shape. T11/T12 của ma trận public kiểm retry/crash cho từng shape. Test bổ sung kiểm single ở cả ba cửa sổ crash và workflow trên T14, nơi host chỉ được hoàn phần chênh.

[Test refund safety](../../src/tests/test_refund_safety_matrix.py) và [test workflow safety](../../src/tests/test_workflow_refund_safety.py) kiểm approval thiếu/hỏng/sai claim, idempotency, T13 và T14. Approval bị chặn không tạo refund, nhưng quote dương vẫn cần đi qua host để có audit `policy_denied`.

Lần chạy [script quét](../../src/scripts/scan_public_benchmark.py) khi hoàn thiện báo cáo đã kiểm 280 trace mới cùng stdout/stderr với 42 chuỗi secret/PII: **0 chuỗi khớp**. Đây là bằng chứng trên dữ liệu public và các chuỗi scanner kiểm, chưa chứng minh an toàn trên mọi input.

### Kiểm chứng và tái lập

Sau khi tích hợp origin/main tại `dfcbd8a`, kết quả thực chạy ngày 17/11/2026:

| Kiểm tra | Kết quả |
| --- | --- |
| Test host mặc định | 131 passed, 248 deselected |
| Ma trận student | 248 passed, 131 deselected; mỗi shape 62 test |
| Kiểm tra format bài nộp | 0 FAIL, 1 WARN do thay đổi tài liệu chưa commit |
| Crash suite chạy lại | 20/20 |
| Quét trace và stdout/stderr | 280 trace, 42 chuỗi, 0 khớp |

Student suite kiểm lỗi T02/55 của swarm là một lỗi kỳ vọng, nên 248 test xanh vẫn tương thích với benchmark swarm chỉ pass 69/70 trial. Khi chạy lại benchmark trước đó, báo cáo sinh mới khớp hoàn toàn file nhóm đã lưu. Chưa có kết quả hidden chính thức; không suy ra đạt ngưỡng hidden 0.75 từ public.

Từ `src/`, sau khi kích hoạt venv:

```bash
python -m pytest -q
python -m pytest -q -m student -o addopts=
python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
python -m lab8.crash_test --count 20 --out reports/crash.json
python scripts/scan_public_benchmark.py
python -m lab8.check_submission
```

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

Phân công dưới đây dựa trên [TEAM.md](../../TEAM.md), test và báo cáo cá nhân hiện có. Commit và thông tin định danh còn thiếu cần được từng thành viên xác nhận trước khi nộp.

| Thành viên | Phần việc | Bằng chứng chính |
| --- | --- | --- |
| Nguyễn Mạnh Quân | Workflow và tích hợp | workflow.py; test_workflow.py; báo cáo cá nhân |
| Đỗ Minh Thanh | Single agent, ReAct, Budget và phân tích ADR | single.py; test_single.py; báo cáo cá nhân |
| Nguyễn Trọng Lâm | Orchestrator, handoff và trace | orchestrator.py; test_orchestrator.py; mục trace trong ADR |
| Vũ Xuân Hiếu | Durability, policy và bảo mật | test_refund_safety_matrix.py; test_workflow_refund_safety.py; script quét |
| Nguyễn Đức Hiếu | Benchmark, Eval và báo cáo | benchmark_swarm_baseline.json; test_eval_fairness.py; báo cáo cá nhân |

Thông tin định danh và bảng đóng góp của cả năm thành viên được ghi trong TEAM.md. Báo cáo cá nhân của Đỗ Minh Thanh nằm tại [2A202603793_DoMinhThanh.md](2A202603793_DoMinhThanh.md).

## 7. Quyết định, hạn chế và hướng cải thiện

Nhóm chọn **workflow**: chất lượng public ngang single và orchestrator, chi phí thấp nhất và không cần model call. Orchestrator tăng cost/success khoảng 72% mà không tăng pass rate; swarm tăng khoảng 78% và giảm pass^5. Chi tiết phân tích nằm trong [ADR](decision_adr.md); giả thuyết nghiên cứu và phép so sánh nằm trong [research card](research_card.md).

Các shape khác nhau cả số call, luồng điều khiển và cách ghi trace. So sánh hiện tại đánh giá các triển khai cụ thể; chưa cô lập hoàn toàn tác động riêng của topology như một ablation chỉ đổi một biến. Public chỉ có 14 claim, có hiệu ứng trần khi ba shape đều đạt 100%; variance bằng 0 trên mẫu này không bảo đảm ổn định ngoài mẫu.

MockLLM tất định theo seed, chi phí là mô phỏng, latency là `unavailable`. Nhóm chưa có bằng chứng về hiệu quả model sản xuất, tốc độ xử lý hay hidden chính thức. Hướng tiếp theo là kiểm corpus rộng hơn, nhiều seed hơn, cô lập thay đổi topology và đo latency thật khi có nhu cầu tra cứu nhiều nguồn độc lập. Chỉ chuyển sang kiến trúc tự chủ hơn khi có lợi ích đo được và vẫn qua ba must-pass: approval, secret và crash.

## 8. Tài liệu tham khảo

- [Business rules](../../docs/business_rules.md): nguồn dữ kiện, điều kiện hoàn và trần theo đơn.
- [Contracts](../../docs/contracts.md): RunResult, audit và trace contract.
- [Metrics](../../docs/metrics.md): pass rate, pass^5, variance và cost/success.
- [Rubric](../../RUBRIC.md): tiêu chí chấm và must-pass.
- [Benchmark](../../src/reports/benchmark.json), [crash report](../../src/reports/crash.json), [ADR](decision_adr.md) và [research card](research_card.md): số liệu và quyết định của nhóm.
