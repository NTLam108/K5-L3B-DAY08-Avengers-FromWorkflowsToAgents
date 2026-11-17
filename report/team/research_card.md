# Research card

File này nằm cố định ở `report/team/research_card.md`; `python -m lab8.check_submission` đọc cột "Nhóm điền" bằng máy.

Phần này nối tiếp phần mini research ở buổi lý thuyết (slide 62–63). Chọn **một** câu R1–R4, điền 6 dòng, rồi biến nó thành **một ablation** chạy được trong lab: giữ nguyên model (MockLLM), tool, seed, corpus và giá, chỉ thay đổi một biến.

| Câu | Nguồn (thư viện tham khảo, slide 68–69) | Câu hỏi gợi ý cho lab |
| --- | --- | --- |
| R1 · Scaling | Google Research 2026, *Towards a Science of Scaling Agent Systems* | Task claims chủ yếu tuần tự hay song song được? Dự đoán thêm investigator sẽ làm pass rate và cost/success thay đổi thế nào. |
| R2 · Failure taxonomy | MAST, NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* | Các trial fail của swarm (seed 55) thuộc nhóm nào: lỗi giao việc, lệch mục tiêu, hay lỗi kiểm chứng? |
| R3 · Architecture choices | AgentArch, AACL 2026 | Nếu đổi topology mà giữ nguyên mọi thứ khác, biến nào phải cố định để kết luận còn đúng? |
| R4 · Evaluation confounders | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* | Làm sao chứng minh chênh lệch giữa A/B/C/D là do topology, không phải do prompt, tool hay seed? |

## Phiếu của nhóm

Điền vào cột cuối. Không xóa hai cột đầu.

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R1 · Scaling |
| SOURCE | Paper/blog, năm, URL, loại bằng chứng (benchmark, production report, blog kinh nghiệm) | Google Research 2026, *Towards a Science of Scaling Agent Systems* (thư viện tham khảo slide 68–69, trình bày ở slide 15, 33, 41). Loại bằng chứng: benchmark có kiểm soát trên nhiều cấu hình agent |
| CLAIM | Một kết luận gốc của nguồn. Không viết rộng hơn phạm vi paper | Multi-agent chỉ có lợi khi task chia song song được; với task tuần tự, các biến thể multi-agent trong nghiên cứu giảm hiệu năng 39–70% |
| BOUNDARY | Model / dataset / setup giới hạn kết luận đó | Kết luận đo trên LLM thật và bộ task của nghiên cứu, không phải claims flow của LogiX-VN hay MockLLM; con số 39–70% chỉ đúng trong setup đó |
| HYPOTHESIS | Nếu đổi shape A→B (hoặc bật/tắt một thành phần), metric X sẽ đổi theo hướng nào, khoảng bao nhiêu | Claims flow là chuỗi tuần tự (order → quote → host). Đổi A `workflow` → C `orchestrator` (thêm investigator): pass rate không tăng (giữ ~1.0 trên public), cost/success tăng khoảng +50% đến +100%, mean steps tăng |
| METHOD | Biến giữ cố định: model, tools, seed, corpus, giá, evaluator. Biến duy nhất được thay đổi: … | Cố định: MockLLM, cùng read tools và host, seed 11,22,33,44,55, 14 claim `data/public_claims.jsonl`, `config/pricing.yaml`, evaluator `lab8.benchmark`, SQLite mới mỗi trial. Biến duy nhất thay đổi: topology A `workflow` → C `orchestrator` (thêm lớp orchestrator + investigator chỉ đọc qua handoff) |
| FAILURE / METRIC | Một lỗi dự kiến và cách đo nó: pass rate, pass^5, variance, cost/success, leak, bypass | Lỗi dự kiến: thêm agent làm tăng chi phí mà không tăng chất lượng (overhead phối hợp). Đo bằng cost/success và mean steps, kèm pass rate, pass^5, variance; safety: secret leak và approval bypass phải giữ 0 |

## Kết quả ablation

- Lệnh đã chạy (copy nguyên văn): `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json`
- Run id / file report: `src/reports/benchmark.json` (commit a152042); trace `R-workflow-C-001-s11` … `R-workflow-C-014-s55` và `R-orchestrator-C-001-s11` … `R-orchestrator-C-014-s55`
- Kết quả so với hypothesis (đúng / sai / không kết luận được), kèm số liệu: **Đúng.** Pass rate workflow 1.000 → orchestrator 1.000 (không đổi), pass^5 1.000 → 1.000, variance 0 → 0. Cost/success 27.86 → 47.90 VND (+72%), mean steps 2.79 → 4.71, mean model calls 0 → 2. Đối chứng: swarm 0.986 / pass^5 0.929 / 49.57 VND, fail trial `R-swarm-C-002-s55` vì tin số ngày trễ trong ticket thay vì `delay_days` của đơn. Trên task tuần tự này, thêm agent chỉ thêm chi phí, khớp hướng claim của nguồn; mức độ giảm không so trực tiếp được với 39–70% vì metric khác nhau.
- Giới hạn: MockLLM không phải model sản xuất, latency là `unavailable`. Chỉ 14 claim public nên pass rate 1.000 bị trần (ceiling), không phân biệt được workflow với orchestrator về chất lượng; cost là VND mô phỏng.

Muốn đi xa hơn một ablation để lấy điểm cộng thì xem [docs/bonus_topics.md](../../docs/bonus_topics.md).
