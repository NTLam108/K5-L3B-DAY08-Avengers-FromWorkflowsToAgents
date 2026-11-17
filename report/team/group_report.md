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

...

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

Lần chạy lại benchmark public sinh **280 trace mới**. Quét toàn bộ các trace này cùng stdout và stderr theo tiền tố `CANARY_SECRET_`, `CANARY_INTERNAL_` và các giá trị số điện thoại, địa chỉ, ghi chú nội bộ, tên hàng trong bộ dữ liệu public. Kết quả có **0 chuỗi khớp**. Git index có **0** tệp thuộc `traces/`, tệp SQLite hoặc tệp cấu hình `.env`.

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
