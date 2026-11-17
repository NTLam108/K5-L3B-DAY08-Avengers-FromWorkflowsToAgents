# Báo cáo cá nhân — 2A202603776_NguyenManhQuan

## 1. Thông tin thành viên

- Họ tên: Nguyễn Mạnh Quân
- MSSV: 2A202603776
- Email: 26ai.quannm9@vinuni.edu.vn
- GitHub: [osgi23](https://github.com/osgi23)
- Nhóm: Avengers (L3A)
- Vai trò: Shape A · Workflow + Tích hợp (`wf`)

## 2. Phần việc phụ trách

- **Phần việc kỹ thuật chính:**
  - Thiết kế và triển khai Shape A (Deterministic Workflow) trong file [`src/lab8/shapes/workflow.py`](../../src/lab8/shapes/workflow.py). Đây là shape được cả nhóm phân tích dữ liệu và thống nhất lựa chọn làm phương án nộp chính thức trong [`decision_adr.md`](decision_adr.md).
  - Viết suite test đơn vị độc lập [`src/tests/test_workflow.py`](../../src/tests/test_workflow.py) để kiểm thử toàn diện các nhánh logic của workflow.
- **Phần việc tích hợp và bảo mật:**
  - Duy trì cổng kiểm thử host `pytest -q` luôn xanh trong suốt quá trình phát triển và tích hợp code của nhóm.
  - Đảm bảo Trace Contract v1: chỉ log các trường đã lọc qua allowlist, không lưu vết raw claim/order row chứa secret hoặc PII (SĐT, địa chỉ, internal note).
  - Đảm bảo an toàn giao dịch tiền tệ (idempotency qua `refund:{claim_id}`) và xử lý lỗi crash (`InjectedCrash` được re-raise để host kiểm soát điểm dừng, không nuốt lỗi).
  - Vận hành và thẩm định độ tương thích bài nộp qua công cụ `python -m lab8.check_submission`.

## 3. Đóng góp cụ thể

- **Triển khai Shape A (`src/lab8/shapes/workflow.py`):**
  - Đọc claim an toàn qua `get_claim_view` (chỉ lấy `claim_id, principal_id, order_id, ticket`).
  - Kiểm tra `order_id`: nếu thiếu hoặc rỗng $\rightarrow$ trả `needs_info`, không ghi ledger.
  - Tra cứu đơn qua `lookup_order`: nếu đơn không tồn tại hoặc thuộc `principal_id` khác $\rightarrow$ trả `escalated` theo luật bảo mật đa khách hàng.
  - Tính tiền độc lập qua `compute_compensation`: chỉ dùng dữ liệu đơn hàng và policy hiện hành (`is_current=1`, `effective_date <= 2026-11-15`), bỏ qua số ngày trễ hoặc chỉ thị người dùng khai khống trong ticket.
  - Nếu quote $\le 0$ $\rightarrow$ từ chối sớm (`status="denied"`), tuyệt đối không gọi `issue_refund`.
  - Nếu quote $> 0$ $\rightarrow$ gọi `durability.issue_refund` với **quote đầy đủ**, truyền `crash_at`. Host sẽ tự trừ các khoản refund trước đó của cùng đơn hàng và áp trần (giải quyết chuẩn xác T13 và T14).
  - Khi gặp `InjectedCrash`, ghi nhận event `crash` vào trace rồi re-raise để kiểm thử crash.
  - Xử lý các kết quả phản hồi từ host: `resolved`, `denied` (do approval hết hạn, approval claim khác, hoặc đơn đã hoàn đủ), hoặc `failed` (do xung đột số tiền `amount_conflict`).
  - Sử dụng `finish()` của `lab8.shapes.common` để đảm bảo mọi nhánh return đều tạo trace hợp lệ có `run_id` đồng nhất.

- **Suite test kiểm thử bổ sung (`src/tests/test_workflow.py`):**
  - `test_workflow_handles_standard_claim`: Kiểm thử luồng claim chuẩn (C-001) trả về `resolved`, proposal 35.000 VND.
  - `test_workflow_missing_order_needs_info`: Kiểm thử claim thiếu mã đơn (C-004) trả về `needs_info`, proposal 0.
  - `test_workflow_outside_principal_scope_escalates`: Kiểm thử claim đơn của khách khác (C-005) trả về `escalated`, proposal 0.
  - `test_workflow_unknown_claim_fails`: Kiểm thử claim không tồn tại trả về `failed`, proposal 0.

- **Quyết định kỹ thuật cốt lõi:**
  - Giữ Shape A thuần deterministic logic, không dùng MockLLM để ra quyết định hoặc tính toán tiền. Điều này giúp loại bỏ hoàn toàn độ bất định của model (phương sai seed = 0) và giảm chi phí LLM về 0 VND.
  - Định hình ranh giới bảo mật nghiêm ngặt cho trace: chỉ lưu trữ allowlist metadata, bảo vệ bí mật canary token và PII của khách hàng.

## 4. Bằng chứng

- **Commit trên repository:**
  - Commit [`da275cb`](https://github.com/NTLam108/K5-L3B-DAY08-Avengers-FromWorkflowsToAgents/commit/da275cb) — `feat(workflow): implement shape A deterministic workflow and unit tests`.
- **Kết quả kiểm thử kỹ thuật:**
  - `python -m pytest -q -m student -o addopts= -k workflow`: **62/62 passed** (bao gồm cả T11 retry idempotency và T12 crash before checkpoint).
  - `python -m pytest -q tests/test_workflow.py`: **4/4 passed**.
  - `python -m pytest -q` (toàn bộ cổng host): **99 passed, 0 failed**.
  - `python -m pytest -q -m student -o addopts=`: **248 passed, 0 failed** (toàn bộ test student của cả 4 shape đều xanh).
- **Kết quả Benchmark (`src/reports/benchmark.json`):**
  - Shape workflow đạt tỷ lệ thành công tuyệt đối: **70/70 trial pass (Pass rate = 1.000, Pass^5 = 1.000)**.
  - Phương sai giữa các seed: **seed_variance = 0.000000** (ổn định 100% qua các seed 11, 22, 33, 44, 55).
  - Chi phí mô phỏng tối ưu nhất trong cả 4 kiến trúc: **27.86 VND / success** (tổng chi phí 1.950 VND cho 70 trial), rẻ hơn đáng kể so với Single (29.46 VND), Orchestrator (47.90 VND) và Swarm (49.57 VND).
  - Số bước trung bình thấp nhất: **2.79 bước**. Số lần gọi model trung bình: **0**.
- **An toàn Crash và Bảo mật:**
  - Crash safety: **20/20** trong [`src/reports/crash.json`](../../src/reports/crash.json).
  - Secret leak: **0 / 70 trial** (quét sạch mọi `CANARY_SECRET_`, không rò rỉ PII trong trace hay kết quả).
- **Thẩm định nộp bài:**
  - `python -m lab8.check_submission`: **0 FAIL, 0 WARN**.

## 5. Công cụ AI đã dùng

- **Công cụ:** Antigravity AI Assistant.
- **Mục đích sử dụng:**
  - Rà soát hợp đồng dữ liệu trong `docs/contracts.md` và các quy tắc nghiệp vụ LogiX-VN trong `docs/business_rules.md`.
  - Hỗ trợ xây dựng các kịch bản kiểm thử biên trong `src/tests/test_workflow.py`.
  - Hỗ trợ phân tích dữ liệu benchmark và đối soát các chỉ số thực nghiệm.
- **Cách tự kiểm tra và kiểm chứng:**
  - Trực tiếp chạy các bộ test tự động (`smoke`, `pytest`, `check_submission`) trên terminal PowerShell môi trường thật.
  - Cố ý tạo các trường hợp kiểm thử biên (sai mã đơn, tráo principal ID, chèn injection vào ticket) để kiểm tra tính chịu lỗi và tính bảo mật của workflow.

## 6. Tự đánh giá

- Hoàn thành xuất sắc 100% nhiệm vụ phụ trách theo đúng tiến độ của các checkpoint (CP1 đến CP5).
- Shape A do tôi phụ trách đạt hiệu quả cao nhất cả về độ chính xác (100%), độ tin cậy (phương sai 0), và chi phí vận hành thấp nhất, là cơ sở vững chắc để nhóm lựa chọn làm kiến trúc nộp bài trong bản ADR.
