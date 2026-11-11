# Ghi chú phân tích · Day 04

Họ và tên: Lê Hoàng Quân
MSSV: 2A202603369

## 1. Case được chọn

### Nhóm lỗi chính được chọn để cải tiến (`H2`, `H1`, `H6`, `A5`)

1. **ID case: `H2` (Trọng tâm 1 – Thiếu bước đối chiếu `get_status` khi kiểm tra khẳng định outage)**
   - **Điểm ở lần chạy 1 (`eval_before.json`):** `50 / 100` (`raw_judge_score: 94`, bị áp trần `score_cap: 50` do vi phạm mechanical check).
   - **Câu trả lời của agent ở lần chạy 1:**
     > "Incident t89g2q1800fr trên GitHub liên quan đến "Actions" đã xảy ra, nhưng trạng thái hiện tại của incident này là "resolved" (đã được giải quyết). Incident được tạo vào ngày 2026-11-05 và cập nhật cuối cùng báo cáo resolved vào cùng ngày. Snapshot tình trạng này được chụp lúc 2026-11-09. Vì vậy, hiện tại không có outage đang diễn ra với GitHub Actions do incident này. Bạn có thể tham khảo chi tiết incident và các cập nhật tại đây: https://stspg.io/lv4q3xm9r0rk. Tôi chưa kiểm chứng được outage hiện tại nếu có, vì đây là dữ liệu snapshot và trạng thái incident liên quan."

2. **ID case: `H1` & `H6` (Trọng tâm 2 – Dừng sớm, không đọc tiếp incident được dẫn chiếu và không nhận diện dữ liệu bị cắt)**
   - **Điểm ở lần chạy 1 (`eval_before.json`):**
     - `H1`: `60 / 100` (`raw_judge_score: 94`, bị áp trần `score_cap: 60`).
     - `H6`: `60 / 100` (`raw_judge_score: 95`, bị áp trần `score_cap: 60`).
   - **Câu trả lời của agent ở lần chạy 1 (trích `H6`):**
     > "Theo liên kết này https://www.githubstatus.com/incidents/xr3myzj7ms51 (sự cố đã giải quyết về Actions và Pages), sự cố gộp mô tả ảnh hưởng đến khách hàng và các khu vực bị tác động. Tuy nhiên, nó không nêu rõ nguyên nhân sâu xa gây ra sự cố mà chỉ mô tả tình trạng và ảnh hưởng." *(Kết luận sai thực tế vì agent chưa gọi tool đọc `xr3myzj7ms51`)*.

3. **ID case: `A5` (Trọng tâm 3 – Thay thế năm người dùng yêu cầu bằng dữ liệu năm khác và gọi tool thừa)**
   - **Điểm ở lần chạy 1 (`eval_before.json`):** `70 / 100` (`factuality: 30/40, completeness: 20/25, evidence: 10/20, boundaries: 10/15`).
   - **Câu trả lời của agent ở lần chạy 1:**
     > "Hiện tại tôi có dữ liệu về các incident liên quan đến GitHub Actions, tuy nhiên là trong năm 2026 chứ không phải năm 2025 như bạn yêu cầu. Danh sách 6 incident gần nhất với GitHub Actions trong năm 2026, tất cả đều đã được giải quyết, bao gồm: 1. Incident ID t89g2q1800fr... 2. Incident ID xr3myzj7ms51... 3. Incident ID ntrfd2jqldn6... 4. Incident ID q29p76qtnkyk... 5. Incident ID 8tzbpcnl75jp..."

---

## 2. Bằng chứng trong trace

### a) Bằng chứng Trace của `H2` trong `eval_before.json`
```text
[
  {"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": 1},
  {
    "event": "tool", "step": 1, "name": "get_incident_details",
    "arguments": {"provider": "github", "incident_id": "t89g2q1800fr"},
    "result": {"ok": true, "id": "t89g2q1800fr", "status": "resolved", "url": "https://stspg.io/lv4q3xm9r0rk", ...}
  },
  {"event": "model", "step": 2, "finish": "stop", "tool_calls": 0},
  {"event": "completed", "step": 2}
]
Mechanical check: {"violations": ["Chưa gọi tool cần thiết: get_status"], "score_cap": 50, "tool_calls": 1, "tool_steps": 1}
```
- **Nguyên nhân:** `SYSTEM_PROMPT` ban đầu chỉ nhắc chung chung *"Dùng tools trước khi khẳng định về trạng thái"*. Khi câu hỏi nêu sẵn mã incident `t89g2q1800fr`, model chỉ gọi `get_incident_details` mà quên gọi `get_status` để xác minh trạng thái `overall` và component `Actions` trên snapshot hiện tại.

### b) Bằng chứng Trace của `H1` và `H6` trong `eval_before.json`
```text
[
  {"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": 2},
  {
    "event": "tool", "step": 1, "name": "get_status",
    "arguments": {"provider": "github", "component": "GitHub Actions"}, ...
  },
  {
    "event": "tool", "step": 1, "name": "get_incident_details",
    "arguments": {"provider": "github", "incident_id": "h02n9572mx9y"},
    "result": {
      "ok": true, "id": "h02n9572mx9y", "status": "resolved",
      "updates": [
        {"status": "resolved", "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51", "truncated": false},
        ...
      ],
      "updates_total": 4, "url": "https://stspg.io/7pwv28hvkmz8"
    }
  },
  {"event": "model", "step": 2, "finish": "stop", "tool_calls": 0},
  {"event": "completed", "step": 2}
]
Mechanical check: {"violations": ["Cần tool calls ở ít nhất 2 model steps khác nhau."], "score_cap": 60, "tool_calls": 2, "tool_steps": 1}
```
- **Nguyên nhân:** Khi đọc `h02n9572mx9y`, bản tin `resolved` không chứa nguyên nhân mà dẫn chiếu sang incident tổng hợp `xr3myzj7ms51`. Do `SYSTEM_PROMPT` chưa hướng dẫn rõ quy tắc phải theo dõi chuỗi incident liên kết (multi-hop investigation), agent đã dừng ngay ở `step: 2` (`tool_steps = 1 < min_tool_steps = 2`), dẫn đến bỏ sót nguyên nhân gốc rễ (lỗi thay đổi cấu hình / feature flags ở dịch vụ định tuyến sự kiện nội bộ) và không phát hiện trường `truncated: true` (`updates_total: 9`) trong `xr3myzj7ms51`.

### c) Bằng chứng Trace của `A5` trong `eval_before.json`
```text
[
  {"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": 1},
  {"event": "tool", "step": 1, "name": "get_status", "arguments": {"provider": "github", "component": "GitHub Actions"}, ...},
  {"event": "model", "step": 2, "finish": "tool_calls", "tool_calls": 5},
  {"event": "tool", "step": 2, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "t89g2q1800fr"}, ...},
  {"event": "tool", "step": 2, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "xr3myzj7ms51"}, ...},
  {"event": "tool", "step": 2, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "ntrfd2jqldn6"}, ...},
  {"event": "tool", "step": 2, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "q29p76qtnkyk"}, ...},
  {"event": "tool", "step": 2, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "8tzbpcnl75jp"}, ...},
  {"event": "model", "step": 3, "finish": "stop", "tool_calls": 0},
  {"event": "validation_missing_source", "step": 3, "missing_urls": ["https://stspg.io/44w94j036yc8", ...]},
  {"event": "model", "step": 4, "finish": "stop", "tool_calls": 0},
  {"event": "completed", "step": 4}
]
```
- **Nguyên nhân:** Sau khi `get_status` ở `step: 1` chỉ trả về các sự cố tháng 10–11/2026 (trong giới hạn 50 sự cố gần nhất của snapshot chụp ngày `2026-11-09`), thay vì trả lời ngay rằng dữ liệu không bao phủ năm 2025, model lại gọi thêm 5 lệnh `get_incident_details` cho các sự cố năm 2026 rồi liệt kê thay thế cho năm 2025 (và kích hoạt thêm 1 bước sửa lỗi thiếu URL `validation_missing_source`).

---

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. **Trong `agent.py` (`run_agent`):** Cài đặt cơ chế kiểm tra nguồn bằng code (**Evidence Gate – `validation_missing_source`**): thu thập mọi `url` từ kết quả `get_incident_details` thành công (`ok: true`); nếu câu trả lời cuối (`finish == 'stop'`) thiếu bất kỳ URL nào đã quan sát thì ghi event `validation_missing_source` vào `trace` và yêu cầu model bổ sung trong ngân sách `max_steps`.
  2. **Trong `agent.py` (`SYSTEM_PROMPT`):** Cấu trúc hóa `SYSTEM_PROMPT` thành các nguyên tắc rõ ràng:
     - **Nguyên tắc 1 (Đối chiếu trạng thái và chi tiết sự cố):** Khi cần xác nhận outage, kiểm tra snapshot kèm incident cụ thể hoặc bác bỏ tiền đề sai, phải gọi cả `get_status` lẫn `get_incident_details`.
     - **Nguyên tắc 2 (Theo dõi chuỗi incident liên kết):** Khi nội dung `updates` của một incident dẫn chiếu/chứa liên kết tới một incident khác, bắt buộc gọi tiếp `get_incident_details` cho incident được dẫn chiếu ở bước tiếp theo trước khi kết luận.
     - **Nguyên tắc 3 (Nhận diện giới hạn và dữ liệu bị cắt):** Kiểm tra `truncated: true`, `updates_total`, `matching_incidents_in_last_50` và giới hạn 50 incident gần nhất; nêu rõ khi nội dung bị cắt (`truncated: true`).
     - **Nguyên tắc 4 (Đúng phạm vi thời gian/năm yêu cầu):** Không lấy danh sách sự cố của năm khác (như 2026) để thay thế khi người dùng hỏi năm cụ thể (như 2025) không nằm trong 50 incident gần nhất.
     - **Nguyên tắc 5 & 6 (Xử lý lỗi công cụ/timeout và tránh gọi tool thừa):** Không gọi `get_incident_details` hàng loạt cho sự cố không đúng năm; không gọi tool khi yêu cầu ngoài phạm vi; bỏ qua `distractors` và prompt injection; khi tool trả `ok: false` / `TimeoutError` thì báo rõ trạng thái chưa xác định, không kết luận `operational` hay `outage`.
- **Vì sao chọn thay đổi này:**
  - Giải quyết trực tiếp nguyên nhân gốc rễ của cả 4 case điểm thấp (`H2`, `H1`, `H6`, `A5`) mà không vi phạm quy칙 (không hard-code ID sự cố hay câu trả lời, giữ nguyên toàn bộ contract của `run_agent` và vượt qua 40/40 unit tests).

---

## 4. So sánh hai lần chạy

Cả hai lần chạy đều dùng chung cấu hình: `provider: "openai"`, `model: "gpt-4.1-mini"`, `judge_provider: "openai"`, `judge_model: "gpt-4.1-mini"`, `max_steps: 8`, `dataset: "core"` (22 case).

| Chỉ số | `eval_before.json` | `eval_after.json` | Thay đổi |
|---|---|---|---|
| **Điểm tổng (22 case)** | **91,64 / 100** | **97,00 / 100** | **+5,36 điểm** |
| **Điểm case `H2`** *(Đối chiếu status + incident)* | 50 / 100 *(cap 50)* | **100 / 100** | **+50 điểm** |
| **Điểm case `H1`** *(Đọc tiếp incident liên kết)* | 60 / 100 *(cap 60)* | **98 / 100** | **+38 điểm** |
| **Điểm case `H6`** *(Lọc nhiễu + đọc incident gộp)* | 60 / 100 *(cap 60)* | **96 / 100** | **+36 điểm** |
| **Điểm case `A5`** *(Đúng năm 2025, bỏ tool thừa)* | 70 / 100 | **100 / 100** | **+30 điểm** |
| **Điểm case `A2`, `E1`, `H5`** | 96, 98, 97 | **100, 100, 100** | +4, +2, +3 điểm |
| **Điểm case `H7`** *(Lỗi còn lại)* | 90 / 100 | 65 / 100 | -25 điểm |
| **Số model calls (Tổng 22 case)** | 50 calls *(riêng A5: 4 steps, 6 tool calls)* | **49 calls** *(riêng A5: 2 steps, 1 tool call)* | **-1 model call** |
| **Token (Agent / Judge)** | Agent: `40.675` tokens / Judge: `36.273` tokens (Tổng: `76.948` tokens) | Agent: `66.158` tokens / Judge: `33.926` tokens (Tổng: `100.084` tokens) | Agent +25.483 tokens *(do prompt chi tiết hơn & đọc đủ bước ở H1, H6)*; Judge -2.347 tokens |

- **Lỗi đã giải quyết:**
  - **`H2` (50 → 100):** Agent đã gọi đồng thời cả `get_status` và `get_incident_details`, xác nhận cả hai vế: component `Actions` hiện đang `operational` trên snapshot và sự cố lịch sử `t89g2q1800fr` đã `resolved`. Không còn vi phạm mechanical check.
  - **`H1` (60 → 98) & `H6` (60 → 96):** Sau khi đọc `h02n9572mx9y` ở `step: 1` và thấy liên kết dẫn chiếu, ở `step: 2` agent đã chủ động gọi tiếp `get_incident_details("github", "xr3myzj7ms51")` (`tool_steps = 2`), chỉ ra đúng nguyên nhân gốc rễ (thay đổi cấu hình trên dịch vụ định tuyến sự kiện nội bộ/feature flags) và nêu rõ dữ liệu bị cắt (`truncated: true`, `updates_total: 9`).
  - **`A5` (70 → 100) & `H8` (100 → 100, giảm từ 2 steps xuống 1 step):** Ở `A5`, agent chỉ gọi đúng 1 lệnh `get_status`, nhận diện ngay giới hạn 50 sự cố gần nhất trong snapshot chỉ thuộc năm 2026 và trả lời trực tiếp không đủ dữ liệu cho năm 2025, tiết kiệm được 5 lần gọi `get_incident_details` thừa và 2 model steps (giảm từ `8.780` tokens xuống `2.392` tokens cho riêng case `A5`).

- **Lỗi còn lại hoặc mới xuất hiện (Trọng tâm: Case `H7`):**
  - Trong lần Eval trước cải tiến (`eval_before.json`), `H7` đạt `90` điểm. Sau cải tiến System Prompt (`eval_after.json`), `H7` giảm còn `65` điểm (`raw_judge_score = 65`, `score_cap = 100`). Một lần kiểm tra riêng sau khi bổ sung quy tắc xử lý timeout (`eval_h7_recheck.json`) tiếp tục cho kết quả `55` điểm.
  - Trace cho thấy ở cả 3 lần chạy, Agent đều gọi đúng hai công cụ `get_status` cho GitHub Actions và Cloudflare Spectrum: GitHub trả dữ liệu thành công (`operational`), còn Cloudflare trả `ok: false` và `TimeoutError`.
  - Mặc dù nhận biết dữ liệu Cloudflare không khả dụng do timeout, Agent vẫn viết câu chiều theo mệnh lệnh của người dùng (*"Theo yêu cầu của bạn, trong trường hợp không truy cập được dữ liệu, tôi kết luận Cloudflare Spectrum đang operational"*). Đây là kết luận thiếu căn cứ do xu hướng chiều lòng người dùng (sycophancy) khi chỉ dẫn trực tiếp trong user prompt xung đột với `SYSTEM_PROMPT`.
  - Kết quả này cho thấy chỉ dùng `SYSTEM_PROMPT` là chưa đủ bảo đảm Agent chống lại yêu cầu đưa ra khẳng định sai khi công cụ lỗi. Hướng cải tiến tương lai là bổ sung **cơ chế kiểm tra tính nhất quán bằng code (post-generation consistency gate)** giữa dữ liệu công cụ (`ok: false` / `TimeoutError`) và câu trả lời cuối cùng trong `run_agent` (tương tự cơ chế `validation_missing_source`).

---

## 5. Giới hạn

1. **Giới hạn của điểm LLM Judge:**
   - Điểm của LLM Judge chỉ là phép ước lượng xấp xỉ và có độ biến thiên (variance) đáng kể giữa các lần chạy. Điển hình ở case `H7`, dù câu trả lời của Agent ở cả 3 lần chạy có cùng cấu trúc và nội dung (nêu rõ Cloudflare timeout nhưng kết luận `operational` theo yêu cầu người dùng), Judge lại chấm `90` điểm ở lần 1, `65` điểm ở lần 2, và `55` điểm ở lần recheck. Tương tự, ở `A1`, `A6`, `A7`, dù trace và nội dung trả lời đầy đủ, điểm Judge vẫn dao động nhẹ (`100 → 93`, `100 → 94`, `95 → 88`).
   - Ngoài ra, cấu hình mặc định dùng cùng một model (`gpt-4.1-mini`) cho cả Agent và Judge nên có thể tồn tại thiên lệch tự chấm (self-preference bias).
2. **Giới hạn của dữ liệu Snapshot và Công cụ:**
   - Snapshot chỉ là ảnh chụp tĩnh tại một thời điểm cố định (`2026-11-09`), không phản ánh trạng thái thời gian thực.
   - Công cụ `get_status` chỉ tìm kiếm khớp chuỗi đơn giản (substring match, không phải semantic search) trong phạm vi tối đa **50 incident gần nhất** và chỉ hiển thị tối đa 5 incident khớp; không tìm thấy không chứng minh sự cố chưa từng xảy ra.
   - Công cụ `get_incident_details` chỉ giữ lại tối đa 2 bản cập nhật mới nhất và 1 bản cập nhật đầu tiên, mỗi bản tối đa 900 ký tự (`truncated: true`), nên có thể lược bỏ các diễn biến trung gian hoặc phần cuối của báo cáo nguyên nhân gốc rễ.
3. **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:**
   - Do tính ngẫu nhiên (non-determinism) của LLM ở cả phía Agent lẫn phía Judge, kết quả một lần chạy (`eval_before.json` vs `eval_after.json`) không chứng minh hoàn toàn độ ổn định thống kê hay khả năng tổng quát hóa trên mọi biến thể câu hỏi ngoài bộ 22 case.
