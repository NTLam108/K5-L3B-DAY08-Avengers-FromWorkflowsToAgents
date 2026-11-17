# Bonus report: B4

- Topic + nguồn (tên, năm, slide): B4 · Baseline công bằng, ablation và thống kê. *Rethinking Evaluation of MAS Efficiency*, EMNLP 2026 (slide 44, 59); AgentArch, AACL 2026 (slide 43).
- CLAIM của nguồn (1 câu): "Efficiency gain" của hệ multi-agent có thể đến từ cách thiết kế benchmark; random pruning tương tác giữa các agent đôi khi vẫn giữ được hiệu năng, nên phải so với single-agent và baseline đơn giản trong khi giữ cố định runtime, model và tool.
- BOUNDARY (model / dataset / setup): Nguồn đo trên LLM thật và benchmark MAS của paper. Ở đây: MockLLM tất định theo seed, 14 claim public của LogiX-VN, 5 seed, giá token mô phỏng, latency `unavailable`.
- HYPOTHESIS trên LogiX: (H1) Baseline rules-only đạt pass rate bằng workflow vì workflow không dùng model. (H2) Cắt ngẫu nhiên khoảng một nửa lời gọi `peer`/`parse` của swarm không làm giảm pass rate. (H3) Trong workflow, bỏ guard `needs_info` hoặc guard phạm vi đơn làm hỏng đúng claim tương ứng; guard "quote 0 thì không gọi host" bỏ đi không đổi metric. (H4) Với 14 claim, chênh lệch workflow so với swarm không có ý nghĩa thống kê.
- Biến cố định / biến thay đổi: Cố định MockLLM, tool, host, 14 claim `data/public_claims.jsonl`, seed `11,22,33,44,55`, `config/pricing.yaml`, evaluator `lab8.benchmark`, SQLite mới mỗi trial. Thay đổi đúng một biến mỗi lần: shape (baseline) hoặc một thành phần bị bỏ (ablation). Biến thể chạy qua `--extra-shape`; không sửa file shape hay file thước đo.
- Lệnh tái lập (copy-paste được, chạy trong `src/`):

```bash
python -m lab8.benchmark --shapes workflow,swarm,rules_only,swarm_pruned,wf_no_needs_info,wf_no_visibility,wf_no_zero_guard --extra-shape rules_only=bonus.ablation:rules_only --extra-shape swarm_pruned=bonus.swarm_pruned:run_claim --extra-shape wf_no_needs_info=bonus.ablation:no_needs_info_guard --extra-shape wf_no_visibility=bonus.ablation:no_visibility_guard --extra-shape wf_no_zero_guard=bonus.ablation:no_zero_quote_guard --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/bonus_benchmark.json
python -m bonus.stats --bench reports/bonus_benchmark.json --out reports/bonus_ablation.json
python -m pytest -q tests/test_bonus_b4.py
```

- File report + run id: `src/reports/bonus_benchmark.json` (490 trial = 7 shape × 14 claim × 5 seed) và `src/reports/bonus_ablation.json`. Run id theo mẫu `R-<shape>-<claim>-s<seed>`, ví dụ trial fail duy nhất của swarm là `R-swarm-C-002-s55`, của baseline cắt tỉa là `R-swarm_pruned-C-002-s55`.

## Kết quả

Số liệu lấy từ `src/reports/bonus_ablation.json`. CI95 là bootstrap 10.000 vòng, **lấy mẫu lại theo claim** (seed 20261115), vì 5 seed của cùng một claim không độc lập. Cost là VND mô phỏng. Safety: 0 secret leak, 0 PII leak ở cả 7 shape.

**Baseline**

| Shape | Pass rate | CI95 | Pass^5 | Variance | Cost/success | Model calls TB | Refund sai |
| --- | ---: | --- | ---: | ---: | ---: | ---: | ---: |
| workflow (shape chọn) | 1.000 | [1.000, 1.000] | 1.000 | 0 | 27.86 | 0.00 | 0 |
| rules_only | 1.000 | [1.000, 1.000] | 1.000 | 0 | 27.86 | 0.00 | 0 |
| swarm | 0.986 | [0.957, 1.000] | 0.929 | 0.000816 | 49.57 | 3.00 | 1 |
| swarm_pruned | 0.986 | [0.957, 1.000] | 0.929 | 0.000816 | 48.94 | 2.09 | 1 |

**Ablation trên workflow** (mỗi dòng bỏ đúng một thành phần)

| Thành phần bị bỏ | Pass rate | CI95 | Claim hỏng | McNemar (trial) | Sign test (claim) | Metric đổi? |
| --- | ---: | --- | --- | ---: | ---: | --- |
| `needs_info_guard` | 0.929 | [0.786, 1.000] | C-004: `escalated` thay vì `needs_info` | p = 0.0625 (5–0) | p = 1.000 (1–0) | Có |
| `visibility_guard` | 0.929 | [0.786, 1.000] | C-005: `denied` thay vì `escalated` | p = 0.0625 (5–0) | p = 1.000 (1–0) | Có |
| `zero_quote_guard` | 1.000 | [1.000, 1.000] | không | p = 1.000 (0–0) | p = 1.000 | Không |

**Kiểm định cặp workflow so với swarm:** chênh pass rate 0.014, CI95 của chênh lệch [0.000, 0.043]; McNemar trên 70 cặp (claim, seed) có 1 cặp lệch (1–0), p = 1.000; sign test trên 14 claim có 1 claim lệch, 13 hòa, p = 1.000.

## Kết luận

**Chênh lệch workflow và swarm không có ý nghĩa thống kê** trên bộ public (p = 1.000 ở cả hai kiểm định, CI chứa 0). Toàn bộ chênh lệch đến từ một cặp duy nhất `C-002`/seed 55. Muốn sign test theo claim đạt p < 0.05 cần ít nhất 6 claim lệch cùng chiều (2 × 0.5⁶ = 0.031); 14 claim hiện có không đủ lực để kết luận. Lý do chọn workflow vì thế không dựa vào pass rate mà vào **chi phí** (27.86 so với 49.57 VND/success) và **rủi ro tiền**: lỗi duy nhất của swarm là hoàn tiền khi không được hoàn.

**H1 đúng.** `rules_only` trùng workflow ở mọi trial (0 cặp lệch), và test kiểm tra 14/14 claim cho cùng quyết định, số tiền và audit. Workflow vốn không gọi model, nên "draft model call" và "investigator" không phải thành phần của shape này; giá của chúng là khoảng cách workflow → single/orchestrator (xem `research_card.md`).

**H2 đúng, khớp CLAIM của nguồn.** Cắt ngẫu nhiên lời gọi `peer`/`parse` giảm 30% số model call (3.00 → 2.09) mà pass rate, pass^5 và variance giữ nguyên. Nhưng cost/success chỉ giảm 1.3% (49.57 → 48.94), vì trong bảng giá này chi phí do tool call chi phối (10 VND/call). Khi đồng xu bỏ `parse` thì lỗi seed 55 biến mất (test `test_dropping_parse_removes_the_seed55_override`); ở seed 55 của C-002, đồng xu tình cờ giữ `parse`, nên lỗi vẫn còn. "Hiệu quả" của việc cắt tỉa vì vậy phụ thuộc may rủi của đồng xu, không phải thiết kế.

**H3 đúng một phần.** Hai guard `needs_info` và phạm vi đơn là cần thiết: bỏ đi thì hỏng đúng một claim và không làm thay đổi claim nào khác (test kiểm tra 14 claim). `zero_quote_guard` bỏ đi không đổi bất kỳ metric nào, **nhưng hành vi có đổi**: host ghi thêm audit `policy_denied` (lý do `non_positive_amount`) cho mọi claim có quote 0. Evaluator chỉ kiểm audit bắt buộc có mặt, không kiểm audit thừa, nên metric không thấy. "Bỏ đi mà metric không đổi" không có nghĩa là thành phần vô dụng; nhóm giữ guard này để audit log chỉ ghi những lần host thực sự chặn.

**Điều kiện đảo kết luận:** nếu bộ claim lớn hơn và có nhiều ticket nói sai số ngày, lỗi của swarm sẽ lặp ở nhiều claim và chênh lệch có thể trở nên có ý nghĩa. Nếu bảng giá đổi sang token đắt, tool rẻ, lợi ích chi phí của việc cắt tỉa sẽ lớn hơn nhiều so với 1.3%. Nếu evaluator kiểm cả audit thừa, ablation `zero_quote_guard` sẽ không còn "miễn phí".

## Giới hạn

- MockLLM tất định: với workflow, 5 seed cho cùng kết quả, nên McNemar trên cặp (claim, seed) đếm một lỗi logic thành 5 lần (p = 0.0625) và phóng đại bằng chứng. Nhóm báo kèm sign test theo claim (p = 1.000) và bootstrap theo claim để tránh lỗi này.
- 14 claim public: CI rộng (đến [0.786, 1.000]) và không đủ lực thống kê; kết quả hidden do giảng viên chấm.
- Pass rate 1.000 bị trần (ceiling), không phân biệt được workflow với rules_only hay biến thể không có `zero_quote_guard`.
- Chi phí là VND mô phỏng, latency `unavailable`; không kết luận về tốc độ.
- `swarm_pruned` dùng đồng xu xác suất 0.5 cố định theo (seed, claim, purpose); chưa quét các mức cắt khác.
