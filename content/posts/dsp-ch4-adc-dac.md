+++
title = "[Ch4 ADC/DAC] How to simulate ADC/DAC process in MATLAB?"
date = "2020-04-17"
series = ["DSP Learning Path"]
categories = ["tech"]
tags = ["dsp", "matlab", "sampling", "adc", "dac"]
+++

> **DSP_lab code:** [Ch4 — Sampling of Continuous Signals](https://github.com/abc1199281/DSP_lab/tree/master/Ch4_SamplingOfContinuousSignal)

## Overview

![Simulated DAC process: ADC stage (sampling + quantization) followed by DAC stage (ZOH + reconstruction filter)](/images/posts/dsp-ch4/1.png)

This article explains how to bridge sampling theory with practical MATLAB implementation for analog-to-digital (ADC) and digital-to-analog (DAC) conversion, specifically addressing reconstruction of signals sampled near the Nyquist rate.

## Key Concepts

### Ideal vs. Simulated Reconstruction

**Ideal DAC Process:**
Consists of a zero-order hold circuit followed by a reconstruction filter. The zero-order hold's frequency response is not flat, requiring a reconstruction filter to correct distortions.

**MATLAB Simulation Approach:**
Rather than simulating a zero-order hold explicitly, the practical approach involves upsampling followed by an ideal low-pass filter.

## Implementation Strategy

The provided MATLAB code demonstrates:

1. **Sampling a simulated 2 Hz analog signal** (1500 Hz sample rate) down to 5 Hz (near Nyquist rate)
2. **Reconstructing the signal** back to the original sample rate

### Code Structure

**Parameters:**
- Analog sample rate: 1500 Hz
- Digital sample rate: 5 Hz (Nyquist = 2 Hz × 2)
- Signal components: 1 Hz and 2 Hz cosine waves

**ADC Stage:**

```matlab
x_s = downsample(x_a, analog_fps/digital_fps);  % Sampling
x_d = x_s;                                        % Skip quantization for basic case
```

**DAC Stage:**

```matlab
x_up = upsample(x_d, analog_fps/digital_fps);    % Upsampling
h = intfilt(analog_fps/digital_fps, 4, 0.9);     % Interpolation filter
x_r = filter(h, 1, x_up);                        % Reconstruction
```

## Critical Parameter: Filter Cutoff Ratio

The `intfilt` parameter of `0.9` represents the ratio relative to Nyquist frequency:

| Value | Effect |
|---|---|
| `0.9` | Filters out frequencies above 2.25 Hz (safer reconstruction) |
| `1.0` | Preserves up to Nyquist rate (theoretically optimal, rarely practical) |
| `0.5` | Enhances SNR via oversampling when bandwidth is known |

## Group Delay Compensation

The code addresses filter group delay by:
- Removing initial samples: `x_r(1:floor(mean(grpdelay(h)))) = []`
- Padding with zeros to maintain signal length

## Results

With default parameters (5 Hz sampling), reconstruction shows noticeable approximation error. Improvement occurs by:
- Increasing digital sampling rate (e.g., 30 Hz instead of 5 Hz)
- Adjusting filter cutoff ratio to `0.5` for faster filter convergence

## Reference

Alan V. Oppenheim and Ronald W. Schafer. *Discrete-Time Signal Processing* (3rd ed., 2009), Chapter 4.8.3.
