# 02 — Code Audit: function review, bug, functions thiếu, kiến trúc đề xuất

> Mục tiêu: làm sạch & module hóa code để dễ thêm 4 lớp bảo vệ P0. Lấy **MoneyBot làm nền**.

---

## 1. Audit từng function (MoneyBot)

| Function | Đánh giá | Vấn đề | Hành động |
|----------|----------|--------|-----------|
| `OnInit` | ⚠️ thiếu | Chỉ `EventSetTimer(1)`. Không set magic, không rebuild state, không lưu `g_dayStartEquity` | Thêm setup đầy đủ |
| `OnTimer` | ⚠️ | Khớp giây tuyệt đối (V10); không có risk gate; gọi scan mỗi giây kể cả khi không cần | Thêm `RiskGateOpen()`, sửa trigger theo ngày |
| `OnTradeTransaction` | ✅ tốt | Reset TP/SL + map order→deal đúng hướng | Giữ, thêm log/persist |
| `startDailyBias` | ⚠️ | Hardcode 19 level, khoảng cách 1, SL −20/TP +2, BUY cứng, NormalizeDouble ẩn | Tham số hóa, gọi `BiasEngine`, `RiskManager` |
| `scanDailyBias` | 🐛 | `i < Total()-1` bỏ sót cuối (V15); `NULL` sentinel (V14); chỉ xử lý BUY (logic SELL không đối xứng) | Sửa biên, sentinel, đối xứng BUY/SELL |
| `updateTicketOpen` | ✅ | Map order→deal ok | Giữ |
| `ClearPriceVolumeList` | ✅ | Giải phóng `delete pv` đúng | Giữ |
| `deleteTicketFromList` | ⚠️ | Khai báo nhưng **không thấy gọi** (dead code?) | Xác minh, xóa nếu thừa |
| `getCurrentPrice` | ✅ | Ask/Bid đúng chiều | Giữ |
| `PlaceOrder` | ⚠️ | Không magic, không validate lot/margin/spread, market truyền giá cũ (V16) | Bọc validation |
| `CloseByTicket` | ⚠️ | Nhầm position vs order ticket (V18) | Tách 2 hàm |
| `PrintPriceVolumeList` | ✅ | Debug ok | Giữ |

### BiasBot (bổ sung)
- `m_tickets.Size()` — `string[]` không có `.Size()`, phải `ArraySize()` (V13, nghi không build).
- `parsePrefix` / `TicketInfoToString` / `SplitString` / `StringToULong` / `ULongToString` — **toàn bộ tầng serialize string là kỹ thuật nợ**. MoneyBot đã thay bằng OOP → **bỏ hẳn** khi hợp nhất.
- `dailyBiasRuning` khai báo `int` ở BiasBot nhưng gán `= false` ở `scanDailyBias` → trộn kiểu.

---

## 2. Functions còn THIẾU (cần cover)

> Đây là khoảng trống giữa "EA demo" và "EA production".

| Function / Module | Vai trò | Severity |
|-------------------|---------|----------|
| `DetermineDailyBias()` | Xác định BUY/SELL/NONE theo cấu trúc thị trường | 🔴 (V4) |
| `IsNewsBlackout()` | Trả true nếu đang trong cửa sổ tin high-impact | 🔴 (V1) |
| `RiskGateOpen()` | Circuit breaker: max daily loss, max DD, max positions | 🔴 (V2) |
| `SaveState()` / `LoadState()` / `RebuildFromBroker()` | Persist + khôi phục state | 🔴 (V5) |
| `CalcLotByRisk()` | Position sizing theo % risk + ATR | 🟠 |
| `NormalizeLot()` | Clamp lot theo min/max/step sàn | 🟠 (V8) |
| `HasEnoughMargin()` | Kiểm tra free margin trước khi đặt | 🟠 (V8) |
| `SpreadPoints()` / spread filter | Chặn đặt khi spread giãn | 🟠 (V9) |
| `CloseAllByMagic()` | Đóng toàn bộ vị thế + hủy pending của bot | 🟠 |
| `ManageBasketExit()` | TP/SL theo **cả cụm** (basket) thay vì từng lệnh | 🟠 (V3) |
| `TrailStop()` / `MoveToBreakeven()` | Bảo vệ lãi khi cụm đã dương | 🟡 |
| `ShouldCloseForSession()` | Đóng trước rollover/cuối tuần | 🟠 (V11) |
| `Log()` (cấp độ) | Logging có level + ghi file | ⚪ |

---

## 3. Kiến trúc module đề xuất

Tách monolith thành các `.mqh` theo trách nhiệm đơn (SRP). EA chính chỉ điều phối.

```
Bot_Trading/
├── DailyBiasGrid.mq5          # EA chính: OnInit/OnTimer/OnTradeTransaction — chỉ orchestrate
├── class/
│   ├── PriceVolume.mqh        # (giữ) data class — nâng cấp enum state
│   ├── Config.mqh             # tất cả input + hằng số (hết magic number)
│   ├── BiasEngine.mqh         # DetermineDailyBias(): EMA/HTF/Asian range
│   ├── NewsFilter.mqh         # IsNewsBlackout(): MT5 Calendar API (doc 04)
│   ├── RiskManager.mqh        # CalcLotByRisk, NormalizeLot, RiskGateOpen, margin
│   ├── TradeExecutor.mqh      # PlaceOrder/Close* + spread/slippage/magic
│   ├── GridManager.mqh        # build thang giá, scan, basket exit, hủy lệnh xấu
│   ├── StateStore.mqh         # Save/Load/RebuildFromBroker (persist)
│   └── Logger.mqh             # log có level + file
```

**Luồng `OnTimer` sau refactor:**
```mql5
void OnTimer() {
   if (!RiskGateOpen())        return;          // V2
   if (IsNewsBlackout())       { PauseOrFlatten(); return; } // V1
   if (ShouldCloseForSession()) { CloseAllByMagic(); return; } // V11

   if (TimeToStartBias() && !g_running) {        // V10 fix
      ENUM_ORDER_TYPE bias = DetermineDailyBias(); // V4
      if (bias != WRONG_VALUE) Grid.Start(bias);
   }
   if (g_running) {
      Grid.Scan();
      Grid.ManageBasketExit();                   // V3
   }
}
```

---

## 4. Nâng cấp `PriceVolume` — enum state

Thay 2 cờ `isOpen`/`isActiveStop` (4 tổ hợp, có tổ hợp vô nghĩa) bằng enum tường minh:
```mql5
enum ENUM_LEVEL_STATE {
   LEVEL_WAITING,    // chưa đặt gì
   LEVEL_PENDING,    // đã đặt BUY_STOP/SELL_STOP, chờ khớp
   LEVEL_OPEN,       // đã thành position
   LEVEL_SKIPPED,    // bị hủy (lệnh xấu)
   LEVEL_CLOSED      // đã đóng (TP/SL/manual)
};
```
Lợi ích: máy trạng thái rõ ràng, dễ kiểm tra bất biến (không bao giờ `isOpen && isActiveStop`).

---

## 5. StateStore — pattern persist + rebuild (V5)

```mql5
// Lưu sau mỗi thay đổi list
void SaveState() {
   string csv = "";
   for (int i=0; i<list.Total(); i++) {
      PriceVolume *pv = (PriceVolume*)list.At(i);
      csv += StringFormat("%I64u,%.2f,%.5f,%d\n",
              pv.TicketId(), pv.Volume(), pv.Price(), pv.State());
   }
   int h = FileOpen("dbg_state.csv", FILE_WRITE|FILE_TXT|FILE_COMMON);
   if (h != INVALID_HANDLE) { FileWriteString(h, csv); FileClose(h); }
}

// OnInit: ưu tiên rebuild từ SÀN (nguồn chân lý), file chỉ bổ trợ metadata
void RebuildFromBroker() {
   for (int i=PositionsTotal()-1; i>=0; i--) {
      ulong t = PositionGetTicket(i);
      if (PositionGetInteger(POSITION_MAGIC) != InpMagic) continue;
      // dựng lại PriceVolume từ POSITION_PRICE_OPEN, POSITION_VOLUME...
   }
   for (int i=OrdersTotal()-1; i>=0; i--) { /* tương tự cho pending */ }
}
```
**Nguyên tắc vàng:** sàn là nguồn chân lý của vị thế; file/GlobalVariable chỉ lưu metadata
(bias hướng nào, level nào là entry đẹp). Không bao giờ tin 100% vào state in-memory.

---

## 6. Inputs đề xuất (Config.mqh) — hết magic number

```mql5
input long   InpMagic            = 880628;     // V6
input int    InpStartHour        = 7;          // mốc giờ bias
input int    InpStartMinute      = 0;
input int    InpLevels           = 19;         // số level grid
input double InpGridStepPoints   = 200;        // khoảng cách level (points)  V7
input double InpRiskPerTradePct  = 0.5;        // % equity / cụm
input double InpMaxDailyLossPct  = 3.0;        // V2
input double InpMaxTotalDDPct    = 15.0;       // V2
input int    InpMaxSpreadPoints  = 50;         // V9
input int    InpSlippagePoints   = 20;
input double InpBasketTP_ATR     = 1.0;        // V3 TP theo ATR
input double InpBasketSL_ATR     = 2.0;        // V3 SL theo ATR
input int    InpNewsBeforeMin    = 30;         // V1 cửa sổ trước tin
input int    InpNewsAfterMin     = 30;         // V1 cửa sổ sau tin
input bool   InpCloseFriday      = true;       // V11
input int    InpCloseFridayHour  = 22;
```

---

## 7. Checklist refactor (làm theo thứ tự)

- [ ] Tách `Config.mqh`, đưa hết input/hằng số ra (V19, V7)
- [ ] `TradeExecutor`: magic + lot/margin/spread/slippage (V6, V8, V9, V16)
- [ ] Sửa trigger timer theo ngày (V10), sentinel `-1` (V14), biên vòng lặp (V15)
- [ ] `RiskManager.RiskGateOpen()` + lưu `g_dayStartEquity` trong OnInit (V2)
- [ ] `NewsFilter.IsNewsBlackout()` (V1, doc 04)
- [ ] `BiasEngine.DetermineDailyBias()` (V4) + đối xứng SELL trong scan
- [ ] `StateStore` save + `RebuildFromBroker` (V5)
- [ ] `GridManager.ManageBasketExit()` ATR-based (V3)
- [ ] `ShouldCloseForSession()` (V11)
- [ ] Bỏ tầng serialize string của BiasBot, hợp nhất về OOP
- [ ] Backtest đối chiếu trước/sau refactor (hành vi grid không đổi)
```
