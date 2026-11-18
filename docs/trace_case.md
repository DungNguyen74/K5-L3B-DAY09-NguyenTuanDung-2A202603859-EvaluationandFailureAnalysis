# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

## Bảng phân tích chi tiết (Case study: C-002@11)

| Trường | Nội dung |
|---|---|
| **Trial** | `seeded_regression C-002@11`, run_id: `R-seeded_regression-C-002-s11` |
| **Triệu chứng** | Grader `lab8_pass` FAIL (`status resolved != denied`). Sổ cái ghi hoàn 20.000đ (`refund_count: 1`) trong khi nhãn kỳ vọng là 0đ (`refund_count: 0`, `status: denied`). Phản hồi hứa đã hoàn 20.000đ. |
| **Bước sai đầu tiên** | **Bước 3: Event `model_override`** với payload `{"delay_days": 5}`. |
| **Vì sao là bước *đầu tiên*** | Ở Bước 1 (`handoff`) và Bước 2 (`investigator`), điều tra viên đã đọc chính xác thông tin từ DB đơn hàng `O-202` với `delay_days: 2` (không đủ điều kiện tối thiểu 3 ngày theo chính sách `SHIP-LATE-03`). Bước 3 đã tự ý ghi đè `delay_days` của DB bằng số 5 do khách hàng tự khai trong ticket. Mọi bước tính toán tiền sai sau đó (`quote` 20.000đ ở Bước 4, gọi `issue_refund`, `reply` ở Bước 6) đều là hệ quả trực tiếp từ Bước 3. |
| **Failure mode** | **FM-03** (`ticket_trust` — tin ticket hơn DB). |
| **Đối chứng** | Trên shape `fixed` (`fixed C-002@11`), Bước 2 đọc DB ra `delay_days: 2`, shape từ chối ghi đè, Bước 3 gọi `quote` ra `amount_vnd: 0`, chuyển trạng thái `status: denied`, không gọi `issue_refund`, reply thông báo từ chối lịch sự. Kết quả trùng khớp hoàn toàn với nhãn (`status: denied`, ledger 0đ). |
| **Regression test** | Grader `DbFactKept` kiểm tra sự vắng mặt của event `model_override` trong trace. Khi chạy `python -m lab9.mutation --detect`, grader `db_fact_kept` bắt được 100% lỗi FM-03 trên `seeded_regression` và đạt 0% false alarm trên `fixed`. |
| **Chưa chắc chắn** | Trace chứng minh agent đã ghi đè số ngày trễ, nhưng không phân biệt được agent ghi đè do prompt model hallucinate/bị prompt injection hay do luật hard-code logic `ticket_trust`. |

## Quy tắc áp dụng

1. Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai trong DB.
2. "Bước sai đầu tiên" phải dẫn số event cụ thể trong trace, không phỏng đoán mơ hồ.
3. Case chỉ hoàn thành khi có regression test chạy được: grader `DbFactKept` hoặc item tương ứng trong eval set với provenance đầy đủ.
