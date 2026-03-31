+++
title = "[Ch7] Why is it a bad idea to filter by zeroing out FFT bins?"
date = "2019-04-22"
series = ["DSP Learning Path"]
categories = ["tech"]
tags = ["dsp", "matlab", "fft", "filter-design", "windowing", "gibbs"]
+++

> **DSP_lab code:** [Ch7 — Filter Design Techniques](https://github.com/abc1199281/DSP_lab/tree/master/Ch7)

## The Question

DFT/IDFT via FFT has complexity O(n log n), while convolution is O(n²). So why not just zero out unwanted FFT bins to filter a signal? It seems simpler and cheaper.

The answer lies in the **windowing effect**.

## The Deception

When you zero out FFT bins to create an ideal bandpass filter, you are implicitly applying a **rectangular window** to the impulse response. At first glance — using a standard FFT size matching the impulse response length — the frequency response looks ideal (Figure 1). Clean passbands, sharp cutoff.

This observation is **misleading**.

## The Revelation: Gibbs Phenomenon

By using **zero-padding** to increase DFT resolution, the hidden problem emerges: ripples in the frequency domain known as the **Gibbs phenomenon** (Figure 2).

These oscillations occur because:
- The DFT samples the true DTFT at discrete frequencies
- Zero-padding increases sampling density, revealing artifacts that were always present but hidden

## Three Critical Points

1. **Window size and DFT order are independent** — these must not be conflated
2. **Window properties dominate the DTFT shape** — a rectangular window produces a wide main lobe and large sidelobes
3. **DFT samples the DTFT** — higher DFT orders provide denser spectral sampling, revealing the true (artifact-ridden) frequency response

## Bottom Line

Zeroing FFT bins applies a rectangular window, which has the worst possible sidelobe characteristics. Use proper FIR filter design methods (windowed sinc, Parks-McClellan, etc.) for controlled frequency response.

## Reference

Alan V. Oppenheim and Ronald W. Schafer. *Discrete-Time Signal Processing* (3rd ed., 2009), Chapters 7.6 and 10.1–10.2.
