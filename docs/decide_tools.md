# Quy ước dùng chung: decide, tool và Budget

Đối chiếu từ [runtime.py](../src/lab8/runtime.py), [schemas.py](../src/lab8/schemas.py), [swarm_reference.py](../src/lab8/shapes/swarm_reference.py), [tools.py](../src/lab8/tools.py) và [mock_llm.py](../src/lab8/mock_llm.py). `Budget` được định nghĩa trong schemas.py rồi import vào runtime.

## Contract có sẵn của run_react

Shape truyền connection, một Budget dùng chung cho cả lần chạy, observation, callback `decide` và `max_steps=8`. Mỗi lượt, runtime gọi `decide(seen)`. Callback trả một trong hai dạng:

```python
{"tool": "lookup_order", "args": {"order_id": "O-107", "principal_id": principal_id}}
```

```python
{"final": {"proposal_amount": 35000}}
```

Ví dụ final là payload nội bộ nhóm có thể quy ước, **chưa phải kết quả đã ghi tiền**. Runtime không kiểm schema final, không xác minh số tiền và không chuyển nó thành RunResult. Shape phải lấy proposal từ output thật của `compute_compensation`, xử lý host rồi dùng `finish()`.

| Tình huống | Hành vi runtime |
| --- | --- |
| Action có khóa `final` | Dừng ngay, trả `stopped="final"`, final và observation hiện tại; không dispatch tool |
| Action chọn tool | Dispatch, sau đó chuyển kết quả gần nhất cho decide |
| Hết lượt | Trả `stopped="step_cap"`, `final={"status": "failed", "error_reason": "step_cap"}` |
| Tool ghi, tool lạ, args sai | Có thể ném exception; runtime không tự bắt rồi thử lại |

**Giới hạn 8 tính số lần gọi decide, kể cả lần trả final.** Để có final trong vòng, chỉ dùng tối đa 7 lượt tool trước đó. Tool ở lượt thứ 8 vẫn chạy nhưng sau đó vòng trả step_cap, không gọi decide lần thứ 9.

Sau mỗi tool, runtime thay toàn bộ observation bằng:

```python
{
    "last_tool": name,
    "last_result": result,
    "claim": observation.get("claim"),
}
```

`claim` luôn lấy từ observation **ban đầu**, không tự cập nhật từ output `get_claim_view`. Runtime không tích lũy order/policy/quote, không giữ lịch sử tool, không gọi model và không ghi trace thay callback. Dữ kiện cần dùng lại phải lưu trong state riêng của từng lần `run_claim`, không dùng cache toàn cục chung claim/seed.

## Quy ước chọn tool của nhóm

Đây là luồng đề xuất khi viết `single`, chưa được triển khai trong stub. Ưu tiên shape đọc `get_claim_view` trước vòng, tính tool cost lần đọc đó, rồi truyền `observation={"claim": claim_view}`. Lấy ID và principal từ claim, không lấy từ ticket/model.

| Dữ kiện hiện có | Quyết định tiếp theo |
| --- | --- |
| Claim `found=false` | Final lỗi unknown_claim; shape trả failed, proposal 0 |
| Thiếu order_id | Final needs_info, proposal 0; không ghi refund |
| Chưa có order view | lookup_order với order_id và principal_id của claim |
| Order `visible=false` | Final escalated, proposal 0; giữ reason từ tool |
| Order hợp lệ, cần policy để giải thích | lookup_policy một lần, lưu kết quả |
| Order hợp lệ, chưa có quote | compute_compensation với order nguyên vẹn từ lookup_order |
| Quote amount_vnd ≤ 0 | Final denied, proposal 0; không gọi host ghi tiền |
| Quote amount_vnd > 0 | Final mang quote đầy đủ để code của shape gọi durability.issue_refund |

Luồng ngắn: `lookup_order → compute_compensation → final`. Khi cần giải thích policy: `lookup_order → lookup_policy → compute_compensation → final`. `compute_compensation` đã tự đọc policy hiện hành bằng code nên lookup_policy không bắt buộc để tính quote. Dùng kết quả đã có, tránh gọi lặp tool khi không có dữ kiện mới.

| Tool được dispatch hỗ trợ | Args | Lưu ý |
| --- | --- | --- |
| get_claim_view | claim_id | Nếu đọc trong vòng, phải lưu output vào state; runtime không tự cập nhật claim |
| lookup_order | order_id, principal_id | Kiểm visible trước khi dùng order |
| lookup_policy | `{}` | Chỉ trả policy hiện hành có hiệu lực |
| compute_compensation | order | Dùng order view thật, không sửa delay_days theo ticket |
| lookup_refund_history | order_id, principal_id | Có thể dùng để giải thích lịch sử; không tự trừ quote trước khi gọi host |
| get_clause | clause_id | Có thể trả clause cũ/tương lai; kiểm is_current và effective_date |

`worker=True` chỉ nhận tool trong READONLY_TOOLS. Cả `worker=False` cũng bị chặn `issue_refund` và `append_audit`: tool ghi là host-only. Runtime kiểm tên tool; code của nhóm vẫn cần giữ nguồn args đáng tin cậy, nhất là principal và order.

MockLLM `complete()` hiện trả chuỗi theo purpose, không trả action JSON hay tự lập kế hoạch tool. Nhóm phải viết callback/adapter chọn action rõ ràng; không giả định runtime tự parse chuỗi model. Model có thể hỗ trợ chọn tool đọc trong giới hạn, nhưng số tiền phải lấy từ compute_compensation.

Sau vòng, shape kiểm stopped/final và quote, gọi host nếu quote dương. Host quyết approval, cap và idempotency: outcome ok → resolved; không ok có conflict → failed; không ok còn lại → denied. Giữ proposal_amount bằng quote đầy đủ. Truyền crash_at; nếu bắt InjectedCrash để flush trace thì phải raise lại như swarm. Các nhánh kết thúc bình thường dùng finish để ghi trace và đọc ledger thật.

## Budget: tính một lần ở đúng nơi

Budget là bộ đếm, **không tự giới hạn** số call, token hay tiền. Giới hạn lượt do max_steps của run_react thực hiện.

| Thao tác | Tự cập nhật | Quy ước |
| --- | --- | --- |
| dispatch tool hợp lệ | add_tool: tool_calls +1, cost tăng tool_cost(), steps +1 | Không cộng lại cho cùng dispatch |
| Gọi trực tiếp adapter trong tools.py | Không tự cập nhật Budget | Cộng budget.add_tool(tool_cost()) một lần cho mỗi lần gọi trực tiếp |
| complete(budget, ...) | add_llm: model_calls +1, cộng token/cost, steps +1 | Không cộng model lần nữa |
| decide trả final | Không tự cộng Budget | Nếu gọi model trong callback, ghi nhận qua complete |
| finish | Chép Budget sang RunResult, làm tròn cost 6 chữ số | Dùng cùng Budget suốt lần chạy |

Tool ghi bị chặn trước khi tính cost; tool lạ có thể bị cộng cost rồi mới ném ToolRejected. Không dùng exception để thử tên tool.

Giá hiện tại trong config/pricing.yaml: tool **10 VND/lần**, input **20 VND/1.000 token**, output **80 VND/1.000 token**. Đây là giá mô phỏng. complete ước lượng token theo độ dài chuỗi. Các lời gọi nội bộ trong một adapter, ví dụ compute_compensation gọi lookup_policy, không được dispatch tính thêm thành tool call riêng.

Phân biệt: lượt ReAct = số lần gọi decide; Budget.steps tăng theo model/tool call; metric steps của benchmark = **số event trong trace**. Ba số không nhất thiết bằng nhau. Shape/callback cần ghi trace an toàn vì run_react không tự tạo event.

## Swarm tham chiếu thực sự chọn tool thế nào?

Swarm **không có decide và không gọi run_react**. `_run()` chạy luồng cố định:

1. SELECT claim bằng các cột được chọn, cộng một tool call; claim không có thì dừng.
2. Gọi model purpose peer, dùng parse_delay_days đọc số ngày ticket, gọi model purpose parse. Hai chuỗi model trả về không được dùng làm action tool.
3. Thiếu order_id thì draft và trả needs_info.
4. Tạo handoff với allowed_tools theo thứ tự: lookup_order, lookup_policy, compute_compensation.
5. run_readonly_investigators đọc claim thêm một lần (cộng tool cost), rồi chạy tool theo thứ tự danh sách; chỉ compute khi đã có order.
6. Order không visible thì draft và escalated. Nếu visible, lấy quote, draft, rồi code gọi host khi quote dương và finish.

Investigator runner không có model chọn tool. Thứ tự handoff có ý nghĩa: compute_compensation dùng lookup_order đã có trong results. get_clause bị bỏ qua trong runner này vì chưa có clause_id; muốn dùng cần đường gọi với args thích hợp.

Nhánh C-001, seed 11, DB mới: **3 model calls, 5 tool calls**, proposal 35.000 VND. Không cộng chi phí các adapter nội bộ lần nữa.

**Điểm yếu cố ý:** seed chia hết cho 5 và số ngày ticket khác order → swarm sửa delay_days theo ticket rồi compute lại, cộng thêm một tool call. T02 seed 55 vì vậy quyết định sai. Shape nhóm viết giữ dữ kiện order, không sao chép REFERENCE_WEAKNESS.

Khi rà code chung: đúng args/principal; không lặp tool vô ích; còn lượt cho final; quote do code; host ghi tiền; Budget không đếm trùng; trace chỉ có dữ liệu đã lọc. `safe_dump()` chỉ sao chép JSON, không lọc secret.
