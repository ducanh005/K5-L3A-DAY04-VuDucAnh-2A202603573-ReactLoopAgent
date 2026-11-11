# Ghi chú phân tích · Day 04

Họ và tên: Vũ Đức Anh
MSSV: 2A202603573

## 1. Case được chọn

- ID case: `H1` (đồng thời liên quan cùng nhóm lỗi với `A8`, `H2`, `H6`)
- Câu trả lời của agent:
  > Incident GitHub h02n9572mx9y có tên là "Incident with Actions" và đã được khôi phục. Nó bắt đầu vào ngày 4 tháng 11 năm 2026 lúc 17:40 và được cập nhật lần cuối vào ngày 7 tháng 11 năm 2026 lúc 01:10. Nó ảnh hưởng đến thành phần "Actions". Có 4 cập nhật cho incident này.
  > - Cập nhật đầu tiên: "Chúng tôi đang điều tra các báo cáo về giảm tính sẵn sàng cho Actions"
  > - Cập nhật tiếp theo: "Tình trạng giảm đối với Actions đã được giảm thiểu. Chúng tôi đang theo dõi để đảm bảo ổn định."
  > - Cập nhật cuối cùng: "Vui lòng tham khảo tóm tắt kết hợp trong incident liên quan: https://www.githubstatus.com/incidents/xr3myj7ms51"
  > Incident Cloudflare yaa0witbl2me có tên là "Workers Builds Issues" và đã được khôi phục... Không có bằng chứng chung về nguyên nhân giữa hai incident này.
- Điểm ở lần chạy 1: **60 / 100** (`raw_judge_score`: 80, bị khóa trần `score_cap`: 60 do lỗi mechanical)

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {"provider": "github", "incident_id": "h02n9572mx9y"},
    "result": {
      "ok": true,
      "id": "h02n9572mx9y",
      "updates": [
        {
          "status": "resolved",
          "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"
        }
      ],
      "url": "https://stspg.io/7pwv28hvkmz8"
    }
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {"provider": "cloudflare", "incident_id": "yaa0witbl2me"},
    "result": {
      "ok": true,
      "id": "yaa0witbl2me",
      "url": "https://www.cloudflarestatus.com/incidents/yaa0witbl2me"
    }
  }
]
Mechanical violations: ["Cần tool calls ở ít nhất 2 model steps khác nhau."]
Judge reason: "Đáp án cũng không dẫn nguồn cho thông tin cung cấp, điều này làm giảm độ tin cậy của thông tin."
```

**Phân tích nguyên nhân:**
1. **Lỗi ở Agent Loop (Thiếu cơ chế kiểm tra nguồn - Evidence Gate):** Trong `step 1`, cả hai lệnh `get_incident_details` đều trả về trường `"url"` (`https://stspg.io/7pwv28hvkmz8` và `https://www.cloudflarestatus.com/incidents/yaa0witbl2me`). Tuy nhiên, ở `step 2`, khi model sinh câu trả lời cuối cùng (`finish == 'stop'`), model bỏ quên hoàn toàn cả 2 URL nguồn này (lỗi tương tự cũng xảy ra ở `A8`). Do vòng lặp `run_agent` chấp nhận ngay câu trả lời mà không kiểm tra sự tồn tại của URL đã quan sát, câu trả lời bị mất điểm `evidence`.
2. **Lỗi ở System Prompt (Thiếu hướng dẫn điều tra nhiều bước - Multi-step):** Bản cập nhật của incident `h02n9572mx9y` chỉ dẫn sang incident gộp `xr3myzj7ms51` để xem nguyên nhân và ảnh hưởng. Do `SYSTEM_PROMPT` ban đầu quá ngắn gọn, model không biết phải tiếp tục gọi `get_incident_details(provider="github", incident_id="xr3myzj7ms51")` ở `step 2`, dẫn đến vi phạm `min_tool_steps: 2` (tương tự ở `H6` và `H2` chỉ gọi `get_status` mà không gọi `get_incident_details`).

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. **Tại `agent.py` (Hàm `run_agent`):** Bổ sung **Evidence Gate (`validation_missing_source`)**. Lưu lại danh sách `observed_urls` từ mọi lần gọi tool thành công (`result.get('ok') and result.get('url')`). Khi model dừng (`finish == 'stop'`), nếu câu trả lời chưa chứa URL nguồn đã quan sát, agent ghi event `{'event': 'validation_missing_source', 'step': step, 'observed_urls': ...}` vào `trace` và nếu `step < max_steps` thì gửi tiếp một user message yêu cầu model bổ sung URL nguồn trước khi hoàn tất.
  2. **Tại `agent.py` (`SYSTEM_PROMPT`):** Cụ thể hóa các quy tắc điều tra: (a) Bắt buộc gọi `get_incident_details` khi có incident liên quan hoặc khi cần kiểm tra phạm vi/nguyên nhân; (b) Bắt buộc gọi tiếp `get_incident_details` ở step sau nếu nội dung update dẫn chiếu sang một incident ID khác (như `xr3myzj7ms51`); (c) Bắt buộc trích dẫn nguyên văn URL nguồn và nêu rõ giới hạn 50 incident gần nhất của snapshot.
- **Vì sao chọn thay đổi này:**
  - Kết hợp cả **kiểm soát bằng code trong loop (Evidence Gate)** để đảm bảo 100% tính tuân thủ nguồn dẫn (vượt qua toàn bộ unit test trong `tests/test_validation.py`), vừa **cải thiện hướng dẫn chiến lược trong prompt** để giải quyết bài toán suy luận nhiều bước (`H1`, `H2`, `H6`, `E2`) mà không cần hard-code bất kỳ ID hay câu trả lời nào.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (`score_0_100`) | **79.09 / 100** (22/22 case) | **84.09 / 100** (22/22 case, **+5.00 điểm**) |
| Điểm case đã chọn | `H1`: 60/100 (đã kích hoạt `validation_missing_source` bổ sung đủ URL ở lần 2)<br>`H6` (cùng lỗi multi-step `xr3myzj7ms51`): 60/100 $\rightarrow$ **80/100** (+20đ)<br>`H2` (thiếu `get_incident_details`): 50/100 $\rightarrow$ **100/100** (+50đ)<br>`A6` & `A8` (thiếu URL nguồn): 75/100 $\rightarrow$ **95/100** & **80/100** | Xem cột bên |
| Số model calls (Agent) | **43 calls** | **49 calls** (+6 calls do tra cứu sâu nhiều bước & sửa nguồn) |
| Token, chi phí nếu có | **34,098 tokens**<br>Agent: \$0.000512 · Judge: \$0.000676 (Tổng: **\$0.001188**) | **64,236 tokens**<br>Agent: \$0.000862 · Judge: \$0.000974 (Tổng: **\$0.001836**) |

- **Lỗi đã giải quyết:**
  1. **Giải quyết triệt để lỗi quên dẫn nguồn URL incident:** Cổng `validation_missing_source` đã tự động phát hiện và buộc model bổ sung URL khi thiếu (ví dụ tại `H1` step 2 đã bắt lỗi thiếu URL và yêu cầu model sửa thành công ở step 3; `A6` tăng từ 75 lên 95 điểm; `A8` tăng từ 75 lên 80 điểm).
  2. **Giải quyết lỗi dừng sớm / thiếu bước điều tra sâu:** Case `H2` tăng vọt từ **50 lên 100 điểm** (đã gọi đủ `get_status` và `get_incident_details`); Case `H6` tăng từ **60 lên 80 điểm** (đã tự động đi theo liên kết sang incident `xr3myzj7ms51` qua 3 model steps liên tiếp, xóa bỏ hoàn toàn lỗi `mechanical` cap 60).
  3. **Giải quyết lỗi không nêu giới hạn dữ liệu & xử lý lỗi bộ phận:** Case `H7` tăng từ **60 lên 100 điểm**, `H8` tăng từ **60 lên 85 điểm**, `E5` tăng từ **70 lên 95 điểm**, `A5` tăng từ **75 lên 95 điểm**.
- **Lỗi còn lại hoặc mới xuất hiện:**
  1. Tại case `H1`, dù cổng `validation_missing_source` đã giúp agent bổ sung đầy đủ URL nguồn, model `mistral-nemo` ở step 1 gọi đồng thời 2 lệnh `get_incident_details` nhưng chưa tách bước để gọi tiếp incident thứ 3 (`xr3myzj7ms51`) nên vẫn bị `score_cap = 60` bởi mechanical check `min_tool_steps: 2`.
  2. Một số biến thiên điểm số tự nhiên của LLM/Judge giữa hai lần chạy ở các case đơn lẻ (như `A1`), phản ánh tính bất định (non-determinism) của mô hình ngôn ngữ nhỏ khi tổng hợp văn bản.

## 5. Giới hạn

1. **Giới hạn của điểm LLM Judge và cổng kiểm tra cơ học:**
   - Điểm của LLM Judge chỉ là **ước lượng**, có thể dao động giữa các lần gọi và chịu thiên lệch (self-bias) khi dùng chung một model (`mistralai/mistral-nemo`) cho cả Agent lẫn Judge.
   - Cổng kiểm tra nguồn (`validation_missing_source`) trong `agent.py` chỉ kiểm tra sự hiện diện của chuỗi URL (`any(u in answer for u in observed_urls)`), **không chứng minh** mọi khẳng định (claims) trong câu trả lời đều khớp chính xác với nội dung bên trong URL đó.
2. **Giới hạn của dữ liệu Snapshot và Tools:**
   - Dữ liệu snapshot là tĩnh (chụp ngày 09/11/2026), chỉ chứa tối đa **50 incident gần nhất** của mỗi nhà cung cấp và mỗi incident chỉ giữ tối đa **3 bản cập nhật** (2 mới nhất + 1 đầu tiên, cắt ở 900 ký tự).
   - Công cụ tìm kiếm chỉ so khớp chuỗi con (`substring match`), không phải tìm kiếm ngữ nghĩa (`semantic search`), nên **không tìm thấy incident không chứng minh sự cố không tồn tại**.
3. **Giới hạn của phương pháp đánh giá một lần chạy (Single-run evaluation):**
   - Mỗi phiên bản (`eval_before.json` và `eval_after.json`) chỉ chạy **1 lần duy nhất** trên 22 case (`n = 1`). Do LLM có tính ngẫu nhiên trong việc chọn tool và diễn đạt, kết quả một lần chạy **không chứng minh độ ổn định thống kê (statistical stability)** trong môi trường thực tế; để đánh giá độ ổn định cần chạy lặp lại nhiều lần (`pass@k` hoặc lấy trung bình/độ lệch chuẩn qua nhiều seed).
