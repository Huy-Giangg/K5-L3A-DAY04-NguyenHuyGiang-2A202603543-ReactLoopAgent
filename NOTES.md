# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Huy Giang
MSSV: 2A202603543

## 1. Case được chọn

- ID case: A5
- Câu trả lời của agent: "" (chuỗi rỗng, không kịp đưa ra câu trả lời do bị dừng ở trạng thái `max_steps`)
- Điểm ở lần chạy 1: 0/100 (Judge lý giải: "Agent hết số bước trước khi hoàn tất.")

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[Step 1] get_status(provider="github") -> matching_incidents_in_last_50: 50
[Step 2] get_status(provider="github", component="Actions") -> matching_incidents_in_last_50: 23
[Step 3] get_incident_details(provider="github", incident_id="t89g2q1800fr") -> ok
[Step 4] get_incident_details(provider="github", incident_id="pyynvrm1vskw") -> ok
[Step 5] get_incident_details(provider="github", incident_id="pfbmpxxfq1cy") -> ok
[Step 6] get_incident_details(provider="github", incident_id="h02n9572mx9y") -> ok
[Step 7] get_incident_details(provider="github", incident_id="xr3myzj7ms51") -> ok
[Step 8] get_incident_details(provider="github", incident_id="smtdswd1h2w4") -> ok
=> Status kết thúc: 'max_steps' (vượt quá ngân sách 8 steps, không kịp sinh final answer).
```

**Nguyên nhân gốc rễ:**
1. **Lỗi logic vòng lặp & nhận thức giới hạn dữ liệu:** Người dùng yêu cầu *"danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025"*. Sau khi gọi `get_status`, tool trả về `matching_incidents_in_last_50: 23`. Model hiểu nhầm rằng cần phải tra cứu chi tiết từng incident một để kiểm tra thời gian và nội dung, dẫn đến việc liên tục gọi `get_incident_details` từ step 3 đến step 8.
2. **Không nhận ra giới hạn snapshot sớm:** Dữ liệu tool chỉ chứa tối đa 50 sự cố gần nhất (thuộc tháng 11/2026 trong snapshot). Model không nhận ra ngay từ đầu rằng không thể có dữ liệu toàn diện của cả năm 2025 để dừng lại và thông báo giới hạn dữ liệu cho người dùng, dẫn đến cạn kiệt ngân sách bước (`max_steps`).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py`: Bổ sung chỉ dẫn vào `SYSTEM_PROMPT` yêu cầu agent nhận biết giới hạn của công cụ (chỉ tìm trong 50 sự cố gần nhất). Khi người dùng hỏi danh sách đầy đủ hoặc thời gian dài, agent cần tổng hợp dựa trên dữ liệu hiện có và nêu rõ giới hạn, tuyệt đối không duyệt từng incident vượt quá số bước cho phép.
  2. Trong `agent.py`: Cài đặt cơ chế kiểm tra nguồn (**Evidence Gate**):
     - Thu thập `url` từ kết quả gọi `get_incident_details` thành công.
     - Khi model đưa ra câu trả lời cuối cùng (`finish == 'stop'`), kiểm tra xem trong answer đã có URL nguồn hay chưa. Nếu thiếu URL nguồn và vẫn còn ngân sách steps, ghi event `validation_missing_source` vào `trace` và nhắc model bổ sung link dẫn chứng trước khi hoàn thành.
- Vì sao chọn thay đổi này:
  - Giúp agent tránh bẫy cạn kiệt số bước (`budget exhaustion`), biết tự lượng thứ dữ liệu và phản hồi trung thực về độ đầy đủ.
  - Đảm bảo câu trả lời luôn có nguồn dẫn chứng xác thực (`url` chính thức từ incident status), đáp ứng đầy đủ yêu cầu kiểm thử trong `test_validation.py` và tiêu chí chấm điểm của rubric.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 76.41 / 100 | 84.64 / 100 (+8.23) |
| Điểm case đã chọn (A5) | 0 / 100 | 91 / 100 (+91.0) |
| Số model calls | 85 calls | 93 calls |
| Token, chi phí nếu có | ~135k tokens ($0 - Groq Free Tier) | ~148k tokens ($0 - Groq Free Tier) |

- Lỗi đã giải quyết:
  1. **Khắc phục lỗi cạn kiệt ngân sách bước (Budget Exhaustion ở case A5):** Agent không còn rơi vào vòng lặp gọi tool liên tiếp để tra cứu từng incident khi gặp yêu cầu danh sách đầy đủ. Agent đã nhận thức được giới hạn 50 incident gần nhất từ snapshot và phản hồi trung thực cho người dùng, đưa điểm case A5 từ 0 lên 91 điểm.
  2. **Tự động bổ sung bằng chứng nguồn (Evidence Gate):** Đối với các case như E4, E5, A6, A7, H1, khi câu trả lời ban đầu thiếu URL incident đã quan sát, hệ thống ghi nhận `validation_missing_source` và nhắc model bổ sung đường dẫn xác thực trước khi kết thúc, giúp tăng điểm ở tiêu chí Evidence và Factuality.
- Lỗi còn lại hoặc mới xuất hiện:
  - Một số case phức tạp nhiều bước (như H1) cần gọi chuỗi nhiều tool liên tiếp đòi hỏi điều phối cẩn thận để không chạm ngưỡng rate limit TPM của provider.
  - Một vài case người dùng đưa ra câu hỏi bẫy hoặc prompt injection ngầm đòi hỏi agent cần lọc kỹ hơn ở khâu phân tích ngữ nghĩa của tool update body.

## 5. Giới hạn

1. **Giới hạn của LLM Judge:** Điểm số được chấm bởi LLM judge là một ước lượng xác suất, có thể có phương sai nhỏ (khoảng ±3-5 điểm) giữa các lần chạy do tính chất ngẫu nhiên của mô hình ngôn ngữ. Việc đạt điểm cao ở judge không đồng nghĩa câu trả lời hoàn hảo 100% trong mọi khía cạnh thực tế.
2. **Giới hạn của Snapshot:** Dữ liệu thử nghiệm là snapshot tĩnh đóng băng tại thời điểm tháng 11/2026 và chỉ bao gồm tối đa 50 sự cố gần nhất. Vì vậy, công cụ không thể trả lời các câu hỏi về các sự cố lịch sử xảy ra ngoài phạm vi 50 bản ghi này nếu không có cơ chế phân trang (pagination) mở rộng.
3. **Giới hạn của việc chạy một lần:** Chạy eval một lần trên 22 case chỉ phản ánh hiệu năng tại một thời điểm cụ thể, chưa chứng minh được độ ổn định (robustness) lâu dài trước các biến động về độ trễ mạng, thay đổi hạ tầng API hoặc các định dạng câu hỏi bất thường trong môi trường production thực tế.
