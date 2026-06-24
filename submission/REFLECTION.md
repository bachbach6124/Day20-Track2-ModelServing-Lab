# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Đào Xuân Bách
**Cohort:** A20-K2
**Ngày submit:** 2026-06-24

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** macOS (Darwin 25.4.0)
- **CPU:** Apple M1 Pro
- **Cores:** 10 physical / 10 logical
- **CPU extensions:** arm64, neon
- **RAM:** 16.0 GB
- **Accelerator:** Apple Metal
- **llama.cpp backend đã chọn:** Metal
- **Recommended model tier:** Llama-3.2-3B-Instruct (Q4_K_M)

**Setup story** (≤ 80 chữ):
Không gặp lỗi lớn trên macOS. Tôi đã tự động chạy script setup của macOS (`make setup`), cài đặt các dependencies vào môi trường ảo, build thư viện llama-cpp-python tích hợp Metal backend để tối ưu hóa hiệu năng, và tải các mô hình Q4_K_M và Q2_K thành công qua Hugging Face CLI.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| Llama-3.2-3B-Instruct-Q4_K_M.gguf | 11796 | 68 / 173 | 18.7 / 19.7 | 1252 / 1366 / 1392 | 53.6 |
| Llama-3.2-3B-Instruct-Q2_K.gguf   | 971 | 64 / 142 | 17.3 / 17.9 | 1159 / 1240 / 1240 | 57.8 |

**Một quan sát** (≤ 50 chữ):
Q2_K load nhanh hơn đáng kể (~12x) và có tốc độ decode nhỉnh hơn chút ít (~8%). Tuy nhiên, Q4_K_M cho kết quả đầu ra mạch lạc và chính xác hơn nhiều cho các prompt phức tạp. Đánh đổi thêm dung lượng RAM và thời gian load cho độ chính xác của Q4_K_M hoàn toàn xứng đáng.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 1.03 | 7800 | 11000 | 12000 | 0 (0.00%) |
| 50 | 0.98 | 15000 | 30000 | 31000 | 0 (0.00%) |

**Batching observation** (từ `record-metrics.py`):
Peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` ở concurrency 50 là ~3.74, nghĩa là hệ thống đang tích cực tận dụng chế độ continuous batching của llama-server để xử lý đồng thời tới gần 4 request cùng lúc trong một lượt decode thay vì xếp hàng tuần tự, giúp duy trì RPS ổn định (~0.98 vs ~1.03) dù số lượng người dùng đồng thời tăng lên gấp 5 lần.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory toy data
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS in-memory mock

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 0.0 ms (stubbed)
- retrieve: 0.0 ms (toy in-memory search)
- llama-server: 3640.5 ms (cho câu hỏi dài/trả lời đầy đủ)

**Reflection** (≤ 60 chữ):
Bottleneck hoàn toàn nằm ở llama-server (chiếm >99.9% thời gian phản hồi) vì prefill và decode LLM tốn rất nhiều tài nguyên tính toán của GPU/CPU. Các bước retrieve và embed diễn ra trên bộ nhớ in-memory cực nhanh, đúng như kỳ vọng với quy mô nhỏ.

---

## 5. Bonus — The single change that mattered most

**Change:** Build llama.cpp trực tiếp từ source với Metal backend hỗ trợ phần cứng Apple Silicon (Metal GPU acceleration qua `-DGGML_METAL=on`).

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: (chạy CPU thuần túy) ~4.5 tok/s
after:  (Metal GPU offload) ~53.6 tok/s
speedup: ~11.9×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Bằng việc offload toàn bộ các layer của mô hình (99 layers qua `-ngl 99`) lên GPU thông qua Metal backend, chúng ta giải quyết trực tiếp bottleneck lớn nhất của LLM serving: băng thông bộ nhớ (memory bandwidth). CPU thông thường phải chia sẻ bus bộ nhớ hệ thống với nhiều tác vụ khác và không tối ưu cho tính toán ma trận song song quy mô lớn. 

Unified memory trên chip Apple M1 Pro kết hợp với lõi GPU Metal cung cấp băng thông cực lớn lên tới 200 GB/s. Điều này cho phép nạp các trọng số của mô hình vào GPU nhanh hơn gấp 12 lần so với việc tính toán tuần tự trên lõi CPU, đưa tốc độ sinh từ 4.5 tok/s lên mức thời gian thực trên 53 tok/s.

---

## 6. (Optional) Điều ngạc nhiên nhất

Sự hiệu quả đáng kinh ngạc của tính năng continuous batching trên `llama-server`. Mặc dù tăng tải từ 10 lên 50 user (gấp 5 lần), số request thất bại vẫn là 0% và thông lượng RPS tổng thể gần như giữ nguyên, chỉ có độ trễ là tăng tuyến tính do dung lượng KV cache chia sẻ.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS
