# 06 — Roadmap nâng cấp

> Lộ trình theo phase. Mỗi phase **độc lập deliver được** và **luôn giữ bot ở trạng thái an
> toàn hơn trước**. Không nhảy phase: P0 (an toàn) phải xong trước khi tối ưu lợi nhuận.

---

## Tổng quan phase

```
P0  SAFETY        ──►  P1  CORRECTNESS  ──►  P2  RISK ENGINE  ──►  P3  VALIDATION  ──►  P4  EDGE/ML
(đừng cháy)            (làm đúng)            (định lượng risk)     (chứng minh)         (mở rộng)
```

---

## P0 — SAFETY (chặn cháy account) · ưu tiên tuyệt đối

> Mục tiêu: bot **không thể** xóa tài khoản kể cả khi chiến lược sai.

| Task | Lỗ hổng | Doc |
|------|---------|-----|
| News filter + action (FLATTEN cho FOMC/NFP/CPI) | V1 | [04](04_NEWS_FILTER.md) |
| Equity circuit breaker (daily loss + max DD) | V2 | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §5 |
| State persistence + RebuildFromBroker | V5 | [02](02_CODE_AUDIT.md) §5 |
| Magic number + lot/margin/spread validation | V6,V8,V9 | [01](01_RISK_ASSESSMENT.md) |
| Đóng trước rollover/cuối tuần | V11 | [01](01_RISK_ASSESSMENT.md) |

**Definition of done:** chạy demo 2 tuần qua ≥1 phiên NFP/FOMC mà bot tự ngừng đúng + circuit
breaker kích hoạt khi ép lỗ giả lập.

---

## P1 — CORRECTNESS (làm đúng hành vi)

| Task | Lỗ hổng | Doc |
|------|---------|-----|
| `BiasEngine.DetermineDailyBias()` (bỏ hardcode BUY) | V4 | [01](01_RISK_ASSESSMENT.md), [05](05_MARKET_BOT_TECHNIQUES.md) §2.1 |
| Đối xứng hóa scan cho cả SELL | — | [02](02_CODE_AUDIT.md) §1 |
| Sửa timer trigger theo ngày | V10 | [01](01_RISK_ASSESSMENT.md) |
| Sentinel `-1`, biên vòng lặp, NormalizeDouble theo `_Digits` | V14,V15,V7 | [01](01_RISK_ASSESSMENT.md) |
| Hợp nhất về OOP (bỏ tầng string BiasBot), port `OnTradeTransaction` | V13 | [02](02_CODE_AUDIT.md) |
| Tách module (Config/Trade/Grid/State/...) | — | [02](02_CODE_AUDIT.md) §3 |

**Definition of done:** một codebase duy nhất (`DailyBiasGrid.mq5` + `class/*.mqh`), build sạch,
chạy được cả BUY lẫn SELL theo bias thật, hành vi grid khớp backtest trước refactor.

---

## P2 — RISK ENGINE (định lượng rủi ro)

| Task | Doc |
|------|-----|
| Position sizing theo % risk (bell-curve thành trọng số) | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §2 |
| Basket TP/SL theo ATR (thay TP+2/SL−20) | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §3 |
| Trần `MaxBasketLot`, `MaxOpenLevels`, risk cụm ≤2% | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §3 |
| Trailing stop / move-to-breakeven | [02](02_CODE_AUDIT.md) §2 |
| Regime filter (ADX/ATR) — tắt grid khi trend mạnh | [05](05_MARKET_BOT_TECHNIQUES.md) §2.5 |

**Definition of done:** risk mỗi cụm có trần cứng theo % equity; backtest cho thấy max DD nằm
trong ngưỡng đặt ra ở mọi giai đoạn stress.

---

## P3 — VALIDATION (chứng minh edge)

| Task | Doc |
|------|-----|
| Backtest tick-data thật, spread+commission+swap | [05](05_MARKET_BOT_TECHNIQUES.md) §5 |
| In-sample / out-of-sample split | [05](05_MARKET_BOT_TECHNIQUES.md) §5 |
| Walk-forward optimization | [05](05_MARKET_BOT_TECHNIQUES.md) §5 |
| Monte Carlo drawdown distribution | [05](05_MARKET_BOT_TECHNIQUES.md) §5 |
| Stress test 2020/2022/2023/2024 | [03](03_RISK_MANAGEMENT_BLUEPRINT.md) §7 |
| Parameter sensitivity (tránh đỉnh nhọn) | [05](05_MARKET_BOT_TECHNIQUES.md) §5 |

**Definition of done:** edge bền vững quanh vùng tham số trên OOS; báo cáo Monte Carlo có
risk-of-ruin chấp nhận được. **Nếu fail → quay lại P1/P2, không live.**

---

## P4 — EDGE / MỞ RỘNG (chỉ khi P0–P3 vững)

| Hướng | Mô tả | Doc |
|-------|-------|-----|
| Nguồn bias nâng cao | SMC/ICT, ORB phiên, multi-TF confluence | [05](05_MARKET_BOT_TECHNIQUES.md) §2 |
| Volatility regime switching | Đổi chế độ theo vol | [05](05_MARKET_BOT_TECHNIQUES.md) §2.5 |
| Portfolio nhiều edge ít tương quan | Giảm phụ thuộc 1 chiến lược | [05](05_MARKET_BOT_TECHNIQUES.md) §1 |
| ML feature-based bias | XGBoost/LSTM dự đoán hướng ngày | [05](05_MARKET_BOT_TECHNIQUES.md) §1 |
| Tích hợp hệ VMAX | Đẩy trạng thái/alert qua Supabase `fx` schema, dashboard giám sát | CLAUDE.md R1 |
| Tự động cập nhật news CSV | Agent Dev cron tải ForexFactory JSON | [04](04_NEWS_FILTER.md) §3 |

---

## Gợi ý tích hợp với dự án VMAX Martis

- News CSV tự động: giao **Agent Dev** chạy script định kỳ tải lịch tin → `Common\Files\news.csv`.
- Telemetry: bot ghi trạng thái cụm + P/L vào schema `fx` (Supabase) → dashboard XAUUSD giám sát realtime.
- Alert: circuit breaker kích hoạt → push qua kênh thông báo của hệ thống.
- Nếu chính thức hóa thành task lớn → tạo **FEAT** theo workflow CLAUDE.md (§FEAT Workflow).

---

## Bảng theo dõi tiến độ (cập nhật khi làm)

| Phase | Trạng thái | Ghi chú |
|-------|-----------|---------|
| P0 Safety | ⬜ chưa bắt đầu | |
| P1 Correctness | ⬜ | |
| P2 Risk Engine | ⬜ | |
| P3 Validation | ⬜ | |
| P4 Edge/ML | ⬜ | |

> ⬜ chưa · 🚧 đang làm · ✅ xong
