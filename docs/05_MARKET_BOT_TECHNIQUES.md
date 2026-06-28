# 05 — Kỹ thuật & chiến lược các bot khác trên thị trường

> Khảo sát rộng để mở tầm nhìn: các họ chiến lược, kỹ thuật thực thi, và kỹ thuật quản trị
> rủi ro mà bot trading chuyên nghiệp đang dùng. Đối chiếu với bot Daily-Bias-Grid hiện tại.

---

## 1. Bản đồ các họ chiến lược

| Họ | Ý tưởng cốt lõi | Phù hợp thị trường | Rủi ro đặc trưng | Liên quan bot này |
|----|-----------------|--------------------|-------------------|-------------------|
| **Grid** | Rải lệnh đều theo lưới giá | Sideway / range | Trend mạnh phá lưới | ✅ bot đang dùng |
| **Martingale / anti** | Tăng/giảm size sau thua | Mean-reversion | Cháy theo cấp số nhân | ⚠️ họ hàng gần |
| **DCA / averaging** | Vào thêm khi giá ngược | Tin vào hồi quy | Adverse exposure lớn | ✅ bot đang dùng |
| **Trend following** | Đi theo xu hướng | Trend | Whipsaw lúc sideway | 🔄 nên bổ sung làm bias |
| **Mean reversion** | Mua thấp bán cao quanh trung bình | Range | Trend phá vỡ | 🔄 |
| **Breakout / ORB** | Vào khi phá biên/range phiên | Volatility expansion | False breakout | 🔄 |
| **Smart Money (ICT/SMC)** | Order block, FVG, liquidity sweep | Mọi TF | Chủ quan, khó định lượng | 🔄 nguồn bias tốt |
| **Stat-arb / pairs** | Giao dịch chênh lệch cặp tương quan | Cặp đồng tích hợp | Tương quan gãy | ❌ ngoài phạm vi |
| **Market making** | Đặt 2 chiều ăn spread | Thanh khoản cao | Inventory risk, adverse selection | ❌ |
| **News straddle** | Đặt 2 chiều quanh tin | Quanh tin lớn | Slippage, requote | ⚠️ ngược với news filter |
| **ML-based** | Dự đoán từ feature | Dữ liệu nhiều | Overfit, regime shift | 🔄 dài hạn |
| **Portfolio of edges** | Kết hợp nhiều chiến lược ít tương quan | Mọi loại | Phức tạp vận hành | 🎯 đích đến |

Ghi chú: ✅ đang dùng · ⚠️ họ hàng/cẩn trọng · 🔄 nên bổ sung · 🎯 mục tiêu dài hạn · ❌ ngoài phạm vi.

---

## 2. Chiến lược chi tiết đáng học cho XAUUSD

### 2.1 Trend following (bổ sung cho Bias Engine)
- **MA cross / EMA stack** (EMA 20/50/200) — xác định hướng ngày.
- **Donchian / Turtle** — breakout kênh N ngày.
- **Supertrend, Ichimoku Kumo** — lọc hướng + vùng cản động.
- **ADX > 25** để xác nhận có trend (tránh grid trong trend mạnh).
> Dùng các tín hiệu này làm `DetermineDailyBias()` thay cho hardcode BUY (V4), và làm
> **bộ lọc tắt grid** khi ADX cao (trend phá lưới).

### 2.2 Breakout / Opening Range Breakout (ORB)
- Lấy range phiên Á (00:00–07:00) hoặc 15–30 phút đầu phiên London/NY.
- Vào khi phá biên + retest. Rất hợp đặc tính "vàng bùng nổ theo phiên".
- Có thể thay cơ chế "07:00 fire cứng" bằng "07:00 chốt range → chờ breakout".

### 2.3 Smart Money Concepts (ICT) — nguồn bias chất lượng
- **Liquidity sweep**: giá quét đỉnh/đáy cũ rồi đảo → entry ngược.
- **Order block / FVG (Fair Value Gap)**: vùng mất cân bằng để canh entry.
- **BOS / CHoCH**: break of structure / change of character xác định đảo chiều.
> Định lượng hóa khó nhưng cho **điểm vào đẹp** — phù hợp triết lý "stop-order confirmation"
> sẵn có của bot (chờ xác nhận hồi rồi vào).

### 2.4 Mean reversion có kỷ luật
- Bollinger Band + RSI phân kỳ; VWAP deviation.
- Khác grid mù: chỉ vào khi **giá lệch xa trung bình + có tín hiệu cạn lực**, có SL rõ.
- Đây là phiên bản "có não" của averaging-down mà bot đang làm.

### 2.5 Volatility regime switching
- Đo ATR/realized vol → **chọn chế độ**: vol thấp → grid/mean-reversion; vol cao → tắt grid,
  chuyển trend/breakout hoặc nghỉ.
- Giải quyết gốc rễ điểm yếu của grid: grid chỉ an toàn ở regime range.

---

## 3. Kỹ thuật thực thi (execution) chuyên sâu

| Kỹ thuật | Mục đích | Áp dụng cho bot |
|----------|----------|-----------------|
| **Slippage control / max deviation** | Tránh khớp giá tệ | `trade.SetDeviationInPoints` (V16) |
| **Spread-aware entry** | Không vào khi spread giãn | spread filter (V9) |
| **Partial fill handling** | Đồng bộ size thực | đối chiếu `POSITION_VOLUME` (V17) |
| **TWAP / VWAP slicing** | Vào lệnh lớn không gây impact | ít liên quan retail XAUUSD, hữu ích lot lớn |
| **Iceberg / hidden size** | Giấu khối lượng | sàn ECN |
| **Latency / retry với backoff** | Xử lý requote, off-quotes | bọc `PlaceOrder` retry có giới hạn |
| **Order throttling** | Tránh spam server (rate limit) | giới hạn số lệnh/giây |

---

## 4. Kỹ thuật quản trị rủi ro nâng cao (các quỹ/bot dùng)

| Kỹ thuật | Ý nghĩa | Tham khảo |
|----------|---------|-----------|
| **Fractional Kelly** | Sizing tối ưu tăng trưởng, dùng ¼ Kelly | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §4 |
| **Volatility targeting** | Điều chỉnh size để giữ vol danh mục cố định | size ∝ 1/ATR |
| **CPPI / risk budgeting** | Phân bổ risk theo "ngân sách" | nâng cao |
| **Max DD circuit breaker** | Ngắt khi thua tới ngưỡng | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §5 |
| **Correlation cap** | Không mở nhiều vị thế tương quan cao | nếu mở rộng đa cặp |
| **Time-based stop** | Đóng nếu lệnh "ì" quá lâu không đạt | thêm `MaxHoldHours` |

---

## 5. Validation — chống tự lừa dối (cực kỳ quan trọng)

> Phần lớn bot "đẹp trên backtest" cháy live vì **overfit**. Đây là quy trình kiểm chứng chuẩn:

| Bước | Mô tả | Cạm bẫy tránh |
|------|-------|----------------|
| **In-sample / Out-of-sample split** | Tối ưu trên IS, kiểm trên OOS chưa từng thấy | Tối ưu trên toàn bộ data |
| **Walk-forward analysis** | Trượt cửa sổ optimize→test lặp lại | Một lần optimize tĩnh |
| **Monte Carlo** | Xáo trộn thứ tự trade → phân phối drawdown | Chỉ nhìn 1 equity curve |
| **Stress test regime** | Test riêng 2020/2022/2023/2024 | Chỉ test giai đoạn thuận |
| **Realistic costs** | Spread thật + commission + slippage + swap | Spread cố định 0 |
| **Parameter sensitivity** | Đổi nhẹ tham số, kết quả có sụp không | "Đỉnh nhọn" = overfit |
| **Tick-data + real ticks** | MT5 "Every tick based on real ticks" | "Open prices only" cho grid = ảo |

**Quy tắc:** chiến lược tốt là **bền vững quanh vùng tham số**, không phải đỉnh nhọn của một bộ số.

---

## 6. Đối chiếu: bot hiện tại đứng ở đâu

| Tiêu chí | Bot hiện tại | Chuẩn production |
|----------|--------------|------------------|
| Edge / nguồn bias | ❌ hardcode | ✅ trend/SMC/regime |
| Risk sizing | ⚠️ lot cố định | ✅ % risk / vol target |
| Basket exit | ⚠️ TP/SL từng lệnh | ✅ basket ATR |
| News handling | ❌ | ✅ filter + action |
| Regime awareness | ❌ grid mọi lúc | ✅ chỉ range / vol thấp |
| State persistence | ❌ | ✅ |
| Validation | ❓ chưa rõ | ✅ walk-forward + MC |
| Execution guards | ❌ | ✅ spread/slippage/retry |

→ Lộ trình thu hẹp khoảng cách: [06_ROADMAP.md](06_ROADMAP.md).

---

## 7. Đọc thêm / từ khóa nghiên cứu

- "Grid trading risk of ruin", "martingale expectancy gold"
- "ATR position sizing", "volatility targeting strategy"
- "Walk forward optimization MetaTrader", "Monte Carlo trade simulation"
- "ICT order block FVG strategy", "opening range breakout XAUUSD"
- "MQL5 CalendarValueHistory news filter"
- Sách: *Building Winning Algorithmic Trading Systems* (Davey), *Trading Systems* (Tomasini),
  *Advances in Financial Machine Learning* (López de Prado — phần overfitting/CSCV).
