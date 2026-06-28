# 01 — Đánh giá rủi ro & lỗ hổng + cách khắc phục

> Bảng lỗ hổng xếp theo severity. Mỗi mục: **Triệu chứng → Tác động → Cách khắc phục (code)**.
> Severity: 🔴 Critical (có thể cháy tài khoản) · 🟠 High · 🟡 Medium · ⚪ Low/cleanliness.

---

## Bảng tổng hợp (đọc nhanh)

| # | Lỗ hổng | Severity | Có trong |
|---|---------|----------|----------|
| V1 | Không có news filter | 🔴 | Cả 2 |
| V2 | Không có equity circuit breaker / max daily loss | 🔴 | Cả 2 |
| V3 | R:R âm (TP nhỏ, SL lớn) — kỳ vọng toán học âm | 🔴 | Cả 2 |
| V4 | Bias hardcode BUY — không có logic xác định hướng | 🔴 | Cả 2 |
| V5 | State chỉ in-memory → mồ côi vị thế khi restart | 🔴 | Cả 2 |
| V6 | Không magic number → lẫn lệnh tay/EA khác | 🟠 | Cả 2 |
| V7 | Giả định digits/point sai (NormalizeDouble 3) | 🟠 | Cả 2 |
| V8 | Không validate lot (min/max/step) + free margin | 🟠 | Cả 2 |
| V9 | Không có spread filter / slippage guard | 🟠 | Cả 2 |
| V10 | Timer khớp giây tuyệt đối → fragile | 🟠 | Cả 2 |
| V11 | Không đóng lệnh cuối ngày / trước cuối tuần | 🟠 | Cả 2 |
| V12 | BiasBot cộng dồn volume sai (gồm cả SKIP) | 🟠 | BiasBot |
| V13 | Không tự reset khi TP/SL (kẹt `dailyBiasRuning`) | 🟠 | BiasBot |
| V14 | NULL dùng làm sentinel cho index int | 🟡 | MoneyBot |
| V15 | Vòng lặp bỏ sót phần tử cuối (`i < Total()-1`) | 🟡 | MoneyBot |
| V16 | Market order truyền giá cố định → requote | 🟡 | Cả 2 |
| V17 | Không xử lý partial fill | 🟡 | Cả 2 |
| V18 | `CloseByTicket` nhầm order ticket vs position ticket | 🟡 | Cả 2 |
| V19 | Hằng số ma thuật (magic numbers) rải khắp code | ⚪ | Cả 2 |

---

## 🔴 CRITICAL — vá trước khi chạy live

### V1 — Không có news filter
**Triệu chứng:** bot fire 07:00 và nhồi lệnh bất kể lịch tin. XAUUSD phản ứng cực mạnh với
NFP, CPI, FOMC, PCE — giá nhảy $20–50 trong vài giây, spread giãn 10×.
**Tác động:** averaging-down vào giữa cú spike = SL toàn cụm tại điểm tệ nhất → cháy.
**Khắc phục:** chặn mở lệnh ± cửa sổ quanh tin high-impact USD/XAU. Toàn bộ ở
[04_NEWS_FILTER.md](04_NEWS_FILTER.md).

### V2 — Không có equity circuit breaker
**Triệu chứng:** không có biến nào theo dõi tổng lỗ ngày / drawdown. Bot nhồi lệnh vô hạn theo thang.
**Tác động:** một ngày trend mạnh = toàn bộ 19 level khớp + SL = mất nhiều % equity một lúc.
**Khắc phục (sườn code):**
```mql5
input double InpMaxDailyLossPct = 3.0;   // % equity tối đa mất trong ngày
input double InpMaxTotalDDPct   = 15.0;  // hard stop tổng

double g_dayStartEquity = 0;

bool RiskGateOpen() {
   double eq = AccountInfoDouble(ACCOUNT_EQUITY);
   double bal = AccountInfoDouble(ACCOUNT_BALANCE);
   double dayLossPct = (g_dayStartEquity - eq) / g_dayStartEquity * 100.0;
   if (dayLossPct >= InpMaxDailyLossPct) { CloseAllAndHalt("daily loss"); return false; }
   if ((bal - eq) / bal * 100.0 >= InpMaxTotalDDPct) { CloseAllAndHalt("max DD"); return false; }
   return true;
}
```
Gọi `RiskGateOpen()` đầu mỗi `OnTimer` trước khi `startDailyBias`/`scanDailyBias`. Chi tiết số liệu: [03_RISK_MANAGEMENT_BLUEPRINT.md](03_RISK_MANAGEMENT_BLUEPRINT.md).

### V3 — R:R âm
**Triệu chứng:** MoneyBot `TP = price + 2`, `SL = price - 20` → R:R = 1:10 (ngược).
BiasBot SL `−100` còn tệ hơn.
**Tác động:** cần **win-rate > 91%** chỉ để hòa vốn (chưa tính spread/commission). Một chuỗi
thua hiếm sẽ xóa hàng trăm phiên thắng. Đây là bản chất "negative skew" của grid.
**Khắc phục:**
- Chấp nhận negative skew **có ý thức** → bắt buộc đi kèm V2 (circuit breaker) để giới hạn cú thua.
- Hoặc đổi sang TP/SL theo **ATR** (ví dụ TP = 1.0×ATR, SL = 1.5×ATR) thay vì điểm cố định.
- Tính lại TP/SL cho **cả cụm** (basket TP) thay vì từng lệnh — xem blueprint §basket-exit.

### V4 — Bias hardcode BUY
**Triệu chứng:** `ENUM_ORDER_TYPE orderTypeDailyBias = ORDER_TYPE_BUY;` cố định, comment ghi
"gọi hàm check buy/sell" nhưng **hàm đó không tồn tại**.
**Tác động:** ngày downtrend bot vẫn BUY và averaging-down vào trend giảm = kịch bản cháy điển hình.
**Khắc phục:** viết `BiasEngine` xác định hướng ngày, ví dụ kết hợp:
```mql5
ENUM_ORDER_TYPE DetermineDailyBias() {
   // 1. Cấu trúc D1: đóng cửa hôm qua so với EMA200 D1
   double ema = iMA(_Symbol, PERIOD_D1, 200, 0, MODE_EMA, PRICE_CLOSE); // qua handle
   double prevClose = iClose(_Symbol, PERIOD_D1, 1);
   // 2. Asian range breakout / prev day high-low
   // 3. Trả về BUY nếu cấu trúc tăng, SELL nếu giảm, NONE nếu sideway → không trade
   return (prevClose > ema) ? ORDER_TYPE_BUY : ORDER_TYPE_SELL;
}
```
Gợi ý nguồn bias: EMA D1/H4, prev-day H/L, Asian session range, ADR vị trí. Xem [05](05_MARKET_BOT_TECHNIQUES.md) §trend/SMC.

### V5 — State chỉ in-memory
**Triệu chứng:** `m_tickets[]` / `priceVolumeList` mất khi EA reload, đổi timeframe, MT5 restart,
mất điện. Nhưng **lệnh vẫn nằm trên sàn**.
**Tác động:** bot "quên" các vị thế đang mở → vị thế mồ côi, không được quản lý SL/TP/đóng → rủi ro không kiểm soát.
**Khắc phục:**
- Persist state ra `GlobalVariableSet` hoặc file (`FileWrite` CSV/JSON) sau mỗi thay đổi.
- `OnInit` → **rebuild state từ sàn**: quét `PositionsTotal()` + `OrdersTotal()` lọc theo magic number, dựng lại `priceVolumeList`.
- Đây là pattern bắt buộc của mọi EA production. Chi tiết: [02_CODE_AUDIT.md](02_CODE_AUDIT.md) §StateStore.

---

## 🟠 HIGH

### V6 — Không magic number
`CTrade trade;` mặc định magic = 0. Không phân biệt lệnh bot vs tay vs EA khác.
**Khắc phục:** `trade.SetExpertMagicNumber(InpMagic);` trong `OnInit`. Mọi vòng lặp quét vị thế
phải lọc `PositionGetInteger(POSITION_MAGIC) == InpMagic`.

### V7 — Giả định digits sai
`NormalizeDouble(price, 3)` cứng 3 chữ số. Sàn 2-digit XAUUSD → giá sai 10×, SL/TP lệch.
**Khắc phục:**
```mql5
double Norm(double p) { return NormalizeDouble(p, (int)_Digits); }
double Pt() { return _Point; }            // dùng _Point thay cho hằng số
// "jump 2 points" => 2 * _Point * hệ số, đừng coi 2 = 2 USD
```
Quy đổi mọi "khoảng cách" (jump, SL, TP) sang đơn vị **point** qua `_Point`, không hardcode.

### V8 — Không validate lot + margin
**Khắc phục:**
```mql5
double NormalizeLot(double lot) {
   double mn = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double mx = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double st = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   lot = MathMax(mn, MathMin(mx, MathRound(lot/st)*st));
   return lot;
}
// trước khi đặt: kiểm tra OrderCalcMargin <= AccountInfoDouble(ACCOUNT_FREEMARGIN)
```

### V9 — Không spread filter / slippage
Giờ tin spread XAUUSD giãn 50–500 points.
**Khắc phục:** `if (SpreadPoints() > InpMaxSpread) return;` trước khi đặt; set `trade.SetDeviationInPoints()`.

### V10 — Timer fragile
`dt.sec == 0` khớp tuyệt đối; nếu OnTimer lệch (CPU bận, gap server) → cả ngày không start.
**Khắc phục:** dùng cờ "đã chạy hôm nay" theo **ngày**, kích hoạt khi `now >= mốc 07:00` lần đầu trong ngày:
```mql5
datetime g_lastBiasDay = 0;
datetime today = (datetime)(now - now % 86400);
if (now >= TodayAt(7,0) && g_lastBiasDay != today && !dailyBiasRuning) {
   g_lastBiasDay = today; startDailyBias();
}
```

### V11 — Không đóng cuối ngày / cuối tuần
Grid để qua đêm/cuối tuần → swap âm + gap thứ Hai (gold gap mạnh).
**Khắc phục:** input `InpCloseBeforeRollover`, `InpCloseFridayHour`; đóng toàn bộ + hủy pending trước rollover/cuối tuần.

### V12 — BiasBot cộng dồn volume sai
```mql5
totalVolume = totalVolume + ticketInfo.volume;  // cộng cả level đã SKIP/ACTIVE
```
`totalVolume` cộng **mọi** level kể cả đã SKIP → khối lượng buy-stop phình to ngoài ý muốn.
**Khắc phục:** chỉ cộng level còn hiệu lực, hoặc tính `basket` rõ ràng; clamp bằng `NormalizeLot` + trần `InpMaxBasketLot`.

### V13 — BiasBot không tự reset
Không có `OnTradeTransaction` → sau TP/SL `dailyBiasRuning` vẫn = 1 → hôm sau **không start lại**.
Ngoài ra `m_tickets.Size()` — `m_tickets` là `string[]` thường, **không có method `.Size()`** (phải `ArraySize()`), nghi vấn không compile như đang viết.
**Khắc phục:** port `OnTradeTransaction` từ MoneyBot; thay `.Size()` → `ArraySize()`.

---

## 🟡 MEDIUM

### V14 — NULL làm sentinel index
`int indexVolumeByPrice = NULL;` rồi `if (indexVolumeByPrice != NULL)`. `NULL == 0` mà 0 là index hợp lệ.
**Khắc phục:** dùng `int idx = -1;` + `if (idx >= 0)`.

### V15 — Bỏ sót phần tử cuối
`for (i=1; i < Total()-1; i++)` → level cuối không bao giờ được xét.
**Khắc phục:** rà lại biên vòng lặp; nếu cần so `pv2 = At(i+1)` thì guard `i+1 < Total()`.

### V16 — Market order truyền giá cố định
`trade.Buy(volume, _Symbol, price, ...)` với `price` snapshot cũ → lệch giá hiện tại → requote/lệch.
**Khắc phục:** market order nên để giá = 0 (broker tự lấy), hoặc lấy lại `SYMBOL_ASK/BID` ngay trước khi gửi.

### V17 — Partial fill
Lot lớn có thể khớp một phần → state `volume` không khớp thực tế.
**Khắc phục:** đối chiếu `POSITION_VOLUME` thực sau khi đặt; cập nhật lại `PriceVolume.volume`.

### V18 — CloseByTicket nhầm ticket
`trade.ResultOrder()` trả **order ticket**; với position cần **position ticket**. MoneyBot map qua
`OnTradeTransaction` (deal), nhưng vẫn có đường đi dùng order ticket để `PositionSelectByTicket`.
**Khắc phục:** chuẩn hóa: position dùng `POSITION_TICKET`, pending dùng order ticket; tách 2 hàm `ClosePosition` / `DeletePending`.

---

## ⚪ LOW / Code cleanliness

### V19 — Magic numbers rải rác
`7`, `0`, `100`, `20`, `2`, `19`, `jump`, `NormalizeDouble(...,3)`... rải khắp.
**Khắc phục:** đưa hết lên `input` + hằng `#define`/`const`. Xem [02_CODE_AUDIT.md](02_CODE_AUDIT.md) §inputs.

---

## Ma trận ưu tiên sửa

```
        Tác động cao
            │  V1 V2 V3        │  V4 V5
            │  (cháy account)  │  (hành vi sai)
   Dễ sửa ──┼──────────────────┼────────────── Khó sửa
            │  V6 V7 V9 V10    │  V12 V13 V17 V18
            │  V14 V15 V19     │  (refactor sâu)
        Tác động thấp
```
**Thứ tự đề xuất:** V6,V7,V9,V10 (quick win) → V1,V2 (P0 safety) → V4,V5 (đúng hành vi) → V3 (R:R) → còn lại.
