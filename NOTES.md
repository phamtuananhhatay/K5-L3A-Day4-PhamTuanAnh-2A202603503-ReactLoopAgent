# Ghi chú phân tích · Day 04

Họ và tên: Phạm Tuấn Anh
MSSV: 2A202603503


## 1. Case được chọn

- ID case: A2
- Câu trả lời của agent: Sự cố với ID wp49bp8bqo6c liên quan đến việc "Fedramp Customers have Delayed Purges" đã được Cloudflare xác định và không ảnh hưởng đến các khách hàng không thuộc FedRAMP. Dưới đây là thông tin chi tiết về sự cố: Tên sự cố: Fedramp Customers have Delayed Purges, Trạng thái: Identified, Cập nhật: Cloudflare has identified a bug with our purge pipeline impacting fedramp customers. Stale content may be served until expired. Non-fedramp customers are not impacted. Nguồn: https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c. Tóm lại, sự cố cụ thể này không ảnh hưởng đến CLB vì CLB không thuộc FedRAMP.
- Điểm ở lần chạy 1: 80 / 100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_incident_details",
  "arguments": {
    "provider": "cloudflare",
    "incident_id": "wp49bp8bqo6c"
  },
  "result": {
    "ok": true,
    "provider": "cloudflare",
    "id": "wp49bp8bqo6c",
    "name": "Fedramp Customers have Delayed Purges",
    "status": "identified",
    "updates": [
      {
        "status": "investigating",
        "body": "Cloudflare has identified a bug with our purge pipeline impacting fedramp customers. Stale content may be served until expired. Non-fedramp customers are not impacted."
      }
    ],
    "url": "https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c"
  }
}
```

- Nhận xét của Judge (grade.reason): "Câu trả lời xác định đúng rằng sự cố không ảnh hưởng đến khách hàng không thuộc FedRAMP và cung cấp thông tin cụ thể theo yêu cầu. Tuy nhiên, không khẳng định rõ rằng mọi vấn đề cache của CLB đều được loại trừ."
- Nguyên nhân: Agent có xu hướng suy diễn tuyệt đối hóa ("sự cố không ảnh hưởng CLB thì CLB không sao"), chưa phân biệt ranh giới giữa "sự cố này không ảnh hưởng" và "không thể loại trừ các nguyên nhân khác ngoài snapshot", đồng thời trong các lần chạy nếu thiếu cơ chế kiểm soát nguồn, agent dễ bỏ quên URL metadata từ tool result.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py`: Cài đặt cơ chế Evidence Gate (kiểm tra dẫn nguồn URL). Lưu tập hợp `observed_urls` từ kết quả `get_incident_details`. Khi model chuẩn bị kết thúc (`finish == 'stop'`), kiểm tra xem trong `answer` có chứa URL sự cố hay không. Nếu thiếu: ghi event `validation_missing_source` vào `trace` và gửi tin nhắn yêu cầu model bổ sung URL trong giới hạn `max_steps`.
  2. Bổ sung xử lý `status = 'max_steps'` nếu model không bổ sung nguồn khi hết ngân sách bước.
- Vì sao chọn thay đổi này:
  - Khắc phục triệt để lỗi thiếu bằng chứng kiểm chứng (evidence) và giúp agent tuân thủ hợp đồng kiểm tra nguồn theo `test_validation.py`.
  - Giúp câu trả lời của Agent luôn có căn cứ xác thực, người dùng có thể click trực tiếp vào URL sự cố để kiểm tra trạng thái thực tế.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 77.27 / 100 | 76.59 / 100 |
| Điểm case đã chọn (A2) | 75 / 100 | 100 / 100 |
| Số model calls (A2) | 2 | 2 |
| Token, chi phí nếu có (A2) | 1,647 tokens | 1,676 tokens |

- Lỗi đã giải quyết: Case A2 tăng từ 75 lên 100/100 điểm tuyệt đối. Agent đã trích dẫn đầy đủ URL sự cố chính xác từ tool result (`https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`), phân biệt rõ phạm vi ảnh hưởng chỉ xảy ra với khách hàng FedRAMP, không khẳng định sai lệch về lỗi dịch vụ ngoài phạm vi.
- Lỗi còn lại hoặc mới xuất hiện: Ở một số case phức tạp khác (như prompt injection H5/H7 hoặc lọc nhiễu H6), điểm số có sự dao động nhỏ do tính ngẫu nhiên (sampling temperature) của LLM và judge model.

## 5. Giới hạn

1. **Giới hạn của điểm Judge:** Điểm chấm bởi LLM Judge (GPT-4o-mini) là điểm ước lượng dựa trên prompt chấm điểm; judge có thể có thiên kiến (bias) hoặc đánh giá khắt khe ở câu từ dù ngữ nghĩa đã đầy đủ.
2. **Giới hạn của Snapshot:** Dữ liệu thử nghiệm là snapshot cố định (tối đa 50 incidents gần nhất); không đại diện hoàn toàn cho tình trạng mạng thời gian thực (live).
3. **Giới hạn của số lần chạy:** Chạy một lần mỗi phiên bản chỉ phản ánh một lát cắt xác suất của mô hình, chưa chứng minh được độ ổn định (variance) qua nhiều lần chạy lặp lại.
