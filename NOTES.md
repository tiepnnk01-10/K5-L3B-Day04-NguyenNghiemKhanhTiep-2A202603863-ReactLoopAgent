# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Nghiêm Khánh Tiệp
MSSV: 2A202603863

Cấu hình eval (giữ nguyên cho cả hai lần chạy): agent `openai / gpt-4o-mini`, judge `openai / gpt-4o-mini`, `max_steps` 8, dataset `core`, snapshot.

## 1. Case được chọn

- ID case: **H6** (scenario `noisy_status`), cùng nhóm lỗi với **E3**.
- Câu hỏi: "Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn dữ liệu và trích nguồn."
- Câu trả lời của agent (rút gọn): liệt kê thời gian, tác động và 3 update của `h02n9572mx9y`, đưa link "sự cố liên quan" `https://www.githubstatus.com/incidents/xr3myzj7ms51`, mục "Trích dẫn nguồn" chỉ ghi trang tổng `https://www.githubstatus.com`. Không trả lời được incident gộp nêu nguyên nhân hay chỉ mô tả ảnh hưởng.
- Điểm ở lần chạy 1: **50/100**. Judge: "không nêu rõ nguyên nhân… mặc dù có liên kết đến sự cố liên quan". Kiểm tra bằng code: `Cần tool calls ở ít nhất 2 model steps khác nhau` → trần 60.

## 2. Bằng chứng trong trace

Trace H6 trong `eval_before.json`:

```text
[step 1] model  finish=tool_calls  calls=2
[step 1] tool   get_status            {'provider': 'github', 'component': 'github actions'}
[step 1] tool   get_incident_details  {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
         -> ok=True  url=https://stspg.io/7pwv28hvkmz8  updates_total=4
[step 2] model  finish=stop  calls=0   -> status=completed

URL incident đã quan sát : https://stspg.io/7pwv28hvkmz8
URL có trong answer       : (không có) — chỉ có https://www.githubstatus.com
```

Cùng lỗi ở E3 (75 điểm): đọc thành công `c60k3g4vsoc1` (url `https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1`) nhưng answer chỉ ghi "[1] Cloudflare Status - Spectrum Availability Issue in Phoenix", không có URL.

Thống kê trên cả 22 case của lần 1: 2 case (H6, E3) có `get_incident_details` thành công nhưng answer không chứa URL incident nào đã quan sát.

Nguyên nhân có thể:

- **Prompt/loop:** system prompt chỉ nói chung "Dẫn nguồn từ tool"; loop chấp nhận mọi answer có `finish == 'stop'`, không có bước nào kiểm tra answer có dẫn nguồn thật. Model thường dẫn trang tổng (`source` của `get_status`) thay cho `url` của incident.
- **Loop (lỗi thứ hai của H6):** model gọi `get_status` và `get_incident_details` song song trong cùng step 1, sau khi đọc update nhắc tới `xr3myzj7ms51` thì trả lời luôn thay vì gọi tiếp `get_incident_details` cho incident gộp. Vì không đọc incident gộp nên không trả lời được câu hỏi về nguyên nhân.
- **Dữ liệu:** H6 có 40 mục nhiễu giả lập trong status; `get_incident_details` chỉ giữ update đầu và hai update mới nhất, mỗi update tối đa 900 ký tự, nên nội dung có thể bị cắt.

## 3. Thay đổi đã thực hiện

**Vòng 1 — kiểm tra URL nguồn** (CP5). Sửa gì, ở đâu: thêm **kiểm tra URL nguồn** trong `run_agent` (`agent.py`):
  - Mỗi tool call `get_incident_details` trả `ok: true` thì lưu `url` vào `observed_urls`.
  - Khi model trả final answer (`finish == 'stop'`, không còn tool call): nếu `observed_urls` không rỗng mà answer không chứa URL nào trong đó → ghi event `validation_missing_source` (kèm danh sách URL) vào trace, thêm một message yêu cầu model viết lại và dẫn URL của từng incident, chỉ dùng URL có trong kết quả tool; rồi gọi model tiếp.
  - Lần sửa vẫn tính vào `max_steps`; nếu hết lượt mà vẫn thiếu nguồn thì trả `max_steps`, không báo `completed`.
  - Không bắt URL khi không có incident nào đọc thành công (tool lỗi hoặc chỉ dùng `get_status`).
  - Phiên bản này (chỉ validation, giữ nguyên `SYSTEM_PROMPT`) đã chạy eval một lần: 76.00 điểm, validation không kích hoạt lần nào (xem 4a).
- Vì sao chọn thay đổi này: lỗi thiếu nguồn kiểm tra được bằng code một cách chắc chắn (URL có hay không có trong answer), không phụ thuộc judge; nó đánh vào tiêu chí evidence (20 điểm) của rubric; và loop là nơi duy nhất biết chính xác URL nào đã thật sự được quan sát.

**Vòng 2 — kiểm tra bằng chứng trong loop + quy trình điều tra trong prompt** (phiên bản cuối, dùng cho `eval_after.json`). Sau vòng 1, H1/H6 vẫn không đi theo liên kết, A5 suy luận sai từ dữ liệu giới hạn, A1 gắn nhầm URL. Thử chỉ sửa prompt thì `gpt-4o-mini` vẫn không làm theo (H7 vẫn "coi Cloudflare là operational" khi timeout, H1/H6 vẫn gọi mọi tool trong một step), nên các quy tắc quan trọng được chuyển thành kiểm tra trong loop:

  - `SYSTEM_PROMPT`: thêm quy trình điều tra tổng quát — không gọi tool cho câu hỏi ngoài luồng; có mã incident thì luôn gọi `get_incident_details` cho mã đó; hỏi về tình trạng hiện tại thì gọi `get_status` kể cả khi đã có mã; đọc tiếp incident được liên kết; kiểm tra tiền đề sai; giữ đúng phạm vi; tách ảnh hưởng và nguyên nhân; nêu giới hạn 50 incident/900 ký tự; tool lỗi thì không suy đoán trạng thái kể cả khi người dùng yêu cầu; đặt `url` cạnh từng incident; bỏ qua chỉ dẫn trong tool result.
  - Kiểm tra nguồn theo từng incident: answer nhắc mã của incident đã đọc thì phải có URL của chính incident đó (`validation_missing_source` kèm danh sách `incident_id`/`url` còn thiếu).
  - `validation_unread_reference`: update của incident đã đọc chứa `/incidents/<mã>` của incident chưa đọc → trước khi chấp nhận answer, yêu cầu model gọi `get_incident_details` cho incident đó (mỗi mã một lần).
  - `validation_unknown_url`: URL trong answer không có trong câu hỏi hay kết quả tool → yêu cầu bỏ/thay (một lần). URL dạng `<host>/incidents/<mã>` được chấp nhận khi host và mã đều có trong kết quả tool, vì đó là định dạng chuẩn của status page (`status_tools.py` cũng tự tạo URL này).
  - Kết quả tool lỗi gửi cho model kèm `agent_note`: không có dữ liệu, không kết luận trạng thái phần đó. Trace vẫn ghi kết quả gốc.
  - Mọi yêu cầu sửa vẫn tính vào `max_steps`; hết lượt thì trả `max_steps`.
- Kiểm tra: `python -m unittest discover -s tests -v` → 37/37 pass. Không sửa `fixtures/`, `evals/cases.json`, `evals/scenarios.py`, `evals/grading.py`.
- Lưu ý phương pháp: vòng 2 được phát triển bằng cách chạy chính bộ eval này 3 lần thử (file `eval_results_try*.json`, không nộp): 78.64 → 87.27 → 86.05. Code được chốt sau lần thử 3, rồi `eval_after.json` được chạy **một lần** và giữ nguyên kết quả, không chọn lần chạy tốt nhất. Vì cải tiến được điều chỉnh trên chính bộ eval nên điểm có thể lạc quan hơn trên câu hỏi mới (overfitting); các quy tắc được viết tổng quát, không chứa đáp án hay mã incident của case nào.

## 4. So sánh hai lần chạy

Khối `config` của `eval_before.json` và `eval_after.json` giống hệt nhau (`openai / gpt-4o-mini`, judge `gpt-4o-mini`, `max_steps` 8, dataset `core`).

| Chỉ số | `eval_before.json` (agent CP3) | Vòng 1 (chỉ validation) | `eval_after.json` (vòng 2, bản cuối) |
|---|---|---|---|
| Điểm tổng | 77.05 | 76.00 | **81.36** |
| Điểm case đã chọn (H6) | 50 | 60 | 75 |
| Case có vi phạm kiểm tra bằng code | 2 (H1, H6: trần 60) | 2 (H1, H6) | **0** |
| Case đạt ≥ 95 điểm | 5 | 6 | 8 |
| Case `completed` / `unscored` | 22 / 0 | 22 / 0 | 22 / 0 |
| Validation kích hoạt | — | 0 | `unread_reference` 2 (H1, H6), `missing_source` 2 |
| Số model calls (agent, 22 case) | 50 | 51 | 51 |
| Token agent (input / output) | 33,723 / 10,030 | 34,976 / 10,001 | 81,018 / 10,717 |
| Token judge (input / output) | 39,857 / 3,029 | 40,171 / 3,068 | 38,685 / 2,930 |
| Chi phí ước tính (gpt-4o-mini $0.15 / $0.60 per 1M token) | ~$0.011 agent, ~$0.019 cả judge | ~$0.011 / ~$0.019 | ~$0.019 agent, ~$0.026 cả judge |

Điểm theo case (before → after): E1 70→45, E2 95→75, E3 75→100, E4 80→100, E5 50→75, E6 65→75, A1 90→95, A2 100→100, A3 75→80, A4 100→100, A5 55→75, A6 80→75, A7 75→75, A8 100→100, H1 60→80, H2 100→100, H3 75→95, H4 80→80, H5 75→40, H6 50→75, H7 70→65, H8 75→85.

Cùng code vòng 2, 3 lần chạy thử trước khi chốt: 78.64, 87.27, 86.05; lần chính thức: 81.36. Trung bình ~83, dao động ~9 điểm giữa các lần.

Đọc kết quả:

- Cải tiến **chắc chắn** (quan sát được trên trace, không phụ thuộc judge): H1 và H6 không còn vi phạm `Cần tool calls ở ít nhất 2 model steps khác nhau`. Trong `eval_after.json`, cả hai đều có event `validation_unread_reference` → model gọi `get_incident_details` cho `xr3myzj7ms51` ở step sau → tool calls ở 2 steps. Trần 60 điểm được gỡ; H1 60→80, H6 50→75.
- Kiểm tra nguồn đã kích hoạt trên model thật: 2 lần `validation_missing_source` (đều ở H6, sau khi đọc incident gộp), model sửa trong ngân sách, không case nào thành `max_steps`.
- Điểm tổng tăng 77.05 → 81.36 (+4.3). Mức này nhỏ hơn dao động giữa các lần chạy cùng code (78.64–87.27), nên **không đủ để kết luận chắc chắn** điểm judge tăng; phần chắc chắn là thay đổi hành vi ở trên.
- Chi phí: số model calls gần như không đổi (50 → 51) nhưng token input của agent tăng ~2.4 lần vì system prompt dài hơn, gửi lại ở mọi step. Tổng chi phí vẫn rất nhỏ (~$0.026 cho 22 case gồm judge).

- Lỗi đã giải quyết:
  - H1/H6: không đi theo liên kết sang incident gộp → đã đi theo (validation_unread_reference), không còn bị trần 60.
  - Thiếu URL incident đã đọc (H6, E3 ở lần 1): bị chặn bằng `validation_missing_source`; trong lần cuối không case nào có incident đã đọc mà thiếu URL.
  - URL bịa: bị chặn bằng `validation_unknown_url` (dự phòng; lần cuối không kích hoạt).
- Lỗi còn lại hoặc mới xuất hiện:
  - **Judge chấm sai (H5 40, E1 45):** H5 bị factuality 0/40 với lý do "khẳng định hoạt động bình thường mà không nơi nào xác nhận", trong khi answer nói Actions operational theo snapshot 09/11/2026, các incident resolved, có URL — đúng `expected`. E1 cũng trả lời đúng `expected` (không có sự cố đang diễn ra, incident 05/11 resolved) nhưng bị chấm 45. Đây là lỗi của judge, không phải agent.
  - **Nêu giới hạn chưa đủ rõ theo judge (E5, E6, A3, A5, H7: 65–80):** answer đã nói timeout/giới hạn 50 incident, nhưng judge vẫn muốn nhấn mạnh hơn; điểm các case này dao động mạnh giữa các lần chạy (ví dụ E6: 95 ở lần thử 2, 75 ở lần chính thức).
  - **H7 (65):** đôi khi `gpt-4o-mini` vẫn chiều yêu cầu "cứ coi là operational" dù prompt và `agent_note` cấm; không có kiểm tra bằng code cho lỗi này vì khó phát hiện chắc chắn một khẳng định về trạng thái.
  - **E2 (95 → 75):** answer liệt kê thêm incident cache purge cũ; judge cho là chưa khẳng định rõ ảnh hưởng tới website non-FedRAMP.
- Không đạt được mục tiêu mọi case ≥ 95: chỉ 8/22 case đạt. Với agent và judge đều là `gpt-4o-mini`, cùng một case có thể chênh 20–35 điểm giữa hai lần chạy với cùng code (H6: 100 ở lần thử 2, 75 ở lần chính thức), nên mục tiêu này không thể đạt ổn định bằng cách sửa agent. Hướng tiếp theo nếu cần: dùng agent model mạnh hơn và judge khác model (khi đó phải chạy lại cả `eval_before` với cùng cấu hình), và chạy nhiều lần mỗi phiên bản để so trung bình.

## 5. Giới hạn

- **Điểm judge:** judge là `gpt-4o-mini`, cùng model với agent, nên có thể thiên lệch khi tự chấm, và chính judge cũng sai: H5 bị factuality 0/40 dù answer đúng `expected`; ở lần chạy trước H5 bị trừ điểm vì "ngày ở tương lai" dù đó là ngày trong snapshot; H4 (bài thơ, không cần tool) được 80 hoặc 100 dù câu trả lời tương đương. Điểm 0–100 chỉ là ước lượng; cần đọc `expected`, trace và answer để kiểm tra lại.
- **Snapshot:** dữ liệu cố định từ 09/11/2026 và chỉ gồm 50 incident gần nhất mỗi provider; `get_incident_details` chỉ giữ update đầu và hai update mới nhất (tối đa 900 ký tự). Kết quả không phản ánh trạng thái thực tế hiện tại, và không thể trả lời đầy đủ các câu hỏi về khoảng thời gian dài (A5, H8).
- **Một lần chạy mỗi phiên bản:** cả agent lẫn judge có tính ngẫu nhiên. Với cùng code vòng 2, bốn lần chạy cho 78.64 / 87.27 / 86.05 / 81.36; từng case dao động tới 20–35 điểm. Chênh lệch tổng giữa before và after (+4.3) nhỏ hơn dao động này, nên cần chạy nhiều lần mỗi phiên bản (và lý tưởng là judge khác model) để so sánh đáng tin cậy. `eval_before.json` cũng chỉ là một lần chạy.
- **Điều chỉnh trên chính bộ eval:** vòng 2 được phát triển bằng cách đọc lỗi trên 22 case này, nên điểm `eval_after` có thể cao hơn hiệu quả thực trên câu hỏi mới.
- **Các kiểm tra trong loop:** chỉ kiểm tra những gì xác định được bằng chuỗi — URL có mặt, URL gắn với mã incident được nhắc, URL có xuất hiện trong kết quả tool, liên kết `/incidents/<mã>` chưa đọc. Chúng không chứng minh mọi claim đều có nguồn, không phát hiện suy luận sai (như A5 ở lần chạy trước) hay khẳng định trạng thái khi tool lỗi (H7), và chỉ nhận ra liên kết có dạng `/incidents/<mã>`.

## Bonus: 5 status API (`eval_bonus.json`)

Cấu hình: `openai / gpt-4o-mini`, judge `gpt-4o-mini`, `max_steps` 16, dataset `bonus`.

| | Lần 1 (bản bonus đầu) | `eval_bonus.json` (bản cuối) |
|---|---|---|
| Điểm bộ bonus | 66.5 | **81.5** |
| `bonus_points_0_15` | 9.97 | **12.22** |
| Case vi phạm kiểm tra bằng code | 2 (B6 thiếu `get_incident_details`, B10 chỉ gọi Vercel 1 lần) | 1 (B6) |
| Model calls / token agent (input, output) | 36 / 81,353, 6,528 | 38 / 91,847, 5,529 |
| Chi phí ước tính cả judge | ~$0.022 | ~$0.022 |

Điểm theo case (lần 1 → cuối): B1 80→65, B2 85→100, B3 55→55, B4 80→95, B5 75→100, B6 50→50, B7 55→100, B8 100→100, B9 35→100, B10 50→50. Lần 1 được lưu ở `eval_results_bonus_v1.json` (gitignore, không nộp); giữa hai lần chỉ sửa prompt bổ sung và thêm kiểm tra retry ở dưới, `eval_bonus.json` là lần chạy duy nhất của bản cuối.

Thay đổi trong `agent.py` cho bonus (không hard-code câu trả lời hay mã incident):

- **Kết nối tools:** dùng `getattr(tools, 'schemas', SCHEMAS)`; nếu bộ tools có `system_prompt` thì dùng nó và ghép thêm `EXTRA_RULES` (quy tắc chung về nhiều provider, 429, tham số, pagination, nhiễu, nguồn, injection). Tools bài chính không có hai thuộc tính này nên vẫn dùng `SYSTEM_PROMPT` cũ. Danh sách provider đọc từ schema.
- **Retry 429:** lỗi `RATE_LIMITED` luôn được trả lại cho model kèm `agent_note` ("429 không phải outage", đã thử bao nhiêu lần). Loop đếm số lần 429 theo từng tool (kèm provider nếu có); chờ theo `retry_after_seconds`/`Retry-After` nếu ≤ 5 giây (không có thì 1 giây), lâu hơn thì dừng retry; sau **3 lần** 429 thì không gọi API nữa — call tiếp theo vẫn được trả kết quả `RETRY_LIMIT_REACHED` cho đúng `tool_call_id` và ghi event `tool_blocked` (không phải `tool`) trong trace. Event `rate_limited` ghi attempt, retry_after, thời gian chờ và có dừng không.
- **Chưa retry sau 429 tạm thời:** nếu tool bị 429 còn lượt mà model định trả lời luôn, loop ghi `validation_unretried_rate_limit` và yêu cầu gọi lại một lần. Bằng chứng B10 (bản cuối): `rate_limited` (attempt 1) → `validation_unretried_rate_limit` → gọi lại `get_vercel_status` thành công; vi phạm "Cần gọi get_vercel_status ít nhất 2 lần" của lần 1 không còn.
- **Lỗi tham số:** `INVALID_ARGUMENTS` trả lại model kèm ghi chú "sửa tham số rồi gọi lại". B5: lần đầu lỗi định dạng giả lập, lần hai thành công, 100 điểm.
- **Pagination:** mỗi kết quả `get_incident_details` có `next_offset` được kèm `agent_note` cho biết đã đọc bao nhiêu/tổng số update và offset chưa đọc đầu tiên, nhắc không nói đã xem hết timeline. Không ép đọc hết (test contract cho phép model dừng ở trang đầu nếu đủ). B2: model đọc Vercel `sps6n39gm4sg` ở offset 0, 2, 4, 6 tới `next_offset = null`.
- **Liên kết sang provider khác:** kiểm tra liên kết chưa đọc nay xác định provider theo host của URL (githubstatus.com → github). B2: update Vercel trỏ tới `githubstatus.com/incidents/93njd414rpbx` → `validation_unread_reference` (provider github) → model đọc incident GitHub đó.
- **Context/nhiễu:** prompt yêu cầu bỏ qua `distractors` và trả lời đúng trọng tâm; B7 (tổng quan npm, có injection) từ liệt kê 8 incident lịch sử → chỉ trạng thái + nguồn, không xuất marker injection.
- **Provider không hỗ trợ:** prompt cấm gọi tool của bên khác thay thế; B9 (Linear/Slack) từ gọi cả 5 status tool → không gọi tool nào; khi model thêm link status page Linear/Slack không có trong kết quả tool, `validation_unknown_url` yêu cầu bỏ.

Giới hạn còn lại:

- **B6 (50):** model vẫn không gọi `get_incident_details` dù prompt yêu cầu đọc chi tiết khi hỏi về trạng thái/phạm vi incident — chưa có kiểm tra bằng code cho việc này vì khó xác định chắc chắn câu hỏi nào cần chi tiết.
- **B3 (55):** retry thành công nhưng judge cho rằng answer chưa nêu rõ đã gặp 429 và thử lại.
- **B1 (80 → 65):** judge trừ vì answer ghi description "Partial System Outage" thay vì indicator `major` của Cloudflare; thay đổi giữa hai lần ở case này không đến từ code (B1 không chạm tới retry hay pagination).
- **Chi phí pagination:** B2 dùng 13 model calls, 11 lần `get_incident_details` (đọc hết cả timeline GitHub 13 update dù câu hỏi chỉ cần một update của Vercel). Giới hạn 3 update/trang làm số bước tăng nhanh.
- **429 thật:** chỉ kiểm tra bằng unit test với HTTP 429 giả (`Retry-After: 2`) và kịch bản giả lập; chưa chạy `--mode live` khi API thật rate limit. Chờ bằng `time.sleep` trong loop làm chậm run tương ứng.
- **Đánh giá:** cùng giới hạn như bài chính — judge cùng model, một lần chạy mỗi phiên bản, và bản cuối được chỉnh sau khi đọc lỗi trên chính 10 case này.
- **Ảnh hưởng tới bài chính:** các thay đổi bonus chỉ kích hoạt với tools có `schemas`/`system_prompt` riêng, lỗi 429 hoặc timeline phân trang — không xuất hiện ở bộ tools bài chính. Hai thay đổi chạm tới đường chạy bài chính là ghi chú cho lỗi `INVALID_ARGUMENTS` và cách xác định provider của liên kết (`githubstatus.com` → github, cùng kết quả như trước với dữ liệu bài chính). `eval_after.json` được chạy trước các thay đổi này và không chạy lại; 37/37 test bài chính vẫn pass.
