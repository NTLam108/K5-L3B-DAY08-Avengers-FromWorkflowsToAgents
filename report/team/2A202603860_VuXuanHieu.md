# Báo cáo cá nhân — Vũ Xuân Hiếu

## 1. Thông tin thành viên

- Họ tên Vũ Xuân Hiếu
- MSSV 2A202603860
- Email [26ai.hieuvx2@vinuni.edu.vn](mailto:26ai.hieuvx2@vinuni.edu.vn)
- GitHub `vuhieuvh`

## 2. Phần việc phụ trách

Kiểm tra C3 Durability và C4 Policy, rà trace và stdout ở các checkpoint CP2–CP5.

## 3. Đóng góp cụ thể

- **CP2.** Bổ sung test T13 và T14 chạy qua `run_claim` của cả ba shape trong `src/tests/test_refund_safety_matrix.py`. T13 kiểm tra audit `order_already_refunded` và tổng tiền hoàn **35.000 VND**. T14 kiểm tra quote **150.000 VND**, refund bổ sung **120.000 VND** và tổng tiền hoàn **150.000 VND**. Rà ba triển khai theo ma trận T01–T14 trên `main`.
- **CP3.** Chạy demo `buggy_workflow` và giải thích trong báo cáo nhóm vì sao crash giữa lúc ghi tiền và ghi checkpoint có thể gây hoàn trùng. Kiểm tra ba lần gọi cùng claim trên mỗi shape vẫn chỉ có một dòng ledger với khóa `refund:{claim_id}` và một audit `refund_committed`.
- **CP4.** Kiểm tra Workflow tại cả ba điểm crash trên đơn đã hoàn một phần. Thử lại các ca approval thiếu, hết hạn, bị thu hồi và thuộc claim khác. Mỗi ca cho **0 refund** và đúng một audit `policy_denied`.
- **CP5.** Quét 280 trace mới và đầu ra console từ benchmark public. Đối chiếu file đã được Git theo dõi và bổ sung giới hạn về MockLLM, chi phí mô phỏng và latency trong báo cáo nhóm.

## 4. Bằng chứng

- Commit test `f4221af`.
- `python -m pytest -q` có **75 pass**. Ma trận public có **248 pass**, gồm **62 test** cho mỗi shape.
- `python -m lab8.demo.buggy_workflow` in `unsafe refund rows: 2`.
- [Báo cáo crash](../../src/reports/crash.json) đạt **20/20**. Lần chạy lại cho cùng kết quả trên từng kịch bản.
- Từ `src/`, chạy `python scripts/scan_public_benchmark.py` để sinh và quét lại **280 trace mới**, stdout và stderr với **42 chuỗi tìm kiếm**. Kết quả có **0 chuỗi khớp**. [Script quét CP5](../../src/scripts/scan_public_benchmark.py)
- Các lệnh chạy lại và kết quả từng ca nằm trong [báo cáo nhóm](group_report.md#c3--idempotency-và-khôi-phục-sau-crash).

## 5. Công cụ AI đã dùng

Codex hỗ trợ rà luồng refund, bổ sung test và tổng hợp báo cáo. Các kiểm chứng gồm chạy pytest, đối chiếu số tiền và audit trong SQLite, chạy lại crash suite và quét trace mới.

## 6. Tự đánh giá

Tôi đã kiểm tra trần hoàn tiền và tính idempotent qua `run_claim` của cả ba shape. Các test đối chiếu kết quả trả về với số tiền, số dòng ledger và audit trong SQLite. Với Workflow, tôi kiểm thêm cả ba điểm crash và bốn dạng approval không hợp lệ khi gọi lại. Script CP5 giúp người chấm chạy lại phép quét trace, stdout và stderr trên dữ liệu public.

Phần kiểm thử approval của tôi bao phủ các trạng thái đã có trước khi gọi claim. Tôi chưa bổ sung test hồi quy cho tình huống approval bị thu hồi giữa lúc host kiểm tra và ghi refund.

### Phạm vi kiểm chứng

Kết quả trên đây áp dụng cho bộ public. Kết quả hidden do giảng viên chấm. MockLLM chạy theo seed và latency của benchmark là `unavailable`.
