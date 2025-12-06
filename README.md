# Macro Alert System v4.3+ (Academic Lite) — README

## 1) Mục tiêu & phạm vi
Macro Alert System là một chỉ báo Pine Script (TradingView) nhằm:
- Theo dõi trạng thái vĩ mô Việt Nam theo thời gian (ưu tiên khung D).
- Tạo cảnh báo dựa trên 4 “trụ” (pillars) rủi ro vĩ mô.
- Tổng hợp thành Risk Score và phân rã theo 3 lớp rủi ro (3-layer risk) để dễ diễn giải.

Hệ thống này là **chỉ số cảnh báo (warning index)** theo hướng *heuristic, explainable, human-in-the-loop*, không phải mô hình dự báo xác suất hay mô hình nhân quả.

---

## 2) Dữ liệu đầu vào (inputs) & ý nghĩa kinh tế
Hệ thống sử dụng 5 chuỗi dữ liệu vĩ mô:
- **VNINTR**: Lãi suất chính sách/điều hành (proxy).
- **VN02Y**: Lợi suất trái phiếu VN 2Y.
- **VN10Y**: Lợi suất trái phiếu VN 10Y.
- **US10Y**: Lợi suất trái phiếu Mỹ 10Y.
- **VNINBR**: Lãi suất liên ngân hàng VN.

**Chốt kỹ thuật học thuật quan trọng (reproducibility):**
- Tất cả dữ liệu vĩ mô được lấy theo `tf_macro` (mặc định **D**) để tránh “đổi timeframe = đổi mô hình”.

---

## 3) Các biến dẫn xuất (spreads) & quy ước dấu (sign convention)
Hệ thống tạo 4 biến chính:

1) `liquidity_stress = interbank_rate - policy_rate`
- **CAO = XẤU**: liên ngân hàng cao hơn nhiều so với policy → stress thanh khoản/funding.

2) `yield_curve_standard = bond_10y - bond_2y`
- **THẤP = XẤU**: đường cong phẳng/đảo → tín hiệu chu kỳ suy giảm, kỳ vọng tăng trưởng yếu.

3) `intl_yield_diff = bond_10y - us_10y`
- **THẤP = XẤU**: chênh lệch VN–US thu hẹp → áp lực vốn/FX/điều kiện tài chính.

4) `long_short_spread = bond_10y - policy_rate`
- **THẤP = XẤU**: chênh lệch dài–ngắn thu hẹp → điều kiện chính sách/chi phí vốn tương đối “căng”.

Quy ước dấu được duy trì nhất quán xuyên suốt dashboard và “risk-side percentile”.

---

## 4) Nền tảng học thuật cốt lõi (Academic foundations)

### 4.1 Signal Detection: Sensitivity vs Specificity
Mọi hệ thống cảnh báo đều có trade-off:
- **Aggressive**: nhạy (high sensitivity) → báo sớm nhưng nhiều false alarm.
- **Conservative**: chắc (high specificity) → ít báo nhưng báo thường “nặng”.
- **Balanced**: trung dung.

Vì dữ liệu vĩ mô có “regime shift”, hệ thống ưu tiên cách tiếp cận có thể **calibrate** thay vì tối ưu hoá tự động kiểu black-box.

---

### 4.2 Percentile-based thresholds (ngưỡng theo phân phối)
Thay vì dùng ngưỡng cố định, hệ thống dùng **percentile** trên cửa sổ lookback:
- Stress: dùng ngưỡng ở **đuôi cao** (ví dụ 85th/90th) vì **cao là xấu**.
- Curve / Intl / Spread: dùng ngưỡng ở **đuôi thấp** (ví dụ 15th/10th) vì **thấp là xấu**.

Lý do học thuật:
- Percentile là cách “phi tham số” (non-parametric), bền hơn khi phân phối **không chuẩn** và có **thay đổi chế độ**.

---

### 4.3 Z-score & Robust z-score (winsorization)
Hệ thống hỗ trợ:
- **Z-score thường**: chuẩn hoá bằng mean/std.
- **Robust z-score (winsorized)**: cắt outliers theo `clip_multiplier * std` rồi mới tính thống kê lại.

Ý nghĩa học thuật:
- Winsorization giảm ảnh hưởng của điểm dị thường (outlier), giúp z-score ổn định hơn trong dữ liệu tài chính/vĩ mô vốn hay có “spike”.

Lưu ý: đây là robust theo hướng “pragmatic in Pine”, không phải MAD-based z-score chuẩn textbook, nhưng đủ tốt cho môi trường TradingView.

---

### 4.4 Percent-rank (percentrank) & “risk-side normalization”
Hệ thống hiển thị percentrank hiện tại để biết vị trí của biến trong phân phối lịch sử.
Để thống nhất “cao = rủi ro cao”:
- Với biến **THẤP = XẤU** dùng: `risk_pct = 100 - percentrank`.
- Với biến **CAO = XẤU** dùng: `risk_pct = percentrank`.

Đây là nền tảng quan trọng cho tính **interpretability** (dễ đọc, ít nhầm dấu).

---

## 5) Kiến trúc cảnh báo: 4 trụ + 3 lớp rủi ro

### 5.1 4 pillars (trạng thái nhị phân)
Mỗi trụ tạo 1 cờ (flag):
- `stress_high`
- `curve_inversion`
- `intl_warning`
- `spread_warning`

Flag được kích hoạt theo 1 trong 3 chế độ threshold:
- Percentile-based (khuyến nghị)
- Dynamic (z-score)
- Static

---

### 5.2 3-layer risk decomposition (không double-counting)
Hệ thống chia rủi ro thành 3 lớp:
- **Layer 1 — Funding/Liquidity**: chỉ dùng `stress_high` (tránh double-counting).
- **Layer 2 — Cycle/Macro**: yield curve + long-short spread.
- **Layer 3 — External**: intl diff + (tùy chọn logic) kết hợp stress & intl để phản ánh áp lực ngoại vi khi funding cũng căng.

Mục tiêu học thuật: tăng tính “structural interpretability” (người dùng hiểu rủi ro đến từ đâu).

---

## 6) Tổng hợp Risk Score (heuristic, explainable)
Risk Score = tổng trọng số của các flags (ordinal scoring):
- `risk_score = Σ wi * 1(flag_i)`
- Chuẩn hoá ra `risk_pct` để hiển thị 0–100.

Đây là **heuristic index**, không phải xác suất suy thoái hay mô hình nhân quả.
Ưu điểm: minh bạch, kiểm soát được, dễ audit và phù hợp thị trường có “regime change”.

---

## 7) Macro Scenarios (tổ hợp điều kiện)
Một số kịch bản được gợi ý bằng tổ hợp flags:
- Liquidity crisis: stress + curve + intl
- Recession warning: curve + spread
- FX pressure: intl + stress
- Inflation proxy: dựa trên dạng spread/cầu lãi suất (chỉ là proxy)

Các scenario nhằm mục đích diễn giải nhanh, không phải kết luận kinh tế vĩ mô chắc chắn.

---

## 8) Calibration (quan trọng hơn nâng cấp code)

### 8.1 Tại sao cần calibration?
Calibrate để “tính cách cảnh báo” phù hợp:
- mục tiêu (báo sớm vs báo chắc),
- đặc thù dữ liệu VN,
- khẩu vị rủi ro của người dùng.

### 8.2 Nguyên tắc calibration
- Chọn 1 preset (Aggressive / Balanced / Conservative hoặc VN baseline).
- Chạy 1–2 tuần, ghi log cảnh báo.
- Mỗi lần chỉ chỉnh 1–2 tham số (để biết nguyên nhân).
- Định kỳ review theo quý hoặc khi regime đổi.

---

## 9) Ứng dụng thực tế (workflow)
Gợi ý cách dùng:
1) Mở chart Daily, chọn `tf_macro = D`.
2) Luân phiên xem 4 pane:
   - Pane 1: quan sát rates.
   - Pane 2: spreads/z-scores/threshold deviation.
   - Pane 3: dashboard + calibration table (explainability).
   - Pane 4: xem rủi ro theo lớp (layering).
3) Dùng alert khi Risk Score hoặc layer vượt ngưỡng.
4) Calibration theo feedback loop.

---

## 10) Hạn chế & lưu ý học thuật
- Dữ liệu vĩ mô có thể có ngày không cập nhật; việc “giữ giá” có thể làm phân phối bị dồn cụm.
- Các biến dùng chung VN10Y có “shared factor” (cộng hưởng), Risk Score có thể tăng mạnh khi VN10Y biến động lớn.
- Hệ thống là cảnh báo định tính/định lượng nhẹ, không thay thế phân tích vĩ mô đầy đủ hay dự báo.

---

## 11) Tài liệu tham khảo (gợi ý học thuật)
- Signal Detection Theory: sensitivity/specificity trade-off.
- Robust Statistics: winsorization, outlier handling.
- Quantile/Percentile methods: non-parametric thresholds trong dữ liệu không chuẩn.
- Financial conditions & yield curve literature (macro finance).

---

## 12) License / Disclaimer
Chỉ báo phục vụ nghiên cứu & hỗ trợ ra quyết định; không phải khuyến nghị đầu tư.
Người dùng tự chịu trách nhiệm khi sử dụng.
