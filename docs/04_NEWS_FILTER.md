# 04 — Tự động né tin tức (News Filter)

> Đây là lớp bảo vệ P0 #1. Với XAUUSD, **một cú NFP/FOMC giữa cụm grid có thể xóa sạch hàng
> tháng lợi nhuận**. Bot phải biết tự ngừng quanh tin high-impact.

---

## 1. Vì sao bắt buộc

XAUUSD nhạy với USD và lãi suất Mỹ. Các tin gây nhảy giá mạnh nhất:

| Tin | Tần suất | Biên độ điển hình XAUUSD |
|-----|----------|--------------------------|
| FOMC (lãi suất + họp báo) | 8 lần/năm | $20–60, whipsaw 2 chiều |
| NFP (Non-Farm Payrolls) | hàng tháng (thứ 6 đầu tháng) | $15–40 trong 1 phút |
| CPI / Core CPI | hàng tháng | $15–35 |
| PCE (thước đo lạm phát Fed thích) | hàng tháng | $10–25 |
| Powell / Fed speakers | bất thường | $10–30 |
| Geopolitics (chiến tranh...) | bất thường | gap mạnh, khó lường |

Grid averaging-down + spike tin = kịch bản cháy điển hình. **Chặn là rẻ nhất.**

---

## 2. Phương án A — MT5 Economic Calendar API (khuyến nghị)

MT5 có sẵn calendar (chỉ hoạt động trên tài khoản broker MetaQuotes-based, **không chạy trong
Strategy Tester** — xem §5). Dùng `CalendarValueHistory` lọc theo quốc gia + tầm quan trọng.

```mql5
// NewsFilter.mqh
input int  InpNewsBeforeMin = 30;
input int  InpNewsAfterMin  = 30;
input bool InpBlockHighOnly = true;   // chỉ chặn high-impact

bool IsNewsBlackout() {
   datetime now = TimeCurrent();
   datetime from = now - InpNewsAfterMin*60;   // tin đã xảy ra trong cửa sổ sau
   datetime to   = now + InpNewsBeforeMin*60;   // tin sắp xảy ra trong cửa sổ trước

   MqlCalendarValue values[];
   // Lọc theo quốc gia US (mã "US"); vàng nhạy USD nhất
   if (CalendarValueHistory(values, from, to, "US") <= 0) return false;

   for (int i=0; i<ArraySize(values); i++) {
      MqlCalendarEvent event;
      if (!CalendarEventById(values[i].event_id, event)) continue;

      if (InpBlockHighOnly && event.importance != CALENDAR_IMPORTANCE_HIGH)
         continue;

      datetime t = values[i].time;
      if (t >= now - InpNewsAfterMin*60 && t <= now + InpNewsBeforeMin*60)
         return true;   // đang trong cửa sổ blackout
   }
   return false;
}
```

**Lọc thêm cho vàng:** ngoài `"US"`, cân nhắc theo currency `"USD"` và sự kiện liên quan
`CALENDAR_SECTOR_*` (jobs, inflation, central bank). Có thể whitelist/blacklist `event_id`.

---

## 3. Phương án B — CSV ngoại (ForexFactory / investing) làm fallback

Khi calendar MT5 không có data (một số broker / tester), nạp lịch tin từ file CSV cập nhật thủ
công hoặc qua script:

```
# news.csv  (datetime,impact,currency,title)
2026-07-03 19:30,HIGH,USD,Non-Farm Payrolls
2026-07-09 01:00,HIGH,USD,FOMC Minutes
```
```mql5
struct NewsEvent { datetime time; string impact; };
NewsEvent g_news[];

void LoadNewsCSV() {
   int h = FileOpen("news.csv", FILE_READ|FILE_CSV|FILE_COMMON, ',');
   // parse từng dòng → g_news[]
}
bool IsNewsBlackout() {
   datetime now = TimeCurrent();
   for (int i=0;i<ArraySize(g_news);i++)
      if (g_news[i].impact=="HIGH"
          && now >= g_news[i].time - InpNewsBeforeMin*60
          && now <= g_news[i].time + InpNewsAfterMin*60)
         return true;
   return false;
}
```

> Tự động hóa cập nhật CSV: cron/script tải JSON ForexFactory
> (`nfs.faireconomy.media/ff_calendar_thisweek.json`) → ghi `news.csv` vào thư mục `Common\Files`.
> Có thể giao cho một agent của dự án (Dev) chạy định kỳ.

---

## 4. Hành vi khi blackout — 3 chế độ

| Chế độ | Hành vi | Khi nào dùng |
|--------|---------|--------------|
| `PAUSE_NEW` | Không mở cụm mới / không thêm level, giữ vị thế đang có | Mặc định, ít can thiệp |
| `TIGHTEN` | Siết SL về breakeven + giảm size trước tin | Cụm đang lãi |
| `FLATTEN` | Đóng hết + hủy pending trước tin X phút | An toàn nhất, bỏ lỡ một số kèo |

```mql5
input ENUM_NEWS_ACTION InpNewsAction = NEWS_PAUSE_NEW;

void HandleNews() {
   if (!IsNewsBlackout()) return;
   switch (InpNewsAction) {
      case NEWS_PAUSE_NEW: g_allowNewEntries = false; break;
      case NEWS_TIGHTEN:   MoveBasketToBreakeven();   break;
      case NEWS_FLATTEN:   CloseAllByMagic();         break;
   }
}
```
Khuyến nghị XAUUSD: `FLATTEN` cho FOMC/NFP/CPI (impact cao nhất), `PAUSE_NEW` cho phần còn lại.

---

## 5. Cạm bẫy & lưu ý

1. **Strategy Tester không có calendar live** → backtest phải dùng phương án B (CSV) để mô phỏng. Tải lịch lịch sử về CSV cho khoảng backtest.
2. **Giờ server vs giờ tin** — `CalendarValueHistory` trả giờ theo broker server time; đối chiếu cẩn thận, lệch 1–3h là chuyện thường.
3. **Tin "all day" / không giờ cụ thể** (bank holiday) — xử lý riêng, coi cả ngày là blackout nhẹ.
4. **Spread proxy** — kể cả không có lịch, spread giãn đột biến là tín hiệu gián tiếp có tin → kết hợp spread filter làm lớp 2.
5. **Quyền truy cập calendar** — cần bật "Allow ... in the terminal" và broker hỗ trợ; kiểm tra `TerminalInfoInteger(TERMINAL_NOTIFICATIONS_ENABLED)` không liên quan, dùng `CalendarValueHistory` trả 0 để phát hiện không có data → fallback CSV.

---

## 6. Tích hợp vào vòng đời bot

```mql5
void OnTimer() {
   if (!RiskGateOpen()) return;

   HandleNews();                      // cập nhật cờ + hành động blackout
   if (IsNewsBlackout()
       && InpNewsAction == NEWS_FLATTEN) return;  // đã flatten, khỏi làm gì thêm

   if (TimeToStartBias() && !g_running && g_allowNewEntries)
      Grid.Start(DetermineDailyBias());

   if (g_running) { Grid.Scan(); Grid.ManageBasketExit(); }
}
```

> Liên quan: spread filter (V9) trong [01_RISK_ASSESSMENT.md](01_RISK_ASSESSMENT.md),
> circuit breaker trong [03_RISK_MANAGEMENT_BLUEPRINT.md](03_RISK_MANAGEMENT_BLUEPRINT.md).
