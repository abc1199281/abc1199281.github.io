+++
title = "TurboQuant Through a DSP Lens: What Signal Processing Reveals About KV Cache Quantization"
date = "2026-03-31"
categories = ["tech"]
tags = ["dsp", "quantization", "llm", "kv-cache", "signal-processing", "transformer"]
+++

*A semiconductor performance modelling engineer's analysis of Google's TurboQuant, with empirical measurements that reveal surprising noise structure in quantized KV caches.*

---

Google Research released TurboQuant in March 2026 — a two-stage vector quantization algorithm that compresses LLM key-value caches to 3 bits per coordinate with near-zero accuracy loss. Within a week, the open-source community produced over a dozen implementations across PyTorch, llama.cpp, vLLM, and Triton.

But every one of those implementations approaches TurboQuant from an ML perspective: distortion bounds, information theory, rate-distortion tradeoffs. Nobody has analyzed it through the lens of **signal processing and hardware engineering** — a field that has been solving quantization problems for decades.

I'm a Staff Model Engineer at a semiconductor company, building performance models for image and display processing pipelines. I also hold a PhD in computer vision and an EE degree with hands-on ADC and DSP experience. When I read the TurboQuant paper, I didn't see a novel ML technique. I saw transform coding, Lloyd-Max quantization, and subtractive dithering — concepts from DSP textbooks, applied to a new domain.

This post reframes TurboQuant using DSP tools, then shares empirical measurements on Qwen2.5-1.5B that reveal something the paper doesn't discuss: **the quantization noise isn't white — it's colored, and it's worst exactly where it hurts most.**

All code and notebooks are available at [https://github.com/abc1199281/AI_Labs](https://github.com/abc1199281/AI_Labs).

---

## TurboQuant in 30 Seconds

When an LLM generates text, it stores a Key and Value vector for every token in every layer — the KV cache. This cache grows linearly with context length and becomes the dominant memory bottleneck for long-context inference.

TurboQuant compresses the KV cache in two stages:

1. **Rotate** each vector by a random orthogonal matrix, making every coordinate follow a predictable Gaussian distribution. Then apply an optimal **Lloyd-Max scalar quantizer** per coordinate.
2. **Correct** the inner product bias using a 1-bit **QJL (Quantized Johnson-Lindenstrauss)** transform on the residual.

At 3 bits per coordinate, this achieves roughly 5× compression with 99.5% attention fidelity. For the full details, see the [original paper (arXiv:2504.19874)](https://arxiv.org/abs/2504.19874).

What follows is a different way to understand *why* it works, and where it doesn't.

---

## The DSP Reinterpretation

### It's 6 dB per bit. That's it.

The classic DSP rule for uniform quantization: SQNR ≈ 6.02b + 1.76 dB, where b is the number of bits.

TurboQuant's MSE distortion from Theorem 1 is D_mse ≈ C / 4^b. Converting to SQNR:

```
SQNR = 10·log₁₀(signal_power / D_mse) = 20b·log₁₀(2) - 10·log₁₀(C) ≈ 6.02b - 10·log₁₀(C)
```

The distortion rate **is** the 6 dB/bit rule, with a different constant term. TurboQuant uses Lloyd-Max (non-uniform) quantization on a Gaussian distribution, which gains roughly 1.4 dB over uniform quantization — again, a classical result from any DSP textbook. The paper doesn't mention this connection.

### 4.3 dB from Shannon: One sentence that summarizes the entire paper

The paper's proudest claim: TurboQuant's distortion is within a factor of 2.7× of the information-theoretic lower bound.

In decibels: 10·log₁₀(2.7) ≈ **4.3 dB from the Shannon limit.**

For anyone who has designed ADCs or communication systems, this single number conveys the paper's contribution instantly. Many practical ADC designs operate 3-6 dB from the Shannon limit. TurboQuant achieves comparable efficiency for vector quantization.

### Rotation = Transform Coding

TurboQuant's random orthogonal rotation is not a new idea in DSP. It's **transform coding**:

| System | Transform | Then what |
|---|---|---|
| JPEG | DCT | Quantize each coefficient independently |
| Audio codecs (MP3/AAC) | MDCT | Quantize each frequency bin independently |
| TurboQuant | Random orthogonal matrix | Quantize each coordinate independently |

The purpose is identical: convert correlated input into decorrelated/independent representation, so that independent scalar quantization becomes near-optimal. JPEG uses DCT because natural image statistics are known. TurboQuant uses random rotation because it needs to be **data-oblivious** — the rotation guarantees a Gaussian distribution regardless of the input vector.

### QJL = Subtractive Dithering (and why the community found it hurts)

TurboQuant's two-stage architecture — MSE quantizer followed by a 1-bit QJL correction on the residual — has a direct analog in DAC design: **subtractive dithering**.

Classical dithering: quantize, compute the residual, send a representation of the residual to the receiver so it can cancel the quantization bias. The cost is a higher noise floor (increased variance).

QJL does exactly this. It projects the quantization residual through a random matrix and stores only the sign (1 bit), producing an unbiased inner product estimator. The cost is increased variance.

The llama.cpp community independently discovered that at the same total bit budget, k-bit MSE-only outperforms (k-1)-bit MSE + 1-bit QJL. In dithering theory, this is a known tradeoff: **dithering eliminates harmonic distortion (bias) but raises the noise floor (variance). When the downstream system is more sensitive to noise floor than to harmonic distortion, skipping dither is better.**

Attention's softmax is exactly such a system — a single high-variance outlier in the attention scores can distort the entire distribution. The bias from MSE-only quantization is small and systematic; the variance from QJL is random and potentially catastrophic. The dithering framework predicts this, and the empirical results confirm it.

---

## What I Measured: Three Findings Nobody Has Reported

I ran a systematic measurement campaign on **Qwen2.5-1.5B-Instruct** (28 layers, 2 KV heads, 12 Q heads, head_dim=128) using WikiText-2 and Python code as inputs. The goal was to characterize the quantization noise structure using DSP metrics. Full notebooks are in the repo.

### Finding 1: SQNR Baseline — TurboQuant is remarkably uniform

At 3-bit quantization across all 28 layers:

- **K average SQNR: 35.8 dB**
- **V average SQNR: 35.8 dB**
- Spread: only 34.7–36.6 dB (< 2 dB across all layers)
- K and V are nearly identical — no need for asymmetric K/V bit allocation on this model

The 6 dB/bit relationship holds: 3 bits × 6.02 ≈ 18 dB of quantization SQNR, plus the Lloyd-Max gain and the signal power contribution, lands us at ~36 dB. The Shannon gap of ≤ 2.7× (4.3 dB) was confirmed on 100K DBpedia vectors at d=1536.

**Takeaway**: TurboQuant is already operating close to the theoretical optimum. There isn't a lot of room for improvement at the scalar quantizer level.

![SQNR vs bit-width](https://raw.githubusercontent.com/abc1199281/AI_Labs/main/experiments/turboquant-dsp/figures/sqnr-tooling/figure3_reproduction.png)
*D_prod and D_mse vs bit-width, reproducing the paper's Figure 3. The 6 dB/bit slope is clearly visible.*

### Finding 2: The Noise Isn't White — Colored Noise in Early Layers

This is the most interesting finding. I defined an **isotropy ratio** to measure whether quantization error is uniformly distributed across directions or biased toward the query direction:

```
isotropy_ratio = ||P_q(e)||² / (||e||² / d)

= 1.0 → error is isotropic (uniformly distributed across all directions)
> 1.0 → error is biased TOWARD the query direction (harmful for attention)
< 1.0 → error naturally avoids the query direction (beneficial)
```

The results:

| Layer group | Isotropy ratio | Interpretation |
|---|---|---|
| Layer 0 | ~1.96 | Error is nearly 2× concentrated in the Q direction |
| Layers 1–9 | ~1.4 | Significant Q-direction bias |
| Layers 10–27 | ~0.97–1.17 | Near-isotropic |

![Per-layer isotropy ratio](https://raw.githubusercontent.com/abc1199281/AI_Labs/main/experiments/turboquant-dsp/figures/isotropy/isotropy_ratio.png)
*Isotropy ratio (mean ± std) across 28 layers. Layer 0 is a clear outlier.*

**In DSP terms: the quantization noise is colored, not white. And the noise spectral density peaks exactly in the "signal band" — the query direction that attention cares about.**

This is the worst possible scenario, analogous to an ADC whose noise figure is highest at the signal frequency.

**Why does this happen?** K and Q are both linear projections of the same hidden state: K = W_k·h, Q = W_q·h. When h has strong anisotropy (common in early layers where embeddings are fresh and layer norm hasn't fully normalized), the dominant directions of K and Q are correlated — both are driven by h's principal components.

TurboQuant's rotation is supposed to eliminate this anisotropy. And it does — asymptotically. But at d=128, the convergence is incomplete. I verified this in Notebook 01: at d=1536 the Beta distribution after rotation nearly perfectly matches a Gaussian, but at d=128 there's a visible gap. The residual anisotropy of the input vector "leaks through" the finite-dimensional rotation, and because K and Q share their anisotropy (via h), the quantization error correlates with Q.

**Implication**: Any KV cache quantization method that assumes isotropic error (including TurboQuant's theoretical analysis) will **underestimate** the impact on attention accuracy in early layers by up to a factor of 2.

### Finding 3: Video-Style Temporal Prediction Is a Dead End

Coming from a signal processing background, my first instinct was: adjacent tokens' K vectors should be similar (temporal redundancy), so store deltas like video compression (I-frames and P-frames). I defined the **Delta Variance Ratio** (DVR):

```
DVR = Var(K[t] - K[t-1]) / Var(K[t])

< 0.5 → delta coding saves at least 3 dB (~0.5 bits)
< 0.25 → saves 6 dB (~1 bit)
≈ 1.0 → no temporal redundancy to exploit
> 1.0 → delta is WORSE than the original (don't bother)
```

The results:

| Input | Average DVR | Layers with DVR < 0.5 |
|---|---|---|
| WikiText-2 | 1.16 | 0 / 28 |
| Python code | 1.25 | 0 / 28 |
| **Overall** | **1.21** | **0 / 28** |

![Per-layer DVR](https://raw.githubusercontent.com/abc1199281/AI_Labs/main/experiments/turboquant-dsp/figures/kvcache-stats/delta_var_ratio.png)
*Delta Variance Ratio across all 28 layers. The 0.5 threshold (dashed line) is never crossed.*

**DVR > 1 means the delta between adjacent tokens is MORE variable than the tokens themselves.** Video-style DPCM (Differential Pulse-Code Modulation) is not just unhelpful — it's actively worse than storing the originals.

But wait — the cosine similarity between adjacent K vectors is about 0.8. How can similarity be high while delta variance is also high?

**The answer is the curse of dimensionality.** In d=128 dimensional space, two vectors with cosine similarity 0.8 have an angle of about 37°. That sounds small, but the "volume" of the difference cone is enormous in high dimensions. A 20% directional difference at d=128 translates to a delta with magnitude comparable to the vectors themselves.

This is a concrete, empirically verified example of why intuitions from low-dimensional signal processing (audio: 1D, video: 2D spatial) can fail catastrophically in the high-dimensional setting of transformer hidden states.

**Takeaway for practitioners**: Don't attempt inter-token KV cache compression. The temporal redundancy that exists in token sequences does not translate to vector-level redundancy at d=128. Every existing KV cache quantization paper (TurboQuant, KIVI, KVQuant) is correct to treat each token independently.

### Bonus: Noise Shaping Ceiling

Borrowing from sigma-delta ADC theory, the theoretical ceiling for noise shaping in d=128 is:

```
Theoretical ceiling: 10·log₁₀(128) ≈ 21.07 dB
Measured ceiling (accounting for isotropy ratio): 20.02 dB
Gap: 1.06 dB
```

A 20 dB ceiling means that a hypothetical perfect noise shaping could make 3-bit quantization perform like ~6.5-bit in terms of attention accuracy. But there are fundamental barriers to realizing this — attention patterns are not ideal low-pass filters (they're often spiky), and the feedback mechanisms that enable noise shaping introduce sequential dependencies that conflict with GPU parallelism.

---

## Practical Takeaways

**For inference system engineers:**

TurboQuant at 3-bit is already within 4.3 dB of the Shannon limit. The scalar quantizer is not the bottleneck. If you need better quality, increasing to 4-bit (gaining ~6 dB) is more practical than any clever noise shaping scheme at 3-bit.

**For researchers working on KV cache quantization:**

The isotropy ratio is a useful diagnostic tool. If you're developing a new quantization method, check whether your quantization error is isotropic or biased toward the query direction. Early layers deserve special attention — they consistently show the worst noise characteristics.

The DVR measurement should be a standard gate before attempting any temporal/delta compression scheme. At d=128, the answer is consistently "don't bother."

**For hardware architects designing NPU/GPU for LLM inference:**

The colored noise finding suggests that future quantization schemes may benefit from per-layer or per-head bit allocation. A hardware design that supports mixed-precision KV cache (e.g., 4-bit for early layers, 2-bit for deep layers) could capture significant efficiency gains. The SQNR framework directly maps to hardware design parameters — memory bandwidth requirements scale with bit-width, and the SQNR tells you exactly what quality you're trading.

---

## Open Questions and Future Directions

**Can we fix the colored noise without calibration?** The isotropy ratio > 1 in early layers is a systematic bias caused by finite-dimensional rotation. One direction is decode-time correction: a per-coordinate Wiener filter that downweights high-noise coordinates, using analytically computed bin-variance statistics. This preserves the data-oblivious property while reducing the effective noise in the query direction. My estimate: 1-3 dB gain in early layers, which may or may not be enough to matter.

**Is subband bit allocation worth the calibration cost?** PCA on query vectors shows effective rank r/d = 0.31–0.36 in early-to-mid layers, suggesting 4–5 dB of potential gain from query-aware bit allocation. But this requires calibration data, sacrificing TurboQuant's key selling point — data-oblivious operation. Whether this tradeoff is worthwhile depends on the deployment context.

**Can attention-aware noise shaping break through the ceiling?** The 20 dB noise shaping ceiling exists but is guarded by a fundamental barrier: attention is not a low-pass filter. Spiky attention patterns (attention sinks, needle-in-haystack retrieval) pass high-frequency quantization noise without attenuation. Solving this requires rethinking the sigma-delta analogy — perhaps block-wise noise shaping or adaptive approaches that respond to the attention pattern itself.

**Why does d=128 matter so much?** Many of the limitations I found trace back to the same root cause: d=128 is not large enough for the asymptotic guarantees to fully kick in. As models evolve toward larger head dimensions (some architectures already use d=256), the isotropy ratio should improve and the noise should become more white. Measuring how these findings change with d is an important next step.

---

## About

I'm a Staff Model Engineer at Arm in the UK, building cycle-accurate performance models for image and display processing pipelines. My background is in semiconductor verification (SystemC/SystemVerilog), computer vision (PhD), and electrical engineering (NCTU, Taiwan — with PCB design and ADC lab experience). I'm interested in bridging semiconductor performance modelling methodology with AI inference optimization.

If you found this analysis useful, I'd love to hear from you — especially if you work on inference systems, hardware for LLM, or quantization research. Find me on [LinkedIn/Twitter/GitHub].

---

## References

1. Zandieh, A., Daliri, M., Hadian, M., & Mirrokni, V. (2025). TurboQuant: Online Vector Quantization with Near-optimal Distortion Rate. ICLR 2026. [arXiv:2504.19874](https://arxiv.org/abs/2504.19874)
2. Zandieh, A., Han, I., Daliri, M., & Karbasi, A. (2024). QJL: 1-Bit Quantized JL Transform for KV Cache Quantization with Zero Overhead. AAAI 2025. [arXiv:2406.03482](https://arxiv.org/abs/2406.03482)
3. Zandieh, A., Hadian, M., & Mirrokni, V. (2025). PolarQuant: Quantizing KV Caches with Polar Transformation. AISTATS 2026. [arXiv:2502.02617](https://arxiv.org/abs/2502.02617)
4. ggml-org/llama.cpp Discussion #20969: TurboQuant — Extreme KV Cache Quantization. [Link](https://github.com/ggml-org/llama.cpp/discussions/20969)
5. scos-lab/turboquant: Reference implementation with K/V ratio engineering insights. [Link](https://github.com/scos-lab/turboquant)

---

*This analysis was conducted as a side project. All measurements were run on Google Colab (free tier, CPU) using Qwen2.5-1.5B-Instruct. The three Jupyter notebooks and all figures are available in the [companion repository](https://github.com/abc1199281/AI_Labs).*
