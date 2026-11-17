# Báo cáo cá nhân — 2A202603776_NguyenManhQuan

## 1. Thông tin thành viên

- Họ tên: Nguyễn Mạnh Quân
- MSSV: 2A202603776
- Email: manhquan03122005@gmail.com
- GitHub: osgi23

## 2. Phần việc phụ trách

- **Vai trò:** Shape A · Workflow + Tích hợp (`wf`).
- **Phạm vi trách nhiệm:**
  - Thiết kế và lập trình kiến trúc Shape A (Deterministic Workflow) trong file `src/lab8/shapes/workflow.py`.
  - Giữ cổng host `pytest -q` xanh trong suốt quá trình phát triển và tích hợp.
  - Đảm bảo tuân thủ Trace Contract v1, không leak secret hay PII vào trace/kết quả.
  - Đảm bảo xử lý an toàn lỗi crash (`InjectedCrash`) và tính toán tiền độc lập bằng code nghiệp vụ, không phụ thuộc MockLLM.
  - Viết test kiểm thử đơn vị cho shape workflow trong `src/tests/test_workflow.py`.
  - Phụ trách quản lý tích hợp, chạy kiểm tra `python -m lab8.check_submission` và chuẩn bị chốt bài nộp của nhóm.

## 3. Đóng góp cụ thể

- **Code chính:**
  - Hoàn thành toàn bộ `src/lab8/shapes/workflow.py`:
    - Truy vấn thông tin claim qua `get_claim_view`, kiểm tra `order_id` hợp lệ (trả về `needs_info` nếu thiếu).
    - Tra cứu đơn hàng an toàn qua `lookup_order` với kiểm tra phạm vi principal (trả về `escalated` nếu không hợp lệ).
    - Tính số tiền bồi thường chính xác qua `compute_compensation` từ cơ sở dữ liệu đơn hàng (không để MockLLM quyết định tiền, bỏ qua số ngày trễ người dùng khai trong ticket).
    - Từ chối sớm (`status="denied"`) nếu quote = 0, không gọi lệnh ghi ledger.
    - Với quote > 0, gọi `durability.issue_refund` với quote đầy đủ, truyền tham số `crash_at` và để ngoại lệ `InjectedCrash` bung ra theo đúng chuẩn kiểm thử crash.
    - Ghi nhận trạng thái `resolved`, `denied` (khi host từ chối do approval hoặc trùng đơn), hoặc `failed` (khi conflict số tiền).
- **Test bổ sung:**
  - Viết `src/tests/test_workflow.py` với 4 test case:
    - `test_workflow_handles_standard_claim`: Kiểm thử luồng claim chuẩn (C-001).
    - `test_workflow_missing_order_needs_info`: Kiểm thử claim thiếu mã đơn (C-004).
    - `test_workflow_outside_principal_scope_escalates`: Kiểm thử claim trái quyền sở hữu đơn hàng (C-005).
    - `test_workflow_unknown_claim_fails`: Kiểm thử claim không tồn tại trong hệ thống.
- **Quyết định kỹ thuật:**
  - Giữ shape A thuần deterministic workflow, 0 model call nhằm đạt chi phí tối thiểu (0 VND chi phí LLM) và phương sai seed = 0.
  - Tuân thủ nghiêm ngặt Trace Contract v1: chỉ log metadata và trường đã lọc qua allowlist, tuyệt đối không log raw claim row hoặc raw order row chứa số điện thoại/địa chỉ.

## 4. Bằng chứng

- **Test suite sinh viên:**
  - `python -m pytest -q -m student -o addopts= -k workflow`: 62/62 passed (bao gồm cả T11 idempotency và T12 crash recovery).
- **Test cổng host:**
  - `python -m pytest -q`: 58 passed.
  - `python -m pytest -q tests/test_workflow.py`: 4/4 passed.
- **Submission check:**
  - `python -m lab8.check_submission`: `[ OK ] shape workflow: format đúng, 12/12 claim public pass ở seed 11`.

## 5. Công cụ AI đã dùng

- **Công cụ:** Antigravity AI Assistant.
- **Mục đích sử dụng:** Hỗ trợ đọc contract kỹ thuật, rà soát đối chiếu các trường trong database schema/trace contract, và tự động hóa kiểm thử cục bộ.
- **Cách tự kiểm tra kết quả:** Trực tiếp chạy các suite kiểm thử chính thức của môn học (`smoke`, `pytest`, `check_submission`) trên terminal, đối chiếu từng assertion với tài liệu `docs/contracts.md` và `docs/business_rules.md`.

## 6. Tự đánh giá

- Hoàn thành đầy đủ 100% mục tiêu của Shape A từ CP1 đến CP3.
- Cổng host và toàn bộ 62 test case dành cho workflow đều đạt kết quả tuyệt đối.
