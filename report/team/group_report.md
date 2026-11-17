# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm:
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Nhóm hiểu bài toán thế nào, chia việc ra sao, thứ tự làm bốn shape.

...

## 3. Giải pháp

Mô tả ba shape nhóm viết (workflow, single, orchestrator): mỗi shape đọc gì, quyết định gì, gọi host ra sao. Chỉ rõ ranh giới code / model (tham khảo [docs/architecture.md](../../docs/architecture.md)), cách handoff được sanitize và trace ghi những event nào.

### Shape A — Workflow

...

### Shape B — Single agent

...

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

Bảng so sánh 4 shape trên bộ public (từ `src/reports/benchmark.json`): pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps. Kèm crash safety (`src/reports/crash.json`) và, nếu có, kết quả tự kiểm trên `student_drop/`.

...

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

...

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

Benchmark dùng MockLLM tất định theo seed và giá token mô phỏng. Lab ghi latency là `unavailable`. Kết quả trên 14 claim public đo tính đúng, số bước và chi phí mô phỏng trong cấu hình này. Chất lượng và thời gian đáp ứng khi dùng model sản xuất cần được đo riêng.

Single dùng vòng ReAct tối đa tám bước, với action được chọn bằng nhánh Python dựa trên kết quả tool. Lời gọi MockLLM tính vào chi phí nhưng output chưa điều khiển action. Khi so sánh kiến trúc, cần đọc kết quả Single theo cách triển khai này.

## 8. Tài liệu tham khảo

...
