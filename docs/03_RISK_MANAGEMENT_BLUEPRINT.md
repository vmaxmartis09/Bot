# 03 — Risk Management Blueprint cho XAUUSD

> Số liệu cụ thể, không lý thuyết suông. XAUUSD = biến động cao → risk management **là**
> chiến lược, không phải phụ kiện. Grid/DCA mà không có cái này = bom hẹn giờ.

---

## 1. Đặc tính XAUUSD cần nhớ

| Đặc tính | Giá trị tham khảo | Hệ quả cho bot |
|----------|-------------------|----------------|
| ADR (Average Daily Range) | ~$20–35 (giai đoạn thường), $50+ khi biến động | SL/TP cố định nhỏ là vô nghĩa → dùng ATR |
| Spread | 15–30 points thường, 100–500 khi tin | Bắt buộc spread filter (V9) |
| Phản ứng tin | NFP/CPI/FOMC nhảy $10–40 trong giây | Bắt buộc news filter (V1) |
| Gap cuối tuần | Thường xuyên, $5–20 | Đóng trước cuối tuần (V11) |
| Tương quan | Nghịch USD/lợi suất thực; thuận rủi ro địa chính trị | Bias engine nên xét DXY/yields |
| Giá trị 1 lot | ~$100 / $1 di chuyển (100oz) | 0.01 lot ≈ $1 / $1 move → tính risk cẩn thận |

> **Quy đổi point:** kiểm tra `_Digits`, `_Point`, `SYMBOL_TRADE_TICK_VALUE` trên đúng sàn.
> Đừng giả định "2 = $2" hay "100 = $100" (V7). Tính bằng `SYMBOL_TRADE_TICK_VALUE`.

---

## 2. Position sizing — thay volume hardcode bằng % risk

Code hiện tại dùng bảng volume bell-curve **cố định** (0.03…0.16). Vấn đề: lot cố định không
co giãn theo equity → tài khoản nhỏ cháy, tài khoản lớn dùng dưới mức.

**Công thức risk-based:**
```
RiskMoney   = Equity × RiskPerTradePct%
SL_points   = khoảng cách entry→SL (theo point)
LotForRisk  = RiskMoney / (SL_points × TickValuePerPoint)
```
```mql5
double CalcLotByRisk(double riskPct, double slPoints) {
   double risk = AccountInfoDouble(ACCOUNT_EQUITY) * riskPct / 100.0;
   double tickVal  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   double valuePerPoint = tickVal * (_Point / tickSize);
   double lot = risk / (slPoints * valuePerPoint);
   return NormalizeLot(lot);
}
```
**Giữ hình dạng bell-curve** nhưng coi nó là **trọng số phân bổ**, không phải lot tuyệt đối:
```
lot_i = TotalBasketLot × weight_i / Σweight
```
→ tổng risk cụm được kiểm soát, hình dạng phân bổ giữ nguyên ý đồ ban đầu.

---

## 3. Giới hạn rủi ro cụm (basket) — quan trọng nhất với grid

Grid nguy hiểm vì **risk thật = tổng tất cả level cộng lại**, không phải từng lệnh. Phải đặt trần:

| Tham số | Giá trị đề xuất | Lý do |
|---------|-----------------|-------|
| `MaxBasketLot` | ≤ Equity-dựa, ví dụ 0.01 lot / $1000 equity / mỗi $1 SL | Trần exposure tuyệt đối |
| `MaxOpenLevels` | 19 (giữ) nhưng + trần tổng lot | Giới hạn số entry |
| Risk cả cụm | ≤ 1–2% equity nếu basket SL hit | Một ngày tệ không > 2% |
| `MaxDailyLossPct` | 3% | Dừng ngày khi chạm |
| `MaxTotalDDPct` | 15% | Hard stop, tắt bot, alert |
| Max trade/ngày | 1 cụm/ngày (đúng thiết kế Daily Bias) | Tránh overtrading |

**Basket TP/SL (thay TP +2 / SL −20 vô lý):**
```
AvgEntry   = Σ(price_i × lot_i) / Σlot_i        // giá vào trung bình cả cụm
BasketTP   = AvgEntry + InpBasketTP_ATR × ATR    // chốt cả cụm khi đạt
BasketSL   = AvgEntry − InpBasketSL_ATR × ATR    // cắt cả cụm
```
Quản lý ở `ManageBasketExit()` chạy mỗi tick/timer: tính floating P/L cả cụm, đóng hết khi
chạm ngưỡng tiền hoặc ATR. Đây là cách grid chuyên nghiệp thoát lệnh.

---

## 4. Kelly & fractional Kelly (sizing nâng cao)

Sau khi có thống kê backtest (win-rate `W`, payoff `R = avgWin/avgLoss`):
```
KellyFraction = W − (1 − W) / R
```
- Kelly thuần thường quá hung hãn → dùng **¼ Kelly** (`0.25 × KellyFraction`).
- Với grid negative-skew, R thường < 1 và W cao → Kelly có thể ra số nhỏ/âm ⇒ **cảnh báo
  chiến lược biên mỏng**. Nếu Kelly ≤ 0 → chiến lược không có lợi thế, đừng tăng size.
- Dùng Kelly như **trần trên** cho `RiskPerTradePct`, không phải mục tiêu.

---

## 5. Circuit breaker — máy ngắt nhiều tầng

```
Tầng 1  Spread > MaxSpread          → hoãn đặt lệnh mới (tạm thời)
Tầng 2  News blackout               → hoãn + (tùy chọn) flatten
Tầng 3  Daily loss ≥ MaxDailyLossPct→ đóng hết, ngừng đến hết ngày
Tầng 4  Total DD ≥ MaxTotalDDPct    → đóng hết, TẮT bot, gửi alert
Tầng 5  Consecutive losses ≥ N      → giảm size 50% hoặc tạm dừng K ngày
```
```mql5
bool RiskGateOpen() {
   double eq = AccountInfoDouble(ACCOUNT_EQUITY);
   double dayLoss = (g_dayStartEquity - eq) / g_dayStartEquity * 100.0;
   if (dayLoss >= InpMaxDailyLossPct) return Halt("DAILY_LOSS");
   double bal = AccountInfoDouble(ACCOUNT_BALANCE);
   if ((bal - eq) / bal * 100.0 >= InpMaxTotalDDPct) return Halt("MAX_DD");
   if (g_consecutiveLosses >= InpMaxConsecLoss) return Halt("LOSS_STREAK");
   return true;
}
```
`g_dayStartEquity` set khi sang ngày mới (trong OnTimer khi đổi `today`).

---

## 6. Bảng tham số khởi điểm (conservative)

| Input | Demo/test | Live thận trọng |
|-------|-----------|-----------------|
| `RiskPerTradePct` (cả cụm) | 1.0% | 0.5% |
| `MaxDailyLossPct` | 5% | 3% |
| `MaxTotalDDPct` | 20% | 12–15% |
| `MaxSpreadPoints` | 60 | 40 |
| `BasketTP_ATR` | 1.0 | 1.0 |
| `BasketSL_ATR` | 2.0 | 1.5 |
| `NewsBefore/After` (min) | 30 / 30 | 45 / 45 |
| Số cụm/ngày | 1 | 1 |

> Đây là **điểm khởi đầu để backtest**, không phải tham số tối ưu. Tối ưu bằng walk-forward
> để tránh overfit — xem [05_MARKET_BOT_TECHNIQUES.md](05_MARKET_BOT_TECHNIQUES.md) §validation.

---

## 7. Kỷ luật vận hành (không phải code)

1. **Demo ≥ 1 tháng** với điều kiện thật trước khi live.
2. Live bắt đầu **lot tối thiểu** bất kể backtest đẹp cỡ nào.
3. Theo dõi **drawdown thật vs backtest** — lệch lớn = regime đã đổi, dừng review.
4. Không bao giờ tắt circuit breaker để "gỡ".
5. Backtest phải gồm các giai đoạn stress: 2020-03 (COVID), 2022–2023 (Fed tăng lãi), 2024 (ATH gold).
