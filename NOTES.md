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

- Sửa gì, ở đâu: Trong `agent.py`, thêm `linked_incident_ids` (tìm `/incidents/<id>` trong body các update của incident đọc thành công) và theo dõi `pending` (được dẫn tới) so với `fetched` (đã đọc chi tiết). Khi model đưa final answer mà còn incident được dẫn tới nhưng chưa đọc, ghi event `validation_unfollowed_link` vào trace và nhắc model gọi `get_incident_details` cho các ID đó rồi trả lời lại. Chỉ nhắc một lần và chỉ khi còn lượt trong `max_steps`.- Vì sao chọn thay đổi này: cùng kiểu với kiểm tra URL nguồn ở Bài 4 (kiểm tra bằng code, không dựa vào prompt), nhắm đúng lỗi H1/H6 "gọi tiếp khi thấy incident liên quan". Giới hạn: chỉ nhận link dạng `/incidents/<id>` và dùng provider của incident chứa link; chưa xử lý nhắc về dữ liệu bị cắt (`truncated`).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
