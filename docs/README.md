# Bot_Trading — Tài liệu phân tích & nâng cấp

> Bộ docs tổng hợp đánh giá chuyên môn 2 EA (`BiasBot.mq5`, `MoneyBot.mq5`) và lộ trình
> giảm thiểu rủi ro khi trade **XAUUSD**. Docs sống gần code theo rule R5 của dự án.

---

## ⚠️ Tóm tắt 30 giây — đọc trước khi làm bất cứ gì

Chiến lược hiện tại **KHÔNG phải hedging thuần** (không mở đồng thời Buy + Sell đối ứng).
Bản chất thật là:

> **Daily-Bias Grid + Averaging-down (DCA)** — mỗi ngày 07:00 mở 1 lệnh theo bias,
> rồi rải một thang giá phía dưới (cho BUY), khi giá hồi về vùng đẹp thì kích hoạt
> `BUY_STOP` để vào thêm. Volume theo hình chuông (bell-curve) để giới hạn exposure.

Đây là họ chiến lược **lãi đều / cháy hiếm nhưng cháy to** (negative skew). Trên XAUUSD —
tài sản biến động mạnh nhất nhì thị trường — nếu không có 4 lớp bảo vệ dưới đây thì
**chỉ là vấn đề thời gian trước khi một phiên NFP/FOMC quét sạch tài khoản**.

**4 lớp bảo vệ bắt buộc (P0 — chưa có cái nào trong code hiện tại):**

| # | Lớp | Trạng thái hiện tại | Doc |
|---|-----|---------------------|-----|
| 1 | News filter (né NFP/CPI/FOMC) | ❌ Không có | [04_NEWS_FILTER.md](04_NEWS_FILTER.md) |
| 2 | Equity circuit breaker (max daily loss) | ❌ Không có | [03_RISK_MANAGEMENT_BLUEPRINT.md](03_RISK_MANAGEMENT_BLUEPRINT.md) |
| 3 | State persistence (sống sót restart) | ❌ Không có | [02_CODE_AUDIT.md](02_CODE_AUDIT.md) |
| 4 | Magic number + lot validation | ❌ Không có | [01_RISK_ASSESSMENT.md](01_RISK_ASSESSMENT.md) |

---

## 📚 Cấu trúc bộ docs

| File | Nội dung | Đọc khi |
|------|----------|---------|
| [00_STRATEGY_ANALYSIS.md](00_STRATEGY_ANALYSIS.md) | Mổ xẻ chiến lược hiện tại, phân loại, cơ chế 2 bot, điểm mạnh/yếu | Hiểu bot đang làm gì |
| [01_RISK_ASSESSMENT.md](01_RISK_ASSESSMENT.md) | Bảng lỗ hổng (severity) + cách khắc phục cụ thể | Trước khi sửa code |
| [02_CODE_AUDIT.md](02_CODE_AUDIT.md) | Audit từng function, bug, functions còn thiếu, kiến trúc module đề xuất | Refactor code |
| [03_RISK_MANAGEMENT_BLUEPRINT.md](03_RISK_MANAGEMENT_BLUEPRINT.md) | Position sizing, max drawdown, R:R, Kelly — số cụ thể cho XAUUSD | Thiết kế risk engine |
| [04_NEWS_FILTER.md](04_NEWS_FILTER.md) | Tự động né tin tức bằng MT5 Calendar API + fallback | Build news filter |
| [05_MARKET_BOT_TECHNIQUES.md](05_MARKET_BOT_TECHNIQUES.md) | Kỹ thuật & chiến lược các bot khác trên thị trường đang dùng | Mở rộng tầm nhìn |
| [06_ROADMAP.md](06_ROADMAP.md) | Lộ trình nâng cấp theo phase P0→P4 | Lên kế hoạch |

---

## 🎯 Khuyến nghị hành động (ưu tiên giảm dần)

1. **DỪNG chạy live ngay** nếu chưa có news filter + equity circuit breaker. Backtest/demo trước.
2. Đọc [01_RISK_ASSESSMENT.md](01_RISK_ASSESSMENT.md) → vá 5 lỗ hổng **Critical** trước tiên.
3. Tách code thành module (xem [02_CODE_AUDIT.md](02_CODE_AUDIT.md) §Kiến trúc đề xuất).
4. Thêm 4 lớp bảo vệ P0 ở bảng trên.
5. Backtest 2018–nay (gồm 2020 COVID, 2022–2023 lãi suất) với spread thật + slippage.

---

## Bối cảnh code hiện tại

| File | Phiên bản | Cách lưu state | Có gì hơn |
|------|-----------|----------------|-----------|
| `BiasBot.mq5` | Cũ | Mảng `string[]` serialize `T..V..S..P..A..` | Có `jump`, 2 volume profile, **cộng dồn volume** |
| `MoneyBot.mq5` | Mới | `CArrayObj` + class `PriceVolume` (OOP) | `OnTradeTransaction` reset khi TP/SL, mapping order→deal |
| `class/PriceVolume.mqh` | — | Data class | `isOpen`, `isActiveStop`, `ticketId` |

> Chi tiết khác biệt: [00_STRATEGY_ANALYSIS.md](00_STRATEGY_ANALYSIS.md).
