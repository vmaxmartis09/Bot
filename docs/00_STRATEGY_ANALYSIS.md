# 00 — Phân tích chiến lược hiện tại

> Mục tiêu: hiểu **chính xác** bot đang làm gì trước khi đánh giá rủi ro. Sai ở bước này
> → mọi khuyến nghị phía sau đều lệch.

---

## 1. Phân loại chiến lược (quan trọng nhất)

Người dùng gọi đây là "hedging", nhưng về mặt kỹ thuật **đây không phải hedging**. Phân loại đúng:

| Tiêu chí | Hedging thật | Bot này |
|----------|--------------|---------|
| Mở 2 chiều đối ứng (Buy + Sell) | ✅ | ❌ chỉ 1 chiều/ngày |
| Mục tiêu | Khóa lỗ / trung hòa rủi ro | Gom vị thế theo bias |
| Khi giá đi ngược | Lệnh đối ứng bù lỗ | Vào thêm cùng chiều (averaging) |

**Phân loại đúng:** `Daily-Bias Directional Grid` + `Averaging-down (DCA)` + `Stop-order confirmation` + `Bell-curve volume scaling`.

Nói cách khác: bot **nhồi thêm lệnh cùng chiều khi giá đi ngược dự đoán**, hy vọng giá hồi
về để chốt cả cụm. Đây là họ **mean-reversion / recovery grid** — đặc trưng:

- **Win-rate cao, lãi nhỏ đều** (TP gần).
- **Thua hiếm nhưng cực to** (khi giá chạy 1 chiều không hồi → SL toàn cụm).
- Đường equity đẹp như mơ... cho đến ngày nó không đẹp. Đây gọi là **negative skew /
  "picking pennies in front of a steamroller"**.

> ⚠️ XAUUSD là tài sản **trend mạnh + gap mạnh** (phản ứng USD, lãi suất, địa chính trị).
> Đây đúng là loại thị trường khắc tinh của recovery grid. Xem [01_RISK_ASSESSMENT.md](01_RISK_ASSESSMENT.md).

---

## 2. Cơ chế chung (cả 2 bot)

```
                 07:00:00 server time
                        │
                        ▼
              ┌──────────────────┐
              │  startDailyBias  │  snapshot giá hiện tại = P0
              └──────────────────┘
                        │
        ┌───────────────┼────────────────────────────┐
        ▼                                              ▼
  Lệnh #0 (market BUY ngay tại P0)        Tạo thang giá phía dưới:
  SL = P0 - X, TP = P0 + Y                 level_i giá = P0 - i*jump
                                           volume_i = bell-curve[i]
                                           state = WAITING_STOP
                        │
                        ▼
              ┌──────────────────┐   mỗi giây (OnTimer)
              │   scanDailyBias  │
              └──────────────────┘
                        │
     Khi giá rớt tới activePrice của level_i:
        → đặt BUY_STOP tại level_i  (vào thêm khi giá xác nhận hồi lên)
        → hủy các lệnh "xấu" phía trên entry đẹp nhất
                        │
                        ▼
              TP hit  → chốt lời cả cụm, reset
              SL hit  → cắt lỗ cả cụm, reset (chỉ MoneyBot tự reset)
```

**Triết lý:** không "đoán đáy" bằng limit order (dễ bị giá xuyên thủng), mà chờ giá **hồi
lên xác nhận** rồi mới vào bằng `BUY_STOP` → giảm rủi ro bắt dao rơi. Đây là điểm thiết kế
**thông minh** nhất của bot, cần giữ.

---

## 3. Khác biệt BiasBot vs MoneyBot

### 3.1 BiasBot.mq5 (string-based, cũ)

- State serialize thành chuỗi: `"T{ticket} V{vol} S{state} P{price} A{activePrice}"`, lưu trong `string m_tickets[]`.
- Có biến `jump` (mặc định 2) → khoảng cách giữa các level + chọn volume profile:
  - `jump==1` → `m_volumes1[19]` (chuông nhẹ: 0.03→0.10→0.03)
  - else → `m_volumes2[10]` (chuông mạnh: 0.05→0.16→0.07)
- **Điểm khác biệt lớn — cộng dồn volume:**
  ```
  totalVolume += ticketInfo.volume   // cộng dồn TẤT CẢ level từ 1..i
  PlaceOrder(BUY_STOP, price, totalVolume, ...)
  ```
  → mỗi lần kích hoạt, đặt 1 buy-stop với **tổng khối lượng tích lũy**, không phải volume
  riêng của level. Đây là cơ chế **recovery**: gom toàn bộ size dồn vào 1 entry sâu hơn.
- Không có `OnTradeTransaction` → **không tự reset khi TP/SL**, không map order→deal.

### 3.2 MoneyBot.mq5 (class-based, mới)

- State là object `PriceVolume` trong `CArrayObj priceVolumeList` — sạch, dễ maintain.
- Luôn tạo **đúng 19 level**, khoảng cách cố định 1 (hardcode `currentPrice - i - 1`).
- **Volume riêng từng level** (KHÔNG cộng dồn như BiasBot).
- Có `OnTradeTransaction`:
  - `DEAL_REASON_TP || DEAL_REASON_SL` → `ClearPriceVolumeList()` + `dailyBiasRuning = 0` (tự reset ✅)
  - Deal add khác → `updateTicketOpen()` map `order_ticket → deal_ticket`.
- SL = `price - 20`, TP = `price + 2`.

### 3.3 Bảng so sánh

| Tiêu chí | BiasBot | MoneyBot |
|----------|---------|----------|
| Lưu state | `string[]` (fragile, parse tay) | `CArrayObj` OOP (tốt hơn) |
| Số level | 10 hoặc 19 (theo `jump`) | luôn 19 |
| Khoảng cách level | `jump` points (2) | 1 point (hardcode) |
| Volume mỗi entry | **cộng dồn** (recovery) | riêng từng level |
| Tự reset TP/SL | ❌ | ✅ `OnTradeTransaction` |
| Map order→deal | ❌ | ✅ |
| SL / TP | `-100 / +2` (TODO "chưa handler") | `-20 / +2` |

**Kết luận:** `MoneyBot` là bản tiến hóa đúng hướng (OOP + lifecycle). Nên **lấy MoneyBot làm
nền**, port lại cơ chế cộng-dồn-volume của BiasBot **chỉ khi** đã hiểu rủi ro (xem dưới).

---

## 4. Điểm mạnh (giữ lại)

1. **Stop-order confirmation** thay vì limit — không bắt dao rơi, chỉ vào khi giá xác nhận hồi.
2. **Bell-curve volume** — cap exposure ở đỉnh chuông, không phải martingale x2 (đỡ cháy hơn nhiều so với martingale thuần).
3. **Hủy lệnh xấu phía trên** — gom về 1 entry đẹp, giảm số vị thế rời rạc.
4. **MoneyBot lifecycle** — `OnTradeTransaction` reset gọn gàng.
5. **Tách `PlaceOrder` / `CloseByTicket`** — abstraction tốt, dễ mở rộng.

---

## 5. Điểm yếu cốt lõi (chi tiết ở [01_RISK_ASSESSMENT.md](01_RISK_ASSESSMENT.md))

1. **Không có bias engine thật** — `orderTypeDailyBias` hardcode `ORDER_TYPE_BUY`. Bot luôn BUY, kể cả ngày giảm.
2. **R:R âm** — TP +2 / SL −20 (MoneyBot) hoặc −100 (BiasBot). Cần win-rate >90% mới hòa vốn.
3. **Không news filter** — fire lúc 07:00 bất chấp NFP/FOMC.
4. **Không equity protection** — không có max daily loss, không circuit breaker.
5. **State chỉ in-memory** — MT5 restart → mất sạch state, vị thế mồ côi trên sàn.
6. **Giả định 3-digit gold** (`NormalizeDouble(x, 3)`) — nhiều sàn XAUUSD là 2-digit → sai giá.
7. **Không magic number** — không phân biệt lệnh của bot vs lệnh tay/EA khác.
8. **Timer fragile** — phụ thuộc `dt.sec == 0` khớp tuyệt đối; lệch 1 tick là cả ngày không chạy.

---

## 6. Sơ đồ trạng thái 1 lệnh (PriceVolume lifecycle)

```
        WAITING_STOP ──(giá chạm activePrice)──► ACTIVE_STOP ──(giá khớp)──► OPEN
             │                                        │                        │
             │ (nằm trên entry đẹp)                   │ (nằm trên entry đẹp)    │
             ▼                                        ▼                        ▼
           SKIP / xóa khỏi list ◄───────── hủy pending ────────────       TP/SL → reset cả cụm
```

> Trong MoneyBot, trạng thái được biểu diễn bằng 2 cờ `isOpen` + `isActiveStop` thay vì
> enum string. Đề xuất chuyển hẳn sang `enum` rõ ràng — xem [02_CODE_AUDIT.md](02_CODE_AUDIT.md).
