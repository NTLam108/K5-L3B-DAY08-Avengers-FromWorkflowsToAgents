# Decision record

Điền sau dấu `:` trên cùng dòng, không xóa nhãn. Giảng viên và `python -m lab8.check_submission` (chạy trong `src/`) đọc các dòng này bằng máy. Không đổi tên hay dời file này khỏi `report/team/`.
`Shape chọn` phải là đúng một trong: `workflow`, `single`, `orchestrator`, `swarm`. Đây là shape được chấm cổng hidden.

- Shape chọn: workflow
- Commit hash: a152042
- Run id dẫn chứng: R-workflow-C-001-s11 … R-workflow-C-014-s55 (70 trial = 14 claim × seed 11,22,33,44,55; src/reports/benchmark.json)
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000–1.000 / 0.000000 (public, 70 trial)
- Cost per success (mô phỏng, VND): 27.86 (tổng 1950 VND / 70 trial pass)
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0 / 0 (70/70 trial lý do `ok` trong src/reports/benchmark.json; không file nào trong src/traces/ chứa `CANARY_SECRET_`)

Ví dụ một dòng đã điền: `- Shape chọn: workflow`, `- Crash safety: 20/20 (src/reports/crash.json)`.

## Vì sao shape này

Cả bốn shape chạy trên cùng 14 claim public, cùng 5 seed (11, 22, 33, 44, 55), cùng MockLLM, cùng bảng giá và cùng evaluator (`src/reports/benchmark.json`). Workflow, single và orchestrator cùng đạt pass rate 1.000, pass^5 1.000, variance 0. Swarm đạt 0.986 / 0.929, variance 0.000816, và fail một trial: seed 55, claim C-002 (T02), trả `resolved` thay vì `denied` vì tin số ngày trễ ghi trong ticket (5 ngày) thay cho cột `delay_days` của đơn (2 ngày). Khi chất lượng ngang nhau thì yếu tố quyết định là chi phí và mức tự chủ. Workflow rẻ nhất (27.86 VND/success, so với 29.46 của single, 47.90 của orchestrator và 49.57 của swarm), ít bước nhất (mean 2.79), không gọi model (mean model calls 0), và có mức tự chủ thấp nhất. Nhóm chọn workflow vì đây là kiến trúc ít tự chủ nhất mà vẫn làm đúng việc.

## Có đáng đổi lấy multi-agent không

Không, trên bộ public. Orchestrator không tăng pass rate (1.000 so với 1.000) nhưng tăng cost/success thêm khoảng 72% (47.90 so với 27.86) và tăng số bước (4.71 so với 2.79). Swarm vừa đắt hơn (49.57) vừa kém ổn định hơn (pass^5 0.929). Claims flow là chuỗi tuần tự: phải có order mới quote được, có quote mới gọi host. Vì vậy thêm investigator không cắt được critical path. Quyết định sẽ đảo lại nếu: (1) ticket thật mơ hồ đến mức phải có model đọc hiểu mới trích được dữ kiện, và workflow bắt đầu fail các claim đó; (2) có nhiều nguồn dữ liệu độc lập cần tra song song, ví dụ hãng vận chuyển hay kho, làm latency trở thành ràng buộc; (3) bộ hidden cho thấy workflow dưới 0.75 trong khi single hoặc orchestrator vượt ngưỡng mà vẫn giữ 3 must-pass.

## Trace và bằng chứng

Trace workflow ghi 3 loại event: `lookup_order` (chỉ `visible` và `reason`), `quote` (`amount_vnd`, `reason`, `clause_ids`) và `decision` (`status`, `proposal_amount`). Trace orchestrator ghi thêm `handoff` đã sanitize bằng `make_handoff` và `investigator` (order view đã qua allowlist của `lookup_order`, số tiền đã hoàn của đơn). Các field cố ý **không** ghi: nguyên văn ticket, claim row thô (có `parent_secret`), order row thô (có SĐT, địa chỉ, tên hàng, internal note) và parent context. Lý do: trace là ranh giới bảo mật, grader quét trace và mọi file `<run_id>*` cùng thư mục. Mọi trace nằm ở `src/traces/<run_id>.json`, đúng chỗ scanner quét; shape không in dữ liệu ra stdout và không ghi file ở chỗ khác. Bằng chứng: 70/70 trial workflow có lý do `ok` (lý do này đòi hỏi trace không chứa chuỗi cấm), và lệnh quét `CANARY_SECRET_` trên `src/traces/` không trả về file nào.

## Giới hạn

MockLLM không phải model sản xuất: hành vi model ở đây là tất định theo seed, nên pass rate 1.000 trên public không chứng minh workflow đúng với ticket thật. Latency để `unavailable`; cost là VND mô phỏng, không phải hóa đơn API. Bộ public chỉ có 14 claim; con số hidden (40 claim × 5 seed) do giảng viên chạy. Không sửa JSON trong `src/reports/` cho đủ 0.75.
