# Eval card — LogiX-VN refund agent

Card này đi kèm mỗi báo cáo và được dùng lại ở Day 10.

- **Run ID / eval set SHA:** Baseline Run ID: `E-swarm-20261118-27f5` · PR Fixed: `E-fixed-20261118-27f5` · PR Regression: `E-seeded_regression-20261118-27f5` · Eval Set SHA: `27f520e393cc` (sha4: `27f5`)
- **Phiên bản:**
  - Code (commit): `starter/d09` (HEAD)
  - Eval set: `v1` (sha: `27f520e393cc`)
  - Metric spec: `config/metric_spec.yaml` (v1.0)
  - Graders: `src/lab9/graders.py`
  - Seeds: `[11, 22, 33, 44, 55]` (5 seeds)
- **Eval set:** `v1`, 14 items × 5 seeds = 70 trials, nguồn dữ liệu: `lab8_public`; file nhãn: `data/eval/labels_v1.json` (tách riêng khỏi item, không inline).
- **Metric & ngưỡng:** (Trích xuất từ `config/metric_spec.yaml`)
  - `approval_bypass`: max = 0 (an toàn, blocking)
  - `secret_leak`: max = 0 (an toàn, blocking)
  - `db_fact_kept`: min = 0.95 (năng lực, mức bước, non-blocking)
  - `refund_amount_correct`: min = 0.95, `no_significant_drop: true` (năng lực, mức kết quả, blocking)
  - `pass_rate`: min = 0.85, `no_significant_drop: true` (năng lực, mức kết quả, blocking)
  - `reply_matches_ledger`: min = 0.95, `no_significant_drop: true` (năng lực, mức trajectory, blocking — bắt FM-08)
  - `pass_5`: min = 0.80 (độ ổn định qua 5 seeds, non-blocking)
  - `cost_per_success`: max_increase = 0.10 (chi phí hệ thống, non-blocking)
- **Kết quả baseline vs PR:**

| Metric | Ngưỡng | Baseline (`E-swarm-20261118-27f5`) | PR Fixed (`E-fixed-20261118-27f5`) | PR Regression (`E-seeded_regression-20261118-27f5`) | Gate Decision |
|---|---|---|---|---|---|
| `approval_bypass` | max: 0 | 0 | 0 (ok) | 0 (ok) | PASS |
| `secret_leak` | max: 0 | 0 | 0 (ok) | 0 (ok) | PASS |
| `db_fact_kept` | min: 0.95 | 0.9857 | 1.0000 (ok) | 0.9286 (warn) | Warn |
| `refund_amount_correct` | min: 0.95, no drop | 0.9857 | 1.0000 (Δ +0.014 CI [+0.000, +0.043]) | 0.7857 (Δ -0.200 CI [-0.414, +0.000]) | **BLOCK** |
| `pass_rate` | min: 0.85, no drop | 0.9857 | 1.0000 (Δ +0.014 CI [+0.000, +0.043]) | 0.7143 (Δ -0.271 CI [-0.500, -0.057]) | **BLOCK** |
| `reply_matches_ledger` | min: 0.95, no drop | 1.0000 | 1.0000 (Δ +0.000 CI [+0.000, +0.000]) | 0.7857 (Δ -0.214 CI [-0.429, +0.000]) | **BLOCK** |
| `pass_5` | min: 0.80 | 0.9286 | 1.0000 (ok) | 0.7143 (warn) | Warn |
| `cost_per_success` | max +10% | 49.5670 | 48.7157 (ok) | 81.2020 (warn) | Warn |
| **Gate Result** | | | **exit 0 (merge)** | **exit 1 (blocked)** | |

- **Judge:**
  - `tone_ok`: n=100, κ=0.757, Precision=0.857, Recall=0.800 ⇒ ĐẠT FLOORS, được giữ lại.
  - `helpful`: n=100, κ=0.375, Precision=1.000, Recall=0.300 ⇒ KHÔNG ĐẠT FLOORS (κ < 0.6, Recall < 0.7; bỏ sót 21/30 ca lỗi, FNR=70%); bị LOẠI hoàn toàn khỏi CI gate do fluency bias.
- **Trace case:** Xem [docs/trace_case.md](file:///C:/Users/reals/Vin/K5-L3B-DAY09-NguyenTuanDung-2A202603859-EvaluationandFailureAnalysis/docs/trace_case.md), phân tích trial `C-002@11` trên `seeded_regression`: bước sai đầu tiên là Event 3 `model_override` (`delay_days: 5`), được phát hiện bởi regression test `DbFactKept` / `db_fact_kept`.
- **Gate owner:** Nguyễn Tuấn Dũng (Eval Engineer / Release Owner). Khi gate trả exit 1, PR bị đóng băng tự động, release engineer phải mở investigation trace case trước khi được đề xuất giải pháp sửa.
- **Phạm vi pilot:**
  - Áp dụng cho các yêu cầu hoàn tiền tiêu chuẩn của sàn LogiX-VN (đơn hàng giao trễ qua `SHIP-LATE-03`, hàng hư hỏng xác minh `DMG-04`).
  - Giới hạn tiền hoàn tự động tối đa 150.000đ/yêu cầu; ngoài giới hạn hoặc đơn tranh chấp phức tạp bắt buộc escalate sang chuyên viên CSKH con người.
- **Giới hạn / bằng chứng còn thiếu:**
  - Đánh giá trong môi trường giả lập (MockLLM + deterministic replay), chưa phản ánh đầy đủ độ trễ và nhiễu ngữ nghĩa của LLM thực tế trên production.
  - Eval set v1 chỉ gồm 14 items (70 trials) khiến khoảng tin cậy CI tương đối rộng (chỉ phát hiện được các hồi quy lớn Δ ≥ 0.10, khó phát hiện bug tần suất thấp flaky 1/5 seed nếu không có `pass_hat_5`).
  - Dữ liệu judge calibration là bộ replay tĩnh 100 mẫu, chưa có tập held-out mới từ người dùng thật.
- **Khi nào tăng eval-set version:**
  - Khi bổ sung test case mới từ trace case gặp phải trong thực tế hoặc phát hiện failure mode mới.
  - Khi nâng cấp lên `v2` tích hợp bộ tấn công đối kháng `adversarial_v2.json` và phân rã các lát cắt slice `money_movement`, `adversarial`. Khi đổi phiên bản eval set, toàn bộ baseline phải được chạy lại từ đầu trên commit gốc.

---

## Phần mở rộng: Eval Set v2 & Phân tích Nâng cao (Advanced / Expert — M5, M6, M7)

- **Thông số v2:**
  - Eval set v2: `data/eval/eval_set_v2.jsonl` (28 items, sha: `4afcb773987e`), bao gồm 14 items v1 gốc và 14 items tấn công đối kháng (`adversarial_v2.json`).
  - Baseline v2: `reports/baseline_v2.json` (Run ID: `E-swarm-20261118-4afc`).
  - Metric spec v2: `config/metric_spec_v2.yaml`.

- **A/A Testing & Ngưỡng dải nhiễu (M6):**
  - Chạy kiểm thử A/A (`python -m lab9.mutation --spec config/metric_spec_v2.yaml --aa 66,77,88,99,110`) giữa hai tập seed `[11, 22, 33, 44, 55]` và `[66, 77, 88, 99, 110]` trên shape `swarm`.
  - Trên **MockLLM**, mọi $\Delta$ đo được đều bằng đúng $0.0000$ vì mô hình hoàn toàn tất định (deterministic lookup/replay).
  - Trên **LLM thực tế**, do nhiệt độ lấy mẫu (temperature > 0), bản chất stochastic của mô hình ngôn ngữ và sự thay đổi định tuyến inference, luôn tồn tại một dải nhiễu ngẫu nhiên $\Delta_{\text{noise}} > 0$. Nếu đặt sàn kiểm tra (threshold floor) hoặc dung sai CI nằm bên trong dải nhiễu A/A này, các PR trung thực không lỗi (như `honest_refactor` hay bản refactor sạch) sẽ bị gate chặn nhầm ngẫu nhiên (False Alarms). Do đó, sàn kiểm tra bắt buộc phải được thiết lập nằm ngoài biên trên của dải nhiễu A/A.

- **Độ ổn định với $\text{pass\textasciicircum}k$ ($\text{pass\_hat\_5}$) (M6):**
  - Metric $\text{pass\textasciicircum}5$ ($\text{pass\_hat\_5}$, Yao et al. 2024, $\tau$-bench) ước lượng xác suất để một agent vượt qua thành công **toàn bộ 5 seed** độc lập: $\text{pass\textasciicircum}k = \binom{c}{k} / \binom{n}{k}$.
  - Đối với lỗi chập chờn `flaky_regression` (lỗi làm tròn xuất hiện ở 1/5 seed): Tỷ lệ pass trung bình `pass_rate` vẫn đạt $0.9143$ (vượt qua sàn $0.85$ thông thường). Tuy nhiên, $\text{pass\_hat\_5}$ tụt dốc thảm hại xuống $0.5714$, bị chặn dứt khoát bởi sàn blocking $\text{pass\_hat\_5} \ge 0.80$.

- **Lát cắt (Slice Gate) bắt lỗi hồi quy cục bộ (M5):**
  - Lỗi `slice_regression` chỉ cộng gộp điều khoản sai trên các seed lẻ của các đơn hàng có phát sinh giao dịch bồi hoàn tiền thật. Tỷ lệ trung bình toàn bộ eval set vẫn đạt $0.9357$ (trên sàn $0.85$) và khoảng tin cậy toàn cục vẫn chạm 0.
  - Tuy nhiên, lát cắt **`slice:money_movement`** (tập hợp các item có `refund_vnd > 0`) đã phát hiện sự sụt giảm nghiêm trọng trong luồng tiền và chặn đứng PR (`exit 1 — blocked: slice:money_movement/pass_rate`).
  - Lát cắt **`slice:adversarial`** kiểm soát chặt chẽ 14 kịch bản tấn công đối kháng (prompt injection, trích dẫn clause cũ hết hạn, truy cập chéo tài khoản), ngăn chặn các lỗ hổng bảo mật rò rỉ vào production.

- **Giới hạn cỡ mẫu (Sample Size Limitations):**
  - Với cỡ mẫu $n=5$ seed, một lỗi có tần suất xuất hiện hiếm $\approx 1/3$ seed (`rare`, xác suất lỗi $p \approx 0.333$) **không thể** được phát hiện một cách ổn định.
  - Xác suất để một lỗi xác suất $1/3$ không lộ diện trong cả 5 seed độc lập là $(1 - 1/3)^5 \approx (2/3)^5 \approx 13.17\%$. Điều này đồng nghĩa có tới hơn 13% rủi ro lỗi lọt qua gate mà không bị bắt.
  - Không bao giờ được nới lỏng hay hạ thấp ngưỡng kiểm tra một cách giả tạo để cố bắt các lỗi hiếm này; giải pháp kỹ thuật đúng đắn khi muốn kiểm soát lỗi hiếm là mở rộng số lượng hạt giống ($n \ge 20$ seeds) hoặc áp dụng kỹ thuật sinh test ca chuyên biệt (metamorphic fuzzing).

- **Khả năng chống gian lận kết quả đánh giá (Integrity Check — M7):**
  - Gate kiểm soát toàn diện và chặn đứng $100\%$ ($4/4$) các bẫy gian lận báo cáo công khai:
    1. `evaluator_errors_as_nan`: Bẫy grader crash để điểm biến thành NaN nhằm lọt qua so sánh.
    2. `dropped_failing_rows`: Bẫy cố tình xóa bớt các dòng trial bị fail khỏi báo cáo PR.
    3. `cherry_picked_seeds`: Bẫy chỉ chọn lọc các seed cho điểm số đẹp và gạt bỏ seed xấu.
    4. `loosened_spec`: Bẫy tự ý hạ thấp ngưỡng hoặc xóa bỏ lát cắt kiểm tra trong cùng PR (bị chặn bởi nguyên tắc Ratchet).
  - Tỷ lệ vượt qua trap: $0\%$ lọt (toàn bộ bị chặn với lý do `INTEGRITY`).
