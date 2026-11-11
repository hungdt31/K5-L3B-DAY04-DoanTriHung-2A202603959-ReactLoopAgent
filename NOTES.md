# Ghi chú phân tích · Day 04

Họ và tên: Đoàn Trí Hùng
MSSV: 2A202603959

Cấu hình cả hai lần eval: `provider=anthropic`, `model=claude-haiku-5-5` cho agent và judge, `max_steps=8`, snapshot `core`, chạy ngày 11/11/2026.

## 0. Sửa cấu hình trước khi chốt `eval_before.json`

Lần chạy eval đầu tiên (chưa lưu làm kết quả) có 17/22 case `unscored`, điểm tổng `null`:

- 12 case agent trả `status: error` với `finish='length'`: `usage.output_tokens` của bước cuối đúng bằng 1024, là giới hạn `max_tokens` trong adapter Anthropic (`providers.py`). Tiếng Việt có dấu tốn khoảng 0,8 token/ký tự, nên câu trả lời 1.200 ký tự đã chạm trần.
- 5 case `Judge lỗi: Judge không trả JSON hoàn chỉnh`: chạy lại judge thủ công trên case E5 cho `finish='length'`, `output_tokens=1024`. Judge Haiku viết trường `reason` quá dài và bị cắt trước khi đóng JSON.

Hai thay đổi cấu hình, áp dụng giống nhau cho cả hai lần eval:

1. `providers.py`: nâng `max_tokens` của adapter Anthropic từ 1024 lên 4096. File này không nằm trong danh sách cấm sửa của `RULES.md`; thay đổi chỉ cho model đủ chỗ hoàn thành output, không đổi cách chấm.
2. `agent.py`: thêm `OUTPUT_RULES` vào system prompt, yêu cầu câu trả lời dưới 550 ký tự, văn bản thuần, gồm kết luận theo snapshot, bằng chứng kèm URL và giới hạn dữ liệu.

Sau hai thay đổi, lần chạy baseline chấm được 22/22 case.

Ghi chú trung thực về baseline: tiến trình eval baseline được khởi động ở nền, và trong lúc nó còn chạy `source .env`, file `agent.py` đã được ghi sang phiên bản có kiểm tra nguồn sơ khởi (quy tắc "ít nhất một URL", so khớp nguyên văn). Trace trong `eval_before.json` cho thấy phiên bản này đã được dùng (event `final` có key `sources_cited`), nhưng **0/22 case** phát sinh event `validation_missing_source`, nên không có lần sửa nào xảy ra và câu trả lời tương đương với loop chưa có kiểm tra nguồn. Baseline vì vậy vẫn phản ánh agent trước cải tiến thực sự ở mục 3.

## 1. Case được chọn

- ID case: **H1** — "Điều tra incident GitHub h02n9572mx9y. Theo liên kết do incident cung cấp để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare yaa0witbl2me ... Dẫn nguồn cho từng bên."
- Câu trả lời của agent (lần 1, nguyên văn):

  > Tại snapshot 03:50 UTC ngày 09/11/2026, cả hai incident đều đã resolved. Không có bằng chứng chung nguyên nhân.
  >
  > GitHub h02n9572mx9y (Actions, mở 04/11 17:40 UTC) được gộp vào xr3myzj7ms51 (Actions, Pages, Webhooks; 15:49–17:55 UTC). Nguyên nhân nêu trong đó là thay đổi cấu hình dịch vụ định tuyến sự kiện nội bộ, liên quan feature flag. Nguồn: stspg.io/pjgptg3cfc6n
  >
  > Cloudflare yaa0witbl2me (Workers Builds; 05/11 19:51–20:20 UTC): tồn đọng hàng đợi build, không nêu nguyên nhân. Nguồn: cloudflarestatus.com/incidents/yaa0witbl2me
  >
  > Giới hạn: tóm tắt GitHub bị cắt ngắn, và Cloudflare không công bố nguyên nhân.

- Điểm ở lần chạy 1: **86/100** (factuality 34, completeness 22, evidence 17, boundaries 13). `grade.improvement` của judge: "... dẫn riêng URL của h02n9572mx9y cho phần trạng thái ban đầu ...".

Cùng lỗi xuất hiện ở **H6** (89): đọc 2 incident, chỉ dẫn URL của xr3myzj7ms51, thiếu URL của h02n9572mx9y.

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json` (rút gọn phần `result`, giữ nguyên các key):

```text
{"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": 2}
{"event": "tool",  "step": 1, "name": "get_incident_details",
   "arguments": {"provider": "github", "incident_id": "h02n9572mx9y"},
   "result": {"ok": true, "id": "h02n9572mx9y", "url": "https://stspg.io/7pwv28hvkmz8", ...}}
{"event": "tool",  "step": 1, "name": "get_incident_details",
   "arguments": {"provider": "cloudflare", "incident_id": "yaa0witbl2me"},
   "result": {"ok": true, "id": "yaa0witbl2me", "url": "https://www.cloudflarestatus.com/incidents/yaa0witbl2me", ...}}
{"event": "model", "step": 2, "finish": "tool_calls", "tool_calls": 1}
{"event": "tool",  "step": 2, "name": "get_incident_details",
   "arguments": {"provider": "github", "incident_id": "xr3myzj7ms51"},
   "result": {"ok": true, "id": "xr3myzj7ms51", "url": "https://stspg.io/pjgptg3cfc6n", ...}}
{"event": "model", "step": 3, "finish": "stop", "tool_calls": 0}
{"event": "final", "step": 3, "answer_chars": 612, "sources_cited": 3}
```

Đối chiếu: agent đọc thành công **3** incident với 3 URL khác nhau, nhưng câu trả lời chỉ chứa 2 URL (`stspg.io/pjgptg3cfc6n`, `cloudflarestatus.com/incidents/yaa0witbl2me`). URL `https://stspg.io/7pwv28hvkmz8` của h02n9572mx9y, incident mà câu hỏi yêu cầu điều tra đầu tiên, không xuất hiện. Kiểm tra bằng code trên câu trả lời cũ:

```text
H1 missing: ['h02n9572mx9y']
H6 missing: ['h02n9572mx9y']
A1 missing: []      A6 missing: []
```

Nguyên nhân có thể xảy ra:

- **Lỗi ở loop, không phải ở dữ liệu**: tool đã trả đủ URL (`result.url`), model có bằng chứng trong context nhưng khi tóm tắt đã gộp h02n9572mx9y vào incident gộp và chỉ dẫn một nguồn. Prompt yêu cầu "dẫn nguồn" nhưng không có cơ chế kiểm tra, nên lỗi lọt qua.
- Yêu cầu trả lời ngắn (dưới 550 ký tự) khiến model có xu hướng bỏ URL để tiết kiệm chỗ.
- Phiên bản kiểm tra nguồn sơ khởi có hai điểm yếu: chỉ cần **một** URL bất kỳ là qua, và so khớp nguyên văn `https://...` trong khi model viết `stspg.io/...` không có scheme (false negative trên chính URL mà H1 đã dẫn).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  - `_observed_source`: sau mỗi `get_incident_details` **thành công** (`ok: true` và có `url`), lưu `{url, id}` vào danh sách `sources`. Tool lỗi không được tính, nên case H3 (ID không tồn tại) và E6/A3 (timeout) không bị bắt URL.
  - `_missing_sources`: so khớp **từng** incident đã đọc với câu trả lời; chấp nhận URL có hoặc không có `https://`, hoặc dạng `.../incidents/<id>` của cùng incident. Trả về danh sách incident còn thiếu.
  - Bước 5 của `run_agent`: khi `finish == 'stop'` và còn incident thiếu URL, ghi event `{'event': 'validation_missing_source', 'step', 'missing_incidents', 'expected_urls'}`; nếu `step < max_steps` thì gửi `REVISION_REQUEST` liệt kê đúng các URL còn thiếu và gọi model tiếp; nếu hết lượt thì trả `status: 'max_steps'`, không báo `completed`.
  - Thêm `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)` để cùng loop chạy được với bộ tools bonus.
- Vì sao chọn thay đổi này:
  - Lỗi thuộc nhóm "nguồn và scope" trong GUIDE (A2/A8/H1): judge trừ điểm evidence ở H1, H6, A6 vì thiếu URL của incident đã đọc. Đây là lỗi có thể **kiểm tra bằng code** vì URL nằm sẵn trong tool result, nên không cần tin vào model.
  - Kiểm tra là *sự hiện diện* của URL, tốn tối đa một lượt gọi model bổ sung, bounded bởi `max_steps`.
  - Hoàn thành contract trong `tests/test_validation.py`: `python3 -m unittest discover -s tests -v` đạt 37/37; `bonus/test_agent.py` 3/3.

Giới hạn của thay đổi: chỉ chứng minh URL có trong câu trả lời, không chứng minh mọi claim đều có nguồn; cũng không kiểm tra URL của `get_status` (trang tổng quan).

## 4. So sánh hai lần chạy

Khối `config` của hai file giống nhau (kiểm tra bằng code: `before["config"] == after["config"]` là `True`).

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 91,05 (22/22 case) | 91,50 (22/22 case) |
| Điểm case đã chọn (H1) | 86 (evidence 17) | 82 (evidence 17) |
| Điểm H6 (cùng lỗi) | 89 | 93 |
| Điểm A6 (cùng lỗi, phát hiện ở lần 2) | 87 | 87 |
| Số model calls (agent, cả bộ) | 59 | 62 (+3, đúng bằng số lần sửa) |
| Token agent in/out | 104.350 / 29.585 | 116.902 / 30.131 (+12% in, +2% out) |
| Token judge in/out | 59.077 / 23.617 | 60.134 / 23.629 |
| Event `validation_missing_source` | 0 | 3 (A6, H1, H6; đều thiếu `h02n9572mx9y`) |
| Chi phí | Provider không trả số tiền trong usage; chỉ so token. | Tăng khoảng 12.500 token input cho 3 lượt sửa. |

Điểm từng case (before → after): E1 90→95, E2 95→92, E3 91→93, E4 94→94, E5 95→90, E6 97→96, A1 86→88, A2 93→89, A3 88→89, A4 97→99, A5 85→89, A6 87→87, A7 90→88, A8 93→94, H1 86→82, H2 88→94, H3 88→91, H4 100→98, H5 91→87, H6 89→93, H7 95→93, H8 85→92.

- Lỗi đã giải quyết:
  - **Lỗi mục tiêu đã hết về mặt cơ học**: cả 3 lần agent định trả lời thiếu URL của `h02n9572mx9y`, loop đều phát hiện (`validation_missing_source` ở step 3), gửi yêu cầu sửa, và model bổ sung URL ngay lượt kế tiếp; cả 3 case kết thúc `completed` với `sources_cited` bằng số incident đã đọc. Trace H1 lần 2:
    ```text
    {"event": "model", "step": 3, "finish": "stop", "tool_calls": 0}
    {"event": "validation_missing_source", "step": 3, "missing_incidents": ["h02n9572mx9y"], "expected_urls": ["https://stspg.io/7pwv28hvkmz8"]}
    {"event": "model", "step": 4, "finish": "stop", "tool_calls": 0}
    {"event": "final", "step": 4, "answer_chars": 722, "sources_cited": 3}
    ```
    Câu trả lời H1 lần 2 có đủ 3 URL: `stspg.io/7pwv28hvkmz8`, `githubstatus.com/incidents/xr3myzj7ms51`, `cloudflarestatus.com/incidents/yaa0witbl2me`.
  - H6 tăng 89 → 93; judge không còn nhắc thiếu nguồn.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H1 giảm 86 → 82 dù đã đủ URL.** Điểm evidence giữ 17; điểm giảm ở factuality (34→33) và completeness (22→19) vì judge bắt lỗi khác: câu trả lời viết "hai đợt 15:49–17:55 UTC", gộp mất khoảng nghỉ 16:48–17:20 UTC. Lỗi này là lỗi tóm tắt của model, nằm ngoài phạm vi kiểm tra URL. Ở lần 1 câu trả lời cũng viết "15:49–17:55 UTC" nhưng judge không trừ, cho thấy judge không ổn định.
  - Lượt sửa thêm 1 model call và khiến câu trả lời dài hơn (H1: 612 → 722 ký tự), đi ngược yêu cầu ngắn gọn trong `OUTPUT_RULES`.
  - Chênh lệch tổng +0,45 điểm nằm trong biên độ nhiễu: 11 case tăng, 9 case giảm, 2 case giữ nguyên dù agent không đổi hành vi ở 19/22 case đó. Không thể kết luận cải tiến làm tăng điểm tổng; chỉ có thể kết luận nó loại bỏ được một lớp lỗi cụ thể, kiểm chứng được bằng trace.
  - Kiểm tra nguồn chưa áp dụng cho `get_status` (A3 lần 1 chỉ dẫn `githubstatus.com` trang chủ), và chưa phát hiện trường hợp model dán URL đúng cạnh nhận định sai.

## 5. Giới hạn

- **Điểm judge là ước lượng.** Judge dùng cùng model Haiku với agent nên có thể thiên lệch khi tự chấm; `reason` của judge khá dễ dãi (nhiều case 93–97 dù câu trả lời còn thiếu chi tiết). Judge cũng nhạy với độ dài: lúc `max_tokens=1024` judge không trả được JSON, chứng tỏ kết quả chấm phụ thuộc cả vào cấu hình hạ tầng.
- **Snapshot cố định ngày 09/11/2026.** Mọi kết luận "operational" hay "resolved" chỉ đúng tại `captured_at`; tool chỉ tìm chuỗi trong 50 incident gần nhất và trả tối đa 5, nên "không tìm thấy" không chứng minh "không có".
- **Một lần chạy mỗi phiên bản** không chứng minh độ ổn định: điểm từng case dao động vài điểm giữa hai lần chạy cùng một agent (A8 lần probe 97, lần baseline 93) do model không deterministic. Chênh lệch tổng nhỏ hơn khoảng 2 điểm không nên coi là cải tiến hay thoái lui.
- Kiểm tra nguồn chỉ xác nhận URL xuất hiện; model vẫn có thể gắn URL đúng vào nhận định sai.
