# Lab 25B — Report

**Họ tên / nhóm:** Cao Đức Anh - MSV: 2A202602754  **Commit:** cf3abe93035f708cc9cc63fdabfef11c7f6156d8  **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4 điểm dữ liệu ẩn = 88 / 88)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | null | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | null | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Đủ cho service_median_s của provider chậm nhất (beta: 1.8s), nhưng ngắn hơn nhiều so với 30s deadline. Ở G6 và G2, timeout 8s giúp client phát hiện sớm provider treo/nghẽn để failover kịp thời thay vì đợi hết turn deadline (30s). |
| max_attempts_per_provider | 2 | Cho phép thử lại đúng 1 lần trên cùng 1 provider khi gặp lỗi transient (500/529/timeout). Giữ amplification thấp ở G1-G6 (amplification <= 1.062), ngăn chặn retry storm. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter với base 0.25s tránh xung đột đồng thời, cap 4.0s đảm bảo một lần retry vẫn kịp thời gian xử lý trước deadline của turn (30s). |
| retry_budget_ratio | 0.1 | Giới hạn số lượng retry không vượt quá 10% tổng số request gửi lần đầu. Ở G3 brown-out, tỷ lệ lỗi 30% nhưng retry_ratio được giữ ở mức 0.034 (đạt yêu cầu <= 0.10). |
| breaker.window_size / min_calls | 20 / 10 | Đủ mẫu thống kê (10-20 calls) để tính tỷ lệ lỗi chính xác mà không trip nhầm bởi các blip ngẫu nhiên ngắn hạn, bảo vệ provider khỏi mở circuit breaker oan. |
| breaker.failure_rate_threshold | 0.5 | Nếu >= 50% request trong cửa sổ bị lỗi server/quá tải/timeout thì mở breaker ngay lập tức. Ở G2 và G6, trip kịp thời giúp 100% availability và chuyển traffic sang alpha-eu/beta. |
| breaker.slow_call_s | 5.0 | Ngưỡng bắt các call chậm bất thường (service_median_s chuẩn là 1.2s - 1.8s). Ở G4 (alpha bị chậm thêm 5s, latency ~6.2s), slow_call_s=5.0 giúp breaker trip ("slow_call_rate") và failover sang alpha-eu, giảm p95 từ 13.764s (naive) xuống 5.562s. |
| breaker.open_duration_s | 10.0 | Thời gian mở breaker để provider hạ nhiệt và xả sạch hàng đợi quá tải. Sau 10s chuyển sang HALF-OPEN thăm dò. |
| breaker.half_open_max_probes | 2 | Chỉ cho phép tối đa 2 probe đồng thời ở HALF-OPEN để tránh thundering herd đè bẹp provider vừa mới hồi phục. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```
+------------------+---------------------+-------------------+---------------------+-------------------------------+
| Loại lỗi         | Ghi Breaker?        | Retry cùng host?  | Failover ngay?      | Cooldown Provider?            |
+------------------+---------------------+-------------------+---------------------+-------------------------------+
| CLIENT_ERROR     | release permit (No) | Không (GiveUp)    | Không               | Không                         |
| QUOTA_EXHAUSTED  | release permit (No) | Không             | Có (ngay lập tức)   | Có: cooldown 300s             |
| RATE_LIMITED     | release permit (No) | Không             | Có (ngay lập tức)   | Có: cooldown now + retry_after|
| OVERLOADED /     | on_failure (Có)     | Có nếu còn lượt & | Có khi hết lượt thử | Không (do Breaker quản lý)    |
| SERVER_ERROR /   |                     | còn retry budget  | hoặc hết budget     |                               |
| TIMEOUT          |                     | & kịp deadline    |                     |                               |
+------------------+---------------------+-------------------+---------------------+-------------------------------+
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`.
   - Khi có retry budget (ratio=0.1): Ở G2, `amplification = 1.380`, `retry_ratio = 0.058`, `recovery_s = 0.0`, `availability = 99.87%`. Ở G3, `amplification = 1.136`, `retry_ratio = 0.068`, `availability = 100%`.
   - Khi tắt retry budget: Ở G2, `amplification = 1.675`, `retry_ratio = 0.495` (tăng gấp gần 9 lần), `recovery_s = 25.0s`, `availability = 96.66%`. Ở G3, `retry_ratio = 0.159` (vi phạm ngưỡng 0.10 của G3).
   - Khi breaker hoạt động đúng, nó ngắt traffic sớm trước khi lỗi lan rộng, nhưng retry budget vẫn là lớp phòng thủ độc lập thiết yếu. Breaker tính lỗi theo tỷ lệ trượt của từng provider, còn retry budget kiểm soát tổng lượng retry tiêu tốn trên toàn hệ thống. Nếu breaker bị cấu hình sai hoặc gặp lỗi chập chờn, retry budget ngăn chặn hoàn toàn nguy cơ retry storm.

2. **`attempt_timeout_s: 30`.** Chạy G6 (và G4 để đối chứng):
   - Ở G6 (provider A sập hoàn toàn): Khi timeout=30s, `availability = 94.88%` (tụt dưới chuẩn 95%), xuất hiện 109 lượt GiveUp vì lý do `deadline` và 13 `no_provider_available`. Mặc dù `latency_p95_s = 5.315s` có vẻ "nhỏ hơn" mức 5.961s của bản chuẩn (timeout=8s), đây là hiện tượng survivorship bias: các call treo 30s đều bị drop khi chạm turn deadline (30s) và không được tính vào p95 của lượt thành công.
   - Ở G4 (slow region): Availability vẫn đạt 100%, `p95 = 5.608s` (so với 5.562s khi timeout=8s) vì breaker phát hiện `slow_call_s = 5.0s` và trip dựa trên `slow_rate_threshold` thay vì chờ timeout.
   - Timeout "đúng" không được đặt bằng turn deadline (30s) mà phải dựa trên: (p99 latency bình thường + dung sai xử lý) và phải đủ nhỏ để cho phép ít nhất 1-2 lần failover sang provider dự phòng trước khi người dùng hết kiên nhẫn.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Chạy G6:
   - Khi bỏ lọc `supports_tools`, `incompatible_routes` tăng vọt lên **258** lần, kéo `availability` tụt từ 100% xuống **89.18%**.
   - Dưới góc nhìn Product Manager: Việc gửi tool call sang một model không hỗ trợ tool calling (`gamma-mini`) giống như giao một đơn hàng cần thanh toán thẻ cho nhân viên giao hàng không mang máy POS. Khách hàng nhận được câu trả lời vô nghĩa hoặc lỗi hệ thống ngay tại quầy thanh toán, làm đứt gãy 258 giao dịch của người dùng và làm tổn hại trực tiếp đến độ tin cậy của dịch vụ.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.00 | 81 | 128 | 81 | 0 | 14,967.00 $ | 1.123 |
| yours | 1.00 | 66 | 0 | 0 | 0 | 0.00 $ | 0.303 |

Trả lời:
- Vì sao chỉ idempotency key là **chưa đủ**, và chỉ checkpoint cũng **chưa đủ**?
  - Nếu chỉ có idempotency key mà không có checkpoint: Khi worker crash và restart, worker mới không biết các step trước đã thành công hay chưa, buộc phải chạy lại từ đầu lãng phí LLM calls, `wasted_llm_per_crash` cao.
  - Nếu chỉ có checkpoint mà không có idempotency key: Khi gặp sự cố mạng như `ServiceTimeout` lệnh thanh toán đã thành công tại cổng thanh toán nhưng response bị rớt trên đường truyền, việc retry step thanh toán mà không có idempotency key sẽ làm cổng thanh toán trừ tiền lần thứ hai.
- Điều gì xảy ra nếu ghi checkpoint trước khi gọi payment?
  - Nếu ghi checkpoint trước khi gọi payment: Nếu worker crash ngay sau khi ghi checkpoint hoặc call payment thất bại thực sự, worker phục hồi sau đó sẽ đọc thấy checkpoint và ngộ nhận rằng payment đã hoàn thành. Hệ thống sẽ bỏ qua bước thanh toán và nhảy sang gửi email, dẫn đến lost refund khách hàng không nhận được tiền nhưng vẫn nhận email thông báo đã hoàn tiền.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (delay 1m) | page | 27 min |
| provider_outage | page | page (delay 0m) | page | 4 min |
| deploy_blip | none | page (false page) | none | — |
| slow_leak | ticket | none (missed) | ticket | 1337 min |
| degraded_evening | page | page (delay 214m) | page | 291 min |
| night_spike | none | page (false page) | none | — |
| small_bump | none | page (false page) | none | — |

Error budget 30 ngày: đã tiêu 42.95%, còn 57.05%. Quyết định: Không đóng băng release (`freeze_releases = False`) vì ngân sách lỗi còn hơn 57%.

Vì sao `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa?
- Ngân sách lỗi 30 ngày ở SLO 99.5% tương đương với tổng thời gian downtime cho phép là 216 phút ở mức 100% lỗi. Một sự cố nhỏ chỉ kéo dài 20 phút ở mức 12% lỗi thực tế chỉ tiêu tốn: `(20 * 0.12) / (43200 * 0.005) = 2.4 / 216 ≈ 1.1%` error budget của cả tháng.
- Naive alerter chỉ nhìn cửa sổ ngắn 5 phút nên bị giật mình bởi các spike nhỏ này, gây ra 3 false pages và làm mệt mỏi kỹ sư on-call. Burn-rate alerter kết hợp cửa sổ dài (1h / 6h) với cửa sổ ngắn (5m / 30m) nên nhận biết được lượng budget bị đốt là không đáng kể, không gây báo động giả.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào thời điểm t=120s của kịch bản G2, deployment chính `alpha` bị giảm 75% GPU trong 3 phút, làm năng lực xử lý sụt giảm nghiêm trọng. Bản client v1 (NaiveClient) phản ứng bằng cơ chế retry 3 tầng xếp chồng không có jitter, không breaker và không fallback, tạo ra retry storm khổng lồ với 34,475 attempts và đè bẹp hàng đợi của provider. Hậu quả là availability sụt giảm còn 45.60%, và hệ thống rơi vào trạng thái metastable failure kéo dài thêm 155s sau khi GPU của provider đã hồi phục hoàn toàn.

**Ảnh hưởng:**
- Tổng cộng 1,285 lượt user bị thất bại (1,077 lượt thành công trên tổng số 2,362 lượt, availability 45.60%).
- Sự cố kéo dài tổng cộng 335 giây (180 giây suy giảm năng lực phần cứng + 155 giây recovery lag do nghẽn hàng đợi).
- Tiêu tốn 1,285 turn lỗi, nếu tính theo SLO 99.5% trên quy mô tương đương (budget cho 2,362 turns là 11.8 turns lỗi), sự cố này đã vượt quá 10,800% error budget được phân bổ cho khoảng thời gian đó.

**Timeline** (giây ảo — lấy từ CSV):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực xử lý (GPU) | calls_ok giảm từ 7 calls/s (t=120) xuống 2 (t=121) rồi 0 (t=122). attempts_alpha bắt đầu tăng vọt từ 5 lên 11 (t=123). |
| 136 | Hàng đợi alpha tràn, retry storm bùng nổ | attempts_alpha tăng vọt lên 53 req/s (t=136), 122 req/s (t=138), đạt đỉnh 254 req/s (t=143-144). calls_ok = 0. |
| 200 | Hệ thống kẹt cứng trong vòng lặp quá tải | attempts_alpha liên tục duy trì ở mức 120 - 180 req/s. calls_ok hoàn toàn bằng 0, hàng loạt request bị 529 và client_timeout. |
| 300 | alpha hồi phục hoàn toàn phần cứng | GPU phục hồi 100%, nhưng attempts_alpha vẫn ở mức 115 req/s (t=300). calls_ok vẫn là 0. |
| 305 | Bắt đầu xả hàng đợi tồn đọng | attempts_alpha giảm dần về 2 req/s (t=305), nhưng calls_ok vẫn = 0 do server bận xử lý các request cũ đã quá hạn. |
| 360-440 | Làn sóng retry thứ hai từ các turn chưa hết hạn | attempts_alpha tăng lại lên 100-158 req/s (t=362-368), calls_ok tiếp tục bằng 0 suốt từ t=300 đến t=452. |
| 453-455 | Hệ thống bắt đầu phục hồi thực sự | calls_ok đạt 4 (t=453), 4 (t=454), 8 (t=456). Hàng đợi được dọn sạch, attempts_alpha ổn định về 10-15 req/s. Hệ thống chính thức phục hồi tại t=455 (recovery_s = 155s). |

**Trigger vs root cause:**
- **Trigger:** Provider `alpha` bị mất 75% GPU trong 3 phút (t=120 đến t=300), làm giảm số slot phục vụ đồng thời và khiến hàng đợi xử lý bị ứ đọng.
- **Root cause:** Kiến trúc client v1 thiếu khả năng tự bảo vệ: sử dụng 3 tầng retry chồng chéo (tối đa 36 attempt/call) với fixed delay 0.2s, không có exponential backoff, không jitter, không retry budget và không có circuit breaker. Khi provider chậm lại, client liên tục nhồi thêm request retry với tốc độ lên đến 254 req/s (gấp hơn 12 lần rate limit 20 rps).
- **Cơ chế metastable failure:** Tại sao t=300 GPU đã phục hồi mà hệ thống vẫn sập đến t=455? Khi client timeout hoặc người dùng từ bỏ, server vẫn tiếp tục xử lý các request cũ trong hàng đợi FIFO. Các request retry mới liên tục nạp vào đuôi hàng đợi nhanh hơn tốc độ server xử lý. Kết quả là hàng đợi luôn đầy ắp các request "rác" đã hết hạn, tạo thành vòng lặp tự duy trì khiến công suất của server bị lãng phí 100% vào việc xử lý các kết quả không còn ai nhận.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Fixed retry delay (0.2s); 3 tầng retry nhân số lượng attempt lên tối đa 36 lần; không kiểm tra deadline trước khi retry; tiếp tục gửi toàn bộ tải vào provider đang chết mà không có fallback sang alpha-eu hay beta.
- *Giúp ích (trong bản sửa):* Circuit breaker sliding-window phát hiện failure rate >= 50% ngắt ngay mạch sang OPEN; chuyển hướng traffic sang alpha-eu và beta; áp dụng RetryBudget (ratio 0.1) triệt tiêu hoàn toàn retry storm; FullJitterBackoff phân tán tải.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế toàn bộ NaiveClient bằng ResilientClient có Circuit Breaker và Retry Budget trên production | Prevent | On-call / Core Team | Chạy test suite và game day tape G2: amplification <= 1.10, availability >= 99% |
| 2 | Thiết lập hệ thống Multi-window Burn-rate Alerting theo chuẩn SRE Workbook ch. 5 | Detect | SRE Team | Kiểm tra trên tập dữ liệu 30 ngày: phát hiện sự cố lớn trong vòng < 30 phút, 0 false page |
| 3 | Tích hợp Fallback Ladder có lọc khả năng model và quy định deadline-aware timeout (8s) | Mitigate | Agent Platform Team | Kiểm chứng trên G6: 0 incompatible_routes và availability >= 98% khi provider chính sập |

## 7. Một điểm yếu còn lại

**Tình huống:** Khi toàn bộ các provider trong cùng vùng hoặc cùng model family đồng loạt bị suy giảm hiệu năng chẳng hạn cả `alpha`, `alpha-eu` và `beta` đều bị nghẽn mạng xuyên biên giới, traffic sẽ bị dồn dập failover sang `beta`. `beta` có chi phí cao hơn 40% (1.4 cost/call) và rate limit chỉ 16 rps. Khi lượng lớn traffic đổ dồn về `beta`, nó có thể nhanh chóng bị rate limit và cạn kiệt slot, làm tăng đột biến chi phí vận hành (`cost_per_good_turn`) mà vẫn có nguy cơ gây cascade failure sang provider cuối cùng.

**Cách khắc phục:**
1. **Adaptive Concurrency Limits / Load Shedding chủ động:** Cài đặt thuật toán giới hạn số lượng request đồng thời gửi tới từng downstream provider như Vegas hoặc AIMD algorithm. Khi phát hiện downstream bắt đầu tăng latency, client chủ động từ chối bớt các tác vụ không khẩn cấp thay vì cố đẩy sang provider đắt tiền.
2. **Chiến lược Dynamic Cost-aware Routing:** Đưa trọng số chi phí và quota còn lại vào bộ định tuyến `_route()`. Khi failover sang provider đắt đỏ (`beta`), chỉ ưu tiên các turn thanh toán hoặc khách hàng VIP, còn các turn thông thường có thể trả về câu trả lời tĩnh hoặc thông báo bận có kiểm soát để bảo vệ ngân sách.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.592 | 192 | 0 | 2 | 0.955 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.90 | Ở ngưỡng 0.80, có 182 câu trả lời sai do nhận nhầm câu hỏi khác (`wrong_answers_by_reason: {'other_question': 182}`). Ngưỡng 0.90 loại bỏ hoàn toàn lỗi này (`wrong_answers = 0`) trong khi vẫn duy trì cache_hit_ratio tốt (59.2%). Nếu tăng quá cao lên 0.97, hit ratio tụt xuống 49.5% (< 50% ngưỡng yêu cầu). |
| ttl_s | 30 | Giới hạn số lượng câu trả lời cũ khi có thay đổi chính sách âm thầm không có webhook (silent update tại t=480). Với ttl=30s, `outdated_answers = 2` (đạt yêu cầu <= 15). Nếu để ttl=60s outdated là 3, và nếu để ttl=300s outdated vọt lên 95 vi phạm nghiêm trọng. |
| max_stale_s | 300 | Cho phép phục vụ câu trả lời cũ trong tối đa 300s khi tất cả provider sập (outage kéo dài 120s từ t=240 đến t=360). Giúp cứu được 192 lượt FAQ (`stale_served = 192`), nâng `faq_availability` từ 84.24% lên 96.45%. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Chạy C1 với `similarity_threshold` = 0.80 / 0.90 / 0.97:
   - Với threshold = 0.80: `cache_hit_ratio = 58.7%`, `wrong_answers = 182` (100% là `other_question`), `cost/turn = 1.033`, `faq_availability = 85.87%`. Các câu hỏi khác biệt về ngữ nghĩa như "Chính sách đổi trả trong bao lâu?" bị nhầm với "Phí ship đổi trả bao nhiêu?" do vector embedding có độ tương đồng cosine ~0.82-0.85.
   - Với threshold = 0.90: `cache_hit_ratio = 59.2%`, `wrong_answers = 0`, `cost/turn = 0.955`, `faq_availability = 96.45%`. Ngưỡng 0.90 phân biệt chính xác các câu hỏi khác nhau nhưng vẫn nhận diện tốt các câu cùng ý được paraphrase lại.
   - Với threshold = 0.97: `cache_hit_ratio = 49.5%` (không đạt check >= 50%), `wrong_answers = 0`, `cost/turn = 1.020`, `faq_availability = 96.16%`. Ngưỡng quá khắt khe bỏ lỡ nhiều câu hỏi đồng nghĩa hợp lệ.

2. **TTL.** Chạy với `ttl_s` = 30 / 60 / 300:
   - Tại t=480, bảng phí ship thay đổi nhưng phía đối tác không gửi webhook thông báo. Cơ chế invalidation theo event hoàn toàn bất lực trước thay đổi âm thầm này.
   - Khi `ttl_s = 300`, cache tiếp tục trả kết quả cũ sai lệch suốt 5 phút, dẫn đến `outdated_answers = 95`. Khi giảm xuống `ttl_s = 60`, `outdated_answers = 3`, và với `ttl_s = 30`, `outdated_answers = 2` rất an toàn so với mức trần 15. TTL chính là cơ chế phòng vệ cuối cùng để giới hạn thời gian tồn tại của dữ liệu bị sai lệch khi hệ thống phát hiện sự kiện không hoạt động.

3. **Stale-if-error.** Chạy với `max_stale_s: 0`:
   - Khi tắt stale-if-error (`max_stale_s = 0`), `stale_served` tụt về 0. `faq_availability` sụt giảm từ **96.45%** xuống còn **85.29%** (thất bại so với ngưỡng yêu cầu >= 93%). Trong giai đoạn 120s mọi provider đều sập, toàn bộ các request FAQ dù có bản ghi trong cache nhưng đã quá TTL đều bị từ chối phục vụ.

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Nguy cơ rò rỉ dữ liệu (Security/Privacy):* Một sai sót nhỏ trong logic scoping hoặc va chạm embedding giữa các session có thể khiến thông tin đơn hàng, địa chỉ, số tiền hoàn của người này bị trả cho người khác.
  2. *Dữ liệu cá nhân biến động liên tục theo thời gian thực (Highly Dynamic):* Trạng thái đơn hàng, điểm thưởng thay đổi từng phút. Phục vụ dữ liệu cache cũ sẽ khiến người dùng hiểu sai trạng thái giao nhận.
  3. *Tỷ lệ cache hit cực thấp:* Mỗi user chỉ hỏi về đơn hàng của mình 1-2 lần, việc lưu cache không đem lại lợi ích kinh tế hay giảm tải đáng kể mà lại tiêu tốn bộ nhớ và tăng rủi ro bảo mật.

- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  - Không được chỉ dựa vào metric `user_availability` tổng thể vì nó bị che mờ bởi cache hit và stale responses.
  - Cần phải alert dựa trên:
    1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio` hoặc `stale_served_count`):** Số lượng request phải dùng đến cơ chế stale-if-error tăng vọt là chỉ dấu trực tiếp cho thấy upstream provider đang chết.
    2. **Tỷ lệ lỗi trực tiếp từ provider (`provider_error_rate` / `circuit_breaker_state`):** Giám sát trạng thái circuit breaker của từng provider OPEN và số lượng 5xx/529 trả về từ các call bắt buộc phải qua provider như tool calls hoặc cache misses.
