# Arm AI Optimization Challenge — Project Plan & Results Log

**Project:** Parallel speculative decoding across heterogeneous cores on mobile
**Track:** Mobile AI
**Deadline:** Aug 14, 2026, 4:00pm PDT
**Status:** Ongoing progress towards Plan A

---

## Setup

| Item | Value |
|---|---|
| Device | Pixel 9 (Tensor G4) |
| Core layout | CPUs 0–3: efficiency @ 1.95 GHz · CPUs 4–6: medium @ 2.6 GHz · CPU 7: prime @ 3.105 GHz |
| Environment | Termux, SSH over USB |
| llama.cpp commit (PINNED) | `00fa7cb284cbf133fc426733bd64238a3588a33e` |
| Big model | Qwen2.5-3B-Instruct, Q4_0 (1.86 GiB, 3.4B params) |
| Draft model | Qwen2.5-0.5B-Instruct, Q4_0 (403 MiB, 630M actual params) |
| Plain-baseline method | `llama-speculative` with `--spec-draft-n-max 0` (same harness as spec runs). Note: `--spec-type none` is broken on this commit — it still drafts. |

---

## Verified numbers (trustworthy — thermally controlled or highly stable)

### Headline A/B result (same harness, 120s cooldowns, interleaved, temp 0, n=192, draft n_max=3, big pinned 4–7, draft pinned 0–3)

| Prompt type | Plain (t/s) | Speculative (t/s) | Speedup |
|---|---|---|---|
| Prose | 7.77 / 7.67 / 7.66 | 11.98 / 11.63 / 11.83 | **~1.53x** |
| Code | 7.39 / 7.36 / 7.37 | 14.31 / 13.50 / 14.13 | **~1.9x** |

Spread within condition: < 3%. These are the numbers to beat.

### Component measurements

| Measurement | Result |
|---|---|
| Drafter solo, cores 0–3, Q4_0 | 23.5–23.7 t/s, extremely stable (±0.3) |
| Drafter thread scaling (cores 0–3) | 1t: 9.8 · 2t: 15.7 · 3t: 20.3 · 4t: 22.7 |
| Drafter under contention (big model running) | 17.2 t/s (−28% vs solo) |
| Big model under contention | unaffected (~6.0 vs ~5.5 solo, within noise) |
| Big model, cores 4–7, cold | ~9.5 t/s |
| Big model, sustained/warm | sags to ~5.1 t/s (thermal) |
| Q4_0 vs Q4_K (drafter) | 13.0 → 23.7 t/s (+82%) — Q4_0 hits Arm-optimized kernels |
| Q4_0 vs Q4_K (big model) | no change — memory-bandwidth-bound, not compute-bound |

### Acceptance rates (draft n_max=3, temp 0)

| Content | Acceptance |
|---|---|
| Python code | **82.6%** ← demo scenario |
| Hash map explanation | 58.6% |
| Prose (raw prompt) | ~50% |
| Chat-formatted prose | 37.3% (formatting made it WORSE — hypothesis falsified) |
| History essay | 38.2% |

### Draft length sweep (pre-thermal-control — treat as noisy, re-run if it matters)

n=1: 10.7 t/s (62.7% acc) · n=2: 6.9 (54.3%) · n=3: 11.8 (49.8%) · n=4: 8.8 (41.7%)
n=3 provisionally best; the n=2 dip is probably contamination.
