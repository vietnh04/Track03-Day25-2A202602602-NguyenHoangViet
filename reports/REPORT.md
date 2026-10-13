# Lab 25B — Report

**Họ tên / nhóm:** <điền họ tên + MSV>  **Commit:** <điền hash sau khi push>  **Điểm tự động (`make grade`):** 84 / 88 (4đ dữ liệu ẩn của Part E không chấm được cục bộ)

> Số liệu lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json` (seed 25).

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.719 | 7.16 | 7.74 | 6.74 | 2.16 | 4 | FAIL |
| G1 | yours | 1.000 | 4.78 | 1.06 | 0.001 | 2.10 | 0 | PASS |
| G2 | naive | 0.456 | 15.03 | 9.90 | 8.90 | 4.63 | 155 | FAIL |
| G2 | yours | 1.000 | 10.36 | 1.04 | 0.005 | 2.11 | 0 | PASS |
| G3 | naive | 1.000 | 4.77 | 1.16 | 0.164 | 2.33 | 0 | FAIL |
| G3 | yours | 1.000 | 4.67 | 1.06 | 0.030 | 2.14 | 0 | PASS |
| G4 | naive | 1.000 | 13.76 | 1.00 | 0.004 | 2.01 | 0 | FAIL |
| G4 | yours | 1.000 | 5.63 | 1.01 | 0.002 | 2.07 | 0 | PASS |
| G5 | naive | 0.370 | 5.36 | 17.03 | 16.03 | 2.12 | – | FAIL |
| G5 | yours | 0.818 | 5.48 | 1.00 | 0.002 | 2.50 | – | PASS |
| G6 | naive | 0.517 | 4.84 | 12.34 | 11.34 | 2.01 | 0 | FAIL |
| G6 | yours | 1.000 | 6.03 | 1.06 | 0.019 | 2.19 | 0 | PASS |

## 2. Cấu hình client và lý do

Giữ nguyên giá trị khối `client:` của starter vì cả 6 băng đã PASS với biên an toàn lớn; không tinh chỉnh thêm để tránh overfit vào băng công khai.

| Tham số | Giá trị | Vì sao |
|---|---:|---|
| attempt_timeout_s | 8.0 | Đủ cho lúc bình thường (~1,2–1,8 s) nhưng đủ ngắn để còn thời gian fail over trong deadline 30 s (G6: `alpha-eu` treo). |
| max_attempts_per_provider | 2 | 1 retry cùng provider rồi fail over; amplification chỉ 1,0–1,06 ở mọi băng. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter, giãn các retry để không đồng bộ. |
| retry_budget_ratio | 0.1 | Retry ≤ ~10% lượng request; G3 retry_ratio = 0,03 (ngưỡng 0,10). |
| breaker.window_size / min_calls | 20 / 10 | Không trip khi mới có vài lỗi, vẫn phản ứng trong vài giây ở 4 turn/s. |
| breaker.failure_rate_threshold | 0.5 | G6 (alpha 503) trip nhanh; G3 (30% lỗi) không trip nhầm. |
| breaker.slow_call_s | 5.0 | G4: provider chậm +5 s không có lỗi — breaker bắt call chậm, p95 13,76 → 5,63 s. |
| breaker.open_duration_s | 10.0 | Thăm dò lại sớm; G2 ≥ 80% traffic quay lại `alpha`. |
| breaker.half_open_max_probes | 2 | Giới hạn probe, tránh thundering herd lên provider đang hồi phục. |

```
on_error(kind):
 CLIENT_ERROR    -> breaker.release            -> GiveUp("client_error")   (không retry, không failover)
 QUOTA_EXHAUSTED -> release, cooldown 300 s    -> failover ngay
 RATE_LIMITED    -> release, cooldown retry_after -> failover ngay (chờ <=5 s nếu cạn ladder)
 OVERLOADED/SERVER_ERROR/TIMEOUT -> breaker.on_failure
                    -> retry cùng provider (backoff) nếu tries<2 && budget && kịp deadline
                    -> ngược lại failover
```

## 3. Ba thí nghiệm

Chưa chạy các thí nghiệm này (cần đổi config và chạy lại các băng) — **cần bổ sung trước khi nộp**.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (cents) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,496,700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

- Chỉ idempotency key: worker chết thì mất state, chạy lại từ đầu → tốn lại LLM call (và phụ thuộc nhà cung cấp lưu key).
- Chỉ checkpoint: nếu tiền đã đi nhưng worker chết/timeout trước khi ghi checkpoint, lần chạy lại gọi payment lần nữa mà không có key → hoàn tiền trùng.
- Ghi checkpoint *trước* khi gọi payment: crash giữa chừng thì lần resume tin rằng refund đã xong → **mất hoàn tiền** (lost refund). Vì vậy: thực thi step → checkpoint sau khi thành công, key ổn định `refund-agent:{session}:{step}`.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (phút) |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 |
| provider_outage | page | page | page | 4 |
| deploy_blip | none | **false page** | không | – |
| slow_leak | ticket | bỏ lỡ | ticket | 1337 |
| degraded_evening | page | page | page | 291 |
| night_spike | none | **false page** | không | – |
| small_bump | none | **false page** | không | – |

Error budget 30 ngày: đã tiêu 42,95%, còn 57,05% → **không** đóng băng release. `small_bump` (20 phút ở 12% lỗi) chỉ đốt ~0,3% budget tháng; cửa sổ dài 1h/6h không đạt ngưỡng nên không page — đánh thức on-call vì nó là alert fatigue.

## 6. Postmortem — Băng G2 với client naive (blameless)

**Tóm tắt:** `alpha` mất 75% năng lực trong 3 phút (t≈120–300). Client naive (3 tầng retry, tối đa 36 attempt/call, delay cố định, không breaker, không fallback) khuếch đại tải ~9,9×, nên hàng đợi `alpha` không bao giờ cạn và hệ thống vẫn sập đến t≈455 dù provider đã hồi phục từ t=300.

**Ảnh hưởng:** availability 45,6% (54,4% lượt user thất bại); recovery 155 s sau khi provider hồi phục. Với SLO 99,5% thì sự cố này một mình vượt xa budget tháng giả định (lỗi 54% ≫ 0,5%).

**Timeline** (giây ảo, từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng |
|---:|---|---|
| 120 | alpha mất 75% năng lực | `attempts_alpha` vọt từ ~5 lên 216/bucket, `calls_ok` về 0 |
| 140–300 | retry storm | 105–216 attempt/bucket nhưng chỉ 0–5 call hoàn tất; amplification 9,9 |
| 300 | alpha hồi phục hoàn toàn | attempt vẫn 115/bucket, `calls_ok` = 0 |
| 300–455 | metastable failure | hàng đợi đầy request đã timeout nhưng server vẫn xử lý; retry mới nạp tiếp |
| ~455 | hồi phục | `calls_ok` về 9–11/bucket, attempt ~10 |

**Trigger vs root cause:** trigger là mất 75% GPU của `alpha`. Root cause là thiết kế retry: 3 tầng chồng nhau, không backoff/jitter, không retry budget, không breaker, không fallback. Vòng tự duy trì: timeout → retry → thêm tải → hàng đợi dài hơn → thêm timeout; server vẫn xử lý request đã bị client bỏ nên công suất bị phí.

**Giúp / làm tệ hơn:** giúp: sự cố có điểm kết thúc rõ ràng. Tệ hơn: 36 attempt/call, delay cố định (đồng bộ), không tôn trọng giới hạn tải, không có đường fail over sang `alpha-eu`/`beta`.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Chỉ retry ở 1 tầng, thêm full jitter + retry budget 10% | prevent | Platform | G2 amplification ≤ 1,6 (hiện 1,04) |
| 2 | Circuit breaker sliding-window + fallback ladder | mitigate | Platform | G2 recovery ≤ 30 s, ≥ 80% quay lại `alpha` |
| 3 | Alert burn-rate đa cửa sổ + dashboard amplification | detect | SRE | `make slo`: page provider_outage trong 4 phút |

## 7. Một điểm yếu còn lại

G2 p95 của client mới vẫn 10,4 s (cao hơn mọi băng khác) do một số call phải chờ timeout 8 s trước khi breaker trip. Có thể giảm `attempt_timeout_s` theo p99 thực tế hoặc hedged request sang `alpha-eu`; chi phí là thêm tải/chi phí. Ngoài ra, khi cả 4 provider cùng chậm thì ladder không còn nơi nào để đi.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.837 | 0.842 | 0.0 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.912 | 0.965 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao |
|---|---:|---|
| similarity_threshold | 0.95 | Ở 0,80 (starter) `wrong_answers` = 181 (toàn bộ `other_question` — câu hỏi gần mà khác nghĩa); 0,95 đưa về 0 mà hit vẫn 65%. |
| ttl_s | 60 | Thay đổi bảng phí ship không báo trước chỉ có TTL giới hạn; ở 600 s starter `outdated_answers` = 80, ở 60 s còn 3 (≤ 15). |
| max_stale_s | 600 | Phải dài hơn cửa sổ sập 120 s của mọi provider; ở 0 (starter) không có stale-if-error, faq_availability chỉ 0,859 → 0,965 với 600. |

Thí nghiệm: các số trên so sánh trực tiếp starter (0,80 / 600 / 0) với cấu hình cuối; chưa quét thêm 0,97 / TTL 30–300.

- Câu hỏi cá nhân không được cache vì câu trả lời phụ thuộc trạng thái thay đổi theo thời gian của đúng user (trạng thái đơn, tiền hoàn); cache theo user vẫn trả dữ liệu cũ và tăng nguy cơ lộ dữ liệu sang user khác nếu scope sai.
- Dashboard xanh nhờ cache: alert theo `stale_served`/tỷ lệ stale, tỷ lệ `GiveUp` và error rate của provider (đo trước cache), không chỉ availability phía user.
