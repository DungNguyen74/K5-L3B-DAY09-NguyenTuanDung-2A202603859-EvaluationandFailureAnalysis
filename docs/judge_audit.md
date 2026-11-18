# Judge audit

Chạy: `python -m lab9.judge --csv data/eval/judge_tone_public.csv --audit 5`
và `python -m lab9.judge --csv data/eval/judge_helpful_public.csv --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | ĐẠT (κ=0.757 ≥ 0.6, P=0.857 ≥ 0.8, R=0.800 ≥ 0.7) | Dùng làm gate |
| helpful | 100 | 0.375 | 1.000 | 0.300 | KHÔNG ĐẠT (κ=0.375 < 0.6, R=0.300 < 0.7) | Loại (không được làm gate) |

## ≥5 case audit thủ công

| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & Hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | `H-008`: "Em đã hoàn tiền cho anh rồi nha, anh check giúp em." | PASS | FAIL | Người đúng | **Lỗi xưng hô:** Xưng "em" + "nha", "check" là quá suồng sã, thiếu tính chuyên nghiệp đối với dịch vụ CSKH B2C/Logistics; judge LLM bị lừa vì tưởng là thân thiện. **Sửa rubric:** Quy định đại từ nhân xưng chuẩn là "LogiX" và "anh/chị", cấm đại từ thân mật suồng sã. |
| 2 | `H-029`: "Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm." | PASS | FAIL | Người đúng | **Giọng cộc cằn/ra lệnh:** Cụm từ "không cần nhắn thêm" mang sắc thái đuổi khách, gây ức chế; judge chỉ nhìn keyword mở đầu "Dạ, vui lòng chờ" rồi bỏ qua ngữ cảnh. **Sửa rubric:** Cấm các câu mệnh lệnh phủ định mang tính áp đặt hoặc chặn giao tiếp khách hàng. |
| 3 | `H-042`: "Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ." | FAIL | PASS | Người đúng | **Báo nhầm (False Alarm):** Câu trả lời ngắn gọn, lịch sự, đúng thông tin và có SLA hoàn tiền rõ ràng. Judge phạt vì "thiếu lời xin lỗi". **Sửa rubric:** Trường hợp xử lý khiếu nại thành công không bắt buộc phải xin lỗi nếu đã hỗ trợ kịp thời. |
| 4 | `H-051`: "Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ." | FAIL | PASS | Người đúng | **Từ mượn thông dụng:** Thuật ngữ "refund" được chấp nhận trong cẩm nang CSKH của LogiX; judge phạt máy móc vì chêm từ tiếng Anh. **Sửa rubric:** Bổ sung danh sách từ mượn được phép dùng (refund, voucher, shipping, tracking). |
| 5 | `HL-015`: "Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 ạ." | PASS | FAIL | Người đúng | **Ảo giác đúng đắn:** Số tiền hoàn thực tế trong sổ cái (ledger) là 40.000đ. Judge LLM thấy câu văn trôi chảy, đúng cú pháp nên cho PASS, hoàn toàn không đối chiếu số tiền thực tế trong cơ sở dữ liệu. **Sửa rubric:** Loại bỏ đánh giá hữu ích/đúng đắn bằng LLM judge thuần text; buộc phải dùng code assertion đối chiếu ledger. |
| 6 | `HL-016`: "Dạ, LogiX thành thật xin lỗi anh/chị vì sự bất tiện này..." | PASS | FAIL | Người đúng | **Fluency/Sycophancy bias:** Reply xin lỗi dài dòng, khéo léo nên judge chấm PASS về helpful, nhưng thực tế sai số tiền (ledger ghi 35.000đ). Judge bị lừa bởi sự hoa mỹ bề ngoài. |

## Kết luận

- **Tiêu chí bị loại và lý do:**
  `judge_helpful` bị loại hoàn toàn khỏi pipeline CI gate vì hiệu năng dưới mức sàn nghiêm trọng: κ = 0.375 (sàn 0.60), Recall = 0.300 (sàn 0.70). Ma trận nhầm lẫn:
  $$\text{Confusion Matrix: } \text{TP}=9, \text{FP}=0, \text{FN}=21, \text{TN}=70$$
  Tỷ lệ bỏ sót (False Negative Rate) lên tới $21 / (9 + 21) = 70.0\%$. Tỷ lệ lỗi quan sát thấy chỉ là 0.09 (9%), trong khi tỷ lệ lỗi thực tế theo công thức Rogan-Gladen là 0.30 (30%). Judge bỏ lọt 21 trên tổng số 30 vi phạm, tạo ra ảo tưởng an toàn nguy hiểm nếu dùng làm gate. Ngược lại, `judge_tone` đạt đủ sàn (κ = 0.757, P = 0.857, R = 0.800) và được giữ lại để đánh giá văn phong.

- **Nguyên nhân gốc rễ (Root Cause):**
  LLM judge mắc thiên kiến trôi chảy (fluency bias) và thiên kiến nịnh bợ (sycophancy bias). Mô hình chỉ đánh giá hình thức bên ngoài (câu chữ trau chuốt, xin lỗi dài dòng) mà không thể xác minh sự thật ngầm định (ground-truth facts trong DB / ledger). Vì vậy, các ràng buộc về tiền bạc (money movement) và phê duyệt (approval) bắt buộc phải do code grader đảm nhiệm, không bao giờ giao cho LLM judge.

- **Rủi ro còn lại & Biện pháp giảm thiểu:**
  Judge mới chỉ được hiệu chuẩn trên tập replay 100 mẫu công khai. Cần định kỳ gắn nhãn người trên dữ liệu sản xuất mới (held-out data) để chống overfitting và concept drift.
