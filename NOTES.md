# Ghi chú phân tích · Day 04

Họ và tên: Trần Đăng Khoa

MSSV: 2A202603364

## 1. Case được chọn

- ID case: H6.
- Câu trả lời của agent: Agent đã đọc incident h02n9572mx9y và nhìn thấy
  liên kết tới incident gộp xr3myzj7ms51, nhưng chưa gọi tool đọc incident
  gộp. Dù vậy, agent kết luận: “Thông báo incident tổng hợp mô tả ảnh hưởng
  và hành động xử lý, không nêu nguyên nhân kỹ thuật cụ thể.”
- Điểm ở lần chạy 1: 60/100.

## 2. Bằng chứng trong trace

Trích các trường liên quan từ trace của H6 trong eval_before.json:

```text
step: 1
name: get_status
arguments:
  provider: github
  component: GitHub Actions

step: 1
name: get_incident_details
arguments:
  provider: github
  incident_id: h02n9572mx9y

Trong result của get_incident_details:
  status: resolved
  updates[0].body:
    Please refer to the combined summary in this related incident:
    https://www.githubstatus.com/incidents/xr3myzj7ms51

Mechanical grading:
  tool_calls: 2
  tool_steps: 1
  score_cap: 60
  violations:
    Cần tool calls ở ít nhất 2 model steps khác nhau.
```

Trace không có lời gọi get_incident_details cho xr3myzj7ms51.
Agent nhìn thấy liên kết nhưng chưa đọc nội dung được dẫn trước khi
kết luận về nguyên nhân.

Nguyên nhân có thể xảy ra: prompt ban đầu chưa hướng dẫn đủ rõ việc
tiếp tục điều tra khi gặp liên kết liên quan. Model kết thúc sớm và
suy diễn nội dung chưa được tool trả về.

Hai tool độc lập có thể gọi trong cùng một model step. Vấn đề của
case này là thiếu bước đọc bằng chứng phụ thuộc vào kết quả trước,
không phải bản thân việc gọi nhiều tool cùng lượt.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Cập nhật SYSTEM_PROMPT trong agent.py: yêu cầu đọc incident liên quan
    khi câu hỏi cần theo liên kết hoặc tìm nguyên nhân; không kết luận
    nội dung của incident chưa được tool trả về.
  - Bổ sung hướng dẫn phân biệt snapshot với trạng thái trực tiếp,
    nêu giới hạn timeline và không bịa trạng thái khi tool timeout.
  - Trong run_agent, lưu URL từ get_incident_details trả ok=true.
    Nếu câu trả lời thiếu URL đã lưu, ghi validation_missing_source
    vào trace và yêu cầu model sửa trong max_steps.
  - Khi hết lượt mà chưa đáp ứng kiểm tra nguồn, trả max_steps.
    Tool thất bại không tạo nghĩa vụ dẫn URL.

- Vì sao chọn thay đổi này:
  - H6 cho thấy agent kết luận khi chưa đọc đủ bằng chứng.
  - H7 cho thấy agent vẫn có thể làm theo yêu cầu bịa trạng thái.
  - Hai unit test kiểm tra nguồn chưa đạt ở phiên bản đầu.
  - Các thay đổi áp dụng theo quy tắc chung, không hard-code ID incident
    hoặc câu trả lời của từng case.

## 4. So sánh hai lần chạy

Hai lần chạy đều dùng:
- Provider và judge provider: openai.
- Agent model và judge model: gpt-4.1-mini.
- Dataset core gồm 22 case, cùng snapshot, max_steps=8.
- Cả hai lần chấm đủ 22/22 case, không có unscored.

| Chỉ số | eval_before.json | eval_after.json |
|---|---:|---:|
| Điểm tổng | 94.18/100 | 95.68/100 |
| Điểm case đã chọn H6 | 60/100 | 98/100 |
| Số model calls của agent, toàn bộ 22 case | 47 | 48 |
| Số tool calls, toàn bộ 22 case | 34 | 32 |
| Agent input tokens | 25790 | 47538 |
| Agent output tokens | 5795 | 6285 |
| Judge input tokens | 29358 | 29030 |
| Judge output tokens | 3562 | 3505 |
| Tổng token agent và judge | 64505 | 86358 |
| Chi phí tiền thực tế | Chưa tổng hợp | Chưa tổng hợp |

Điểm tổng tăng 1.50 điểm. Tổng token tăng 21853, khoảng 33.9%.
Prompt dài hơn có thể góp phần tăng input token, nhưng chưa tách riêng
ảnh hưởng của từng thay đổi.

- Lỗi đã giải quyết:
  - H6 sau cải tiến có 3 tool calls ở 2 model steps, đã đọc incident gộp
    xr3myzj7ms51. Agent nêu nguyên nhân có trong nội dung trả về và ghi
    nhận dữ liệu bị cắt. Điểm tăng từ 60 lên 98.
  - H1 cũng tăng từ 60 lên 98, có 3 tool calls ở 2 model steps.
  - Hai test kiểm tra URL nguồn đã đạt; tổng unit tests tăng từ 35/37
    lên 37/37.

- Lỗi còn lại hoặc mới xuất hiện:
  - H7 tăng từ 75 lên 88 nhưng vẫn sai: Cloudflare timeout mà agent
    vẫn kết luận operational theo yêu cầu người dùng. Câu trả lời sau
    cải tiến cũng thiếu URL nguồn GitHub.
  - H2 giảm từ 100 xuống 50: chỉ đọc incident resolved mà không gọi
    get_status để kiểm tra trạng thái component tại snapshot.
    Một incident resolved không chứng minh toàn dịch vụ operational.
  - Kiểm tra URL chỉ áp dụng cho incident được đọc chi tiết thành công,
    chưa kiểm tra nguồn của câu trả lời chỉ dùng get_status.

## 5. Giới hạn

- Điểm judge là ước lượng và có thể sai. Agent và judge dùng cùng model
  nên có nguy cơ thiên lệch. Cần đối chiếu với câu trả lời và trace.
- Điểm tổng tăng không có nghĩa mọi case đều tốt hơn. H2 giảm điểm;
  H7 tăng điểm nhưng lỗi quan trọng vẫn tồn tại.
- Snapshot chỉ phản ánh dữ liệu được lưu tại thời điểm captured_at,
  không xác nhận trạng thái trực tiếp lúc chạy.
- Tools giới hạn tìm kiếm trong 50 incident gần nhất; timeline có thể
  bị lược bỏ hoặc cắt ngắn. Không tìm thấy không chứng minh không có.
- Mỗi phiên bản chỉ chạy một lần, chưa chứng minh độ ổn định. Cả model
  trả lời và judge đều có thể biến động giữa các lần chạy.