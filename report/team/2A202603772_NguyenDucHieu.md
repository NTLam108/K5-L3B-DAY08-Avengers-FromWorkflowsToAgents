# Báo cáo cá nhân — 2A202603772_NguyenDucHieu

## 1. Thông tin thành viên

- Họ tên: Nguyễn Đức Hiếu
- MSSV: 2A202603772
- Email: 26ai.hieund6@vinuni.edu.vn
- GitHub: [Hie24](https://github.com/Hie24)
- Nhóm: Avengers
- Vai trò: Benchmark · Đánh giá · Báo cáo (Eval + Report)

## 2. Phần việc phụ trách

Phụ trách chạy baseline Swarm, đọc các chỉ số trong `docs/metrics.md`, bổ sung
test về tính công bằng của benchmark và kiểm chứng kết quả của bốn kiến trúc
sau khi nhóm tích hợp code. Dùng câu nghiên cứu R1 và quyết định kiến trúc trong
research card, ADR đã được nhóm chốt để đối chiếu số liệu.

## 3. Đóng góp cụ thể

### Baseline Swarm và phân tích lỗi

- Chạy Swarm trên 14 claim public với seed `11,22,33,44,55`, tạo
  `src/reports/benchmark_swarm_baseline.json` bằng công cụ của lab.
- Đối chiếu pass rate, pass^5, phương sai giữa các seed và cost/success.
  Cost/success tính cả chi phí của trial fail; variance dùng phương sai tổng thể.
- Lần theo `C-002` ở seed 11 và 55. Order có `delay_days=2`; ở seed 55,
  event `model_override` thay thành 5 theo ticket, làm quote thành 20.000 VND
  và trả `resolved` thay vì `denied`. Đây là điểm yếu cố ý của Swarm tham chiếu.

### Ba test bổ sung cho phần Eval

Thêm [test_eval_fairness.py](../../src/tests/test_eval_fairness.py):

| Test | Mục đích |
| --- | --- |
| `test_every_trial_starts_with_only_seeded_refunds` | Mỗi trial dùng đường dẫn SQLite riêng và khởi đầu chỉ với refund lịch sử đã seed; kết quả không hưởng refund của trial trước. |
| `test_reordering_seeds_preserves_trial_results_and_metrics` | Đảo thứ tự seed vẫn giữ nguyên kết quả từng cặp claim–seed và các chỉ số tổng hợp; cho phép sai số cộng số thực ở chi phí. |
| `test_repeated_benchmark_reproduces_results_and_simulated_cost` | Chạy lại cùng cấu hình tái lập được kết quả, chi phí và lỗi C-002/55; tính ổn định không đồng nghĩa mọi trial đều đúng. |

Các test dùng Swarm tham chiếu để chạy được trước khi ba shape của nhóm hoàn
thành. Trace của test được ghi vào thư mục tạm của pytest để tách khỏi trace
baseline. Các file thước đo, config và dữ liệu public được giữ nguyên.

### Kiểm chứng kết quả sau tích hợp

Chạy lại benchmark bốn shape trên cùng 14 claim và năm seed, tạo file kiểm tra
riêng `src/reports/benchmark_hieu_check.json`. Đối chiếu toàn bộ nội dung JSON
sau khi parse, gồm biến kiểm soát, thống kê và 280 trial, với
`src/reports/benchmark.json` của nhóm: kết quả khớp hoàn toàn.
File kiểm tra trùng kết quả được giữ local; bằng chứng nộp chung dùng file
`benchmark.json` đã có trên main.

## 4. Bằng chứng

- Commit trên nhánh cá nhân:
  [7110ea4](https://github.com/NTLam108/K5-L3B-DAY08-Avengers-FromWorkflowsToAgents/commit/7110ea41b2ccd069db2a7ccf6c617ad53d606822).
- Commit phần Eval đã được đưa vào main:
  [4c947f9](https://github.com/NTLam108/K5-L3B-DAY08-Avengers-FromWorkflowsToAgents/commit/4c947f95d7ff5e00493e3d8e52aff3b53d436cb5).
  Hai commit chứa cùng nội dung test Eval và baseline Swarm.
- [Baseline Swarm](../../src/reports/benchmark_swarm_baseline.json): **69/70**
  trial pass, pass rate **98,5714%**, pass^5 **92,8571%**, seed variance
  **0,0008163265**, cost/success **49,5669565 VND mô phỏng**.
- Run id đối chiếu: `R-swarm-C-002-s11` (pass, `denied`) và
  `R-swarm-C-002-s55` (fail, `resolved`). Trace được sinh trên máy chạy và nằm
  trong thư mục được Git bỏ qua theo quy định lab.

Kết quả kiểm chứng từ [benchmark nhóm](../../src/reports/benchmark.json):

| Shape | Trial pass | Pass rate | Pass^5 | Seed variance | Cost/success (VND mô phỏng) |
| --- | ---: | ---: | ---: | ---: | ---: |
| Workflow | 70/70 | 100% | 100% | 0 | 27,86 |
| Single | 70/70 | 100% | 100% | 0 | 29,46 |
| Orchestrator | 70/70 | 100% | 100% | 0 | 47,90 |
| Swarm | 69/70 | 98,57% | 92,86% | 0,0008163265 | 49,57 |

Các lệnh đã chạy trong `src/`:

```powershell
.\.venv\Scripts\python.exe -m lab8.benchmark --shapes swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark_swarm_baseline.json
.\.venv\Scripts\python.exe -m pytest -q -o addopts=
.\.venv\Scripts\python.exe -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark_hieu_check.json
```

Lần kiểm tra toàn bộ test sau khi nhận các test mới của nhóm ngày 17/11/2026
đạt **347 passed**, bao gồm ba test Eval. Cây mã được kiểm tra có cùng nội dung
với main tại `4c947f9`. Đây là kết quả của toàn bộ suite tích hợp, không phải
số test do cá nhân bổ sung.

## 5. Công cụ AI đã dùng

Sử dụng **OpenAI Codex** giải thích metric,
thực thi benchmark và test, đối chiếu JSON, quản lý thay đổi Git và tổng hợp báo
cáo từ bằng chứng thực tế.

Việc kiểm chứng được thực hiện bằng các lệnh trên môi trường Python của lab. Số liệu trong báo cáo lấy từ file do công cụ sinh và output lệnh đã chạy; các file JSON không sửa tay.

## 6. Tự đánh giá

Phần Eval đã có sản phẩm cụ thể: baseline 70 trial, ba test kỹ thuật và lần
kiểm chứng 280 trial khớp kết quả nhóm. Các bằng chứng này hỗ trợ tính công
bằng và khả năng tái lập của phép đo, đồng thời cho thấy cần báo cáo cả
pass rate lẫn pass^5 để nhận ra lỗi phụ thuộc seed.

Phạm vi kiểm chứng hiện là bộ public. MockLLM tất định theo seed, chi phí là
mô phỏng và latency là `unavailable`; các số đo này chưa chứng minh hiệu quả
trên model sản xuất hoặc bộ hidden. Các nhận định kiến trúc trong báo cáo
nhóm sử dụng [ADR](decision_adr.md) và [R1 đã chốt](research_card.md).
