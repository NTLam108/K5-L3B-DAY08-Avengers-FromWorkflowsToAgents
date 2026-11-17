# Báo cáo cá nhân — Đỗ Minh Thanh

## 1. Thông tin thành viên

- Họ tên: Đỗ Minh Thanh
- MSSV: 2A202603793
- GitHub: [thd050810](https://github.com/thd050810)
- Nhóm: Avengers
- Vai trò: Shape B · Single agent

## 2. Phần việc phụ trách

Phụ trách Shape B trong [single.py](../../src/lab8/shapes/single.py): triển khai vòng ReAct có giới hạn, chọn tool chỉ đọc, ghi nhận model/tool call vào Budget và gọi host ghi refund sau vòng lặp. Phần việc gắn với rubric C1 về implementation, C3 về retry/crash và C4 về policy.

Bổ sung [test_single.py](../../src/tests/test_single.py), ghi lại quy ước chọn tool trong [decide_tools.md](../../docs/decide_tools.md), phân tích seed variance và lỗi T02 seed 55 của swarm trong mục “Có đáng đổi lấy multi-agent không” của [ADR](decision_adr.md).

## 3. Đóng góp cụ thể

- **CP1 — Đọc runtime và thống nhất cách chọn tool.** Đọc `run_react`, `dispatch`, `Budget`, MockLLM và swarm tham chiếu. Ghi rõ action của `decide` có dạng tool/args hoặc final; giới hạn tám lượt có tính cả lượt final; runtime chỉ giữ kết quả tool gần nhất. Phân biệt swarm chạy investigator theo danh sách cố định với single dùng callback `decide`. Tài liệu chung nêu cách tính Budget để tránh cộng trùng tool/model call.
- **CP2 — Triển khai single agent.** Thay stub bằng `run_react(max_steps=8, worker=True)`. Luồng chính là `get_claim_view → lookup_order → compute_compensation → final`. Claim không tồn tại trả `failed`; thiếu `order_id` trả `needs_info`; order không visible trả `escalated`. Khi chạm step cap, trả `failed`, không refund.
- **Quyết định kỹ thuật.** `decide` chỉ chọn tool đọc và không gọi `issue_refund`. Tiền lấy từ `compute_compensation` dựa trên order thật và policy hiện hành; không lấy số ngày trễ hay clause khách viện dẫn trong ticket làm nguồn quyết định. Chỉ sau vòng lặp, code mới gửi quote đầy đủ cho host để kiểm approval, cap theo đơn và idempotency. Các nhánh kết thúc bình thường dùng `finish()` để lấy ledger/audit thật và ghi trace.
- **Budget và trace.** Mỗi lượt `decide` gọi MockLLM qua `complete(budget, ...)`, tự ghi nhận model call, token và cost. `dispatch` tính tool cost nên single không cộng lại. Trace ghi action, observation đã lọc và quyết định; khi quan sát claim chỉ ghi `found`, `claim_id`, `order_id`, không ghi nguyên ticket hoặc parent context. Connection luôn đóng bằng `finally`.
- **CP3 — Crash và retry.** Truyền `crash_at` vào `durability.issue_refund`, không bắt `InjectedCrash`. T11 xác nhận gọi lại không ghi thêm refund; T12 xác nhận crash thật rồi resume vẫn giữ một refund và một audit committed. Khi quote mới khác số đã commit, single trả `failed` với `amount_conflict`, không thay effect cũ.
- **Test bổ sung.** Thêm tám test trong `test_single.py` để kiểm các hành vi chưa được ma trận public xác nhận riêng: unknown claim và trace hợp lệ; Budget; ba điểm crash; amount conflict; tool chỉ đọc và thời điểm ghi refund; step cap.
- **CP5 — Phân tích kiến trúc.** Chạy lại benchmark bốn shape trên cùng 14 claim và năm seed, đối chiếu báo cáo sinh lại với file nhóm đã lưu. Phân tích swarm có seed rates `[1, 1, 1, 1, 13/14]`, variance `0.0008163265`; trial `R-swarm-C-002-s55` hoàn sai do ghi đè số ngày order theo ticket. Viết kết luận trong ADR: trên public, multi-agent chưa tăng chất lượng đủ để bù chi phí; nhóm giữ workflow.

## 4. Bằng chứng

### Commit và sản phẩm

| Commit | Nội dung | File chính |
| --- | --- | --- |
| `bbd7c4e` — `checkpoint1-thanh` | Ghi chú runtime, chọn tool và Budget | `docs/decide_tools.md`, `docs/README.md` |
| `e30eb01` — `dothanh checkpoint 2` | Triển khai single agent | `src/lab8/shapes/single.py` |
| `018ffe9` — `add test/single.py` | Bổ sung test riêng cho single | `src/tests/test_single.py` |
| `85f981d` — `cp5` | Phân tích multi-agent và lỗi phụ thuộc seed | `report/team/decision_adr.md` |

### Kết quả kiểm tra

Lần chạy lại ngày 17/11/2026 xác nhận **8/8 test riêng** và **62/62 test public của single** đều pass, gồm T11 và T12. Chạy trong thư mục `src/` sau khi activate venv:

```bash
python -m pytest -q tests/test_single.py
python -m pytest -q -m student -o addopts= -k 'single and not workflow and not orchestrator and not swarm'
```

Bộ lọc loại các shape còn lại vì tên `test_public_single_run_cases` cũng chứa chữ single; chỉ dùng `-k single` có thể chọn thêm test của shape khác.

| Test riêng | Hành vi được kiểm |
| --- | --- |
| `test_unknown_claim_fails_with_valid_trace_and_no_refund` | Claim không tồn tại trả failed, có trace đúng contract và không refund |
| `test_budget_records_actual_model_usage_and_tool_cost` | Số model call khớp lần gọi thật; token dương; tool không đếm trùng; cost khớp bảng giá |
| `test_crash_propagates_and_resume_keeps_one_refund` × 3 | Crash ở before_effect, after_effect_before_checkpoint, after_checkpoint; resume giữ một effect 35.000 VND và một audit |
| `test_changed_quote_conflicts_without_replacing_committed_refund` | Quote đổi từ 35.000 xuống 10.000 VND dẫn đến conflict, giữ refund đã commit |
| `test_react_uses_read_tools_and_commits_only_after_loop` | Chạy worker=True, max_steps=8; action chỉ chọn tool đọc; chưa có refund khi vòng đang chạy |
| `test_step_cap_fails_without_refund_and_writes_trace` | Ép runtime dừng sớm bằng giới hạn một lượt; shape xử lý step_cap, không refund và vẫn ghi trace |

[Benchmark nhóm](../../src/reports/benchmark.json) ghi single **70/70 trial pass**, pass rate **1.000**, pass^5 **1.000**, seed min–max **1.000–1.000**, variance **0**, cost/success **29,46 VND mô phỏng**. Mean model calls là **3,79**, mean tool calls **2,79**. Các run id có dạng `R-single-C-001-s11`; T02 seed 55 được đối chiếu bằng `R-single-C-002-s55`.

Single và workflow đều từ chối C-002: ticket khai trễ 5 ngày nhưng order chỉ trễ 2 ngày. Single cũng không dùng `SHIP-LATE-OLD` mà ticket C-006 viện dẫn. Ngược lại, swarm seed 55 cố ý tin ticket nên hoàn sai 20.000 VND. Swarm pass 69/70 trial, pass^5 13/14; cost/success 49,57 VND. Bằng chứng này được ghi trong ADR, không sửa swarm tham chiếu để làm mất đối chứng.

Khi hoàn thiện báo cáo nhóm, cổng host đạt **107 passed**, ma trận student toàn nhóm đạt **248 passed**, crash suite đạt **20/20**. Script quét benchmark public kiểm **280 trace mới** cùng stdout/stderr với **42 chuỗi**, cho **0 chuỗi khớp**. Crash suite 20/20 là suite của host và swarm tham chiếu; an toàn crash riêng single được chứng minh thêm bằng T12 và ba ca crash trong `test_single.py`.

## 5. Công cụ AI đã dùng

Codex (OpenAI) hỗ trợ đọc runtime và contract, soạn tài liệu chọn tool, triển khai `single.py`, viết test và tổng hợp số liệu vào ADR/báo cáo. Các công cụ hỗ trợ không được dùng để sửa host, `shapes/common.py`, file thước đo, config hay fixture public để làm test xanh.

Việc kiểm chứng được thực hiện bằng lệnh thật trong môi trường Python của repo: chạy tám test riêng, 62 test public single, cổng host và ma trận student toàn nhóm; đối chiếu trace với order và kết quả benchmark; kiểm tra số liệu trong báo cáo bằng file JSON do công cụ sinh. Không tự sửa JSON để tăng pass rate. Các test mới bảo vệ hành vi đã triển khai; báo cáo không khẳng định đã mutation-test chúng.

## 6. Tự đánh giá

Phần single đáp ứng giới hạn ReAct, quyền chỉ đọc trong callback, tính Budget và ghi refund qua host sau vòng lặp. Các test xác nhận retry, crash/resume, conflict và nhánh kết thúc sớm; quote dùng dữ kiện đã xác minh nên không mắc lỗi T02 seed 55 của swarm.

Hạn chế quan trọng là `decide` hiện chọn action bằng nhánh Python; output MockLLM chưa điều khiển việc chọn tool. Vì vậy kết quả không chứng minh một agent dùng model sản xuất tự lập kế hoạch đạt cùng chất lượng. Tám lượt là số lần gọi decide; mean steps 8,21 trong benchmark là số event trace nên không mâu thuẫn với giới hạn đó.

Trên public, single không tăng pass rate so với workflow nhưng cost/success tăng khoảng **5,76%**; orchestrator tăng khoảng **71,96%**, swarm khoảng **77,93%**. Nhóm chọn workflow vì phù hợp chuỗi nghiệp vụ hiện tại với mức tự chủ thấp nhất. Các kết luận chỉ áp dụng cho 14 claim public, năm seed, MockLLM và giá mô phỏng; latency là unavailable, hidden do giảng viên chấm. Hướng cải thiện là mở rộng dữ liệu kiểm tra và chỉ tăng mức tự chủ khi có lợi ích được đo rõ ràng.
