# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Hoàng Cương
MSSV: 2A202603568

## 1. Case được chọn

- ID case: H1 (điều tra incident GitHub `h02n9572mx9y`, theo liên kết trong incident, đối chiếu với Cloudflare `yaa0witbl2me`)
- Câu trả lời của agent: Nêu đúng trạng thái, thời gian, component của hai incident và không kết luận nguyên nhân chung, nhưng chỉ nhắc `xr3myzj7ms51` như một đường link; không đọc nó nên thiếu nguyên nhân và ảnh hưởng mà đề bài yêu cầu ("theo liên kết do incident cung cấp"). Câu trả lời cũng không nêu dữ liệu có thể bị cắt.
- Điểm ở lần chạy 1: 60/100. `grade.mechanical.violations`: "Cần tool calls ở ít nhất 2 model steps khác nhau."

## 2. Bằng chứng trong trace

Cả 4 tool call nằm trong một model step (step 1), rồi model dừng ở step 2 mà chưa đọc incident được dẫn tới. Update của `h02n9572mx9y` có nội dung: "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51".

```text
{"event": "model", "step": 1, "finish": "tool_calls", "calls": 4}
{"event": "tool", "step": 1, "name": "get_status", "arguments": {"provider": "github"}, ...}
{"event": "tool", "step": 1, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "h02n9572mx9y"}, ...
    "updates": [{"status": "resolved", "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"} ...
{"event": "tool", "step": 1, "name": "get_status", "arguments": {"provider": "cloudflare"}, ...}
{"event": "tool", "step": 1, "name": "get_incident_details", "arguments": {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}, ...}
{"event": "model", "step": 2, "finish": "stop", "calls": 0}
```

Nguyên nhân có thể: model gọi song song mọi thứ ở step 1 vì các ID có sẵn trong câu hỏi, nên chưa thấy link trong tool result; sau khi thấy link ở step 2 thì trả lời luôn. Không có gì trong loop hay system prompt buộc model đọc tiếp. Lỗi nằm ở vòng lặp (thiếu bước kiểm tra), không phải giới hạn dữ liệu: tool đã trả đủ thông tin để biết cần đọc `xr3myzj7ms51`.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, thêm `linked_incident_ids` (tìm `/incidents/<id>` trong body các update của incident đọc thành công) và theo dõi `pending` (được dẫn tới) so với `fetched` (đã đọc chi tiết). Khi model đưa final answer mà còn incident được dẫn tới nhưng chưa đọc, ghi event `validation_unfollowed_link` vào trace và nhắc model gọi `get_incident_details` cho các ID đó rồi trả lời lại. Chỉ nhắc một lần và chỉ khi còn lượt trong `max_steps`.
- Vì sao chọn thay đổi này: cùng kiểu với kiểm tra URL nguồn ở Bài 4 (kiểm tra bằng code, không dựa vào prompt), nhắm đúng lỗi H1/H6 "gọi tiếp khi thấy incident liên quan". Giới hạn: chỉ nhận link dạng `/incidents/<id>` và dùng provider của incident chứa link; chưa xử lý nhắc về dữ liệu bị cắt (`truncated`).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 84.36 | 88.18 |
| Điểm case đã chọn (H1) | 60 | 89 |
| Số model calls | 56 | 60 |
| Token, chi phí nếu có | 141.918 token (agent) | 171.616 token (agent) |

Cùng `config` ở hai file: openai / gpt-5-nano, judge gpt-5-nano, max_steps 8, dataset core. Chi phí tiền không tính (provider không trả giá).

- Lỗi đã giải quyết: H1. Trace lần 2 có `validation_unfollowed_link` ở step 2 với `xr3myzj7ms51`; agent gọi `get_incident_details` cho incident đó ở step 3 (tool call ở 2 model step khác nhau, hết vi phạm cơ học) rồi trả lời với nguyên nhân/ảnh hưởng từng bên. H1 từ 60 lên 89. Kiểm tra này cũng kích hoạt ở H6 (điểm không đổi, 91).
- Lỗi còn lại hoặc mới xuất hiện: A5 (76) và H4 (76), H8 (74), H7 (74) vẫn thấp. Một số case giảm: A4 98->89, E6 92->84, A2 92->86, A7 92->87. Đổi lại chi phí: +4 model calls và +29.698 token, phần lớn từ H1/H6 phải gọi lại và các case có `validation_missing_source` (E3, A7).

## 5. Giới hạn

- Điểm judge là ước lượng do chính gpt-5-nano chấm nên có thể thiên lệch; một số lệch điểm (ví dụ A4 98->89) có thể do judge chứ không do agent.
- Chỉ chạy một lần mỗi phiên bản. Nhiều case tăng/giảm mà không có thay đổi hành vi nào liên quan: H3 50->93 và H7 48->74 không có event validation nào trong trace, tức là cải thiện đến từ biến động của model chứ không phải từ thay đổi của agent. Vì vậy mức tăng 84.36->88.18 không thể gán toàn bộ cho thay đổi; chỉ H1 (và H6) có bằng chứng trace trực tiếp.
- Snapshot cố định và chỉ có 50 incident gần nhất, nên không đánh giá được dữ liệu ngoài phạm vi đó. Thay đổi chỉ nhận link dạng `/incidents/<id>` và chưa xử lý dữ liệu bị cắt (`truncated`).
