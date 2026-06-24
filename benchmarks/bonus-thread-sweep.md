# Bonus — Thread sweep

Model: `Llama-3.2-3B-Instruct-Q4_K_M.gguf`  ·  GPU layers: `99`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 55.7 |
| 2 | 58.6 |
| 5 | 58.6 |
| 10 | 53.7 |
| 20 | 45.7 |

**Best**: `-t 5` at 58.6 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
