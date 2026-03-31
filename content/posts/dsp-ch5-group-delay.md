+++
title = "[Ch5.1] What is General Linear Phase (Group Delay) of a filter?"
date = "2020-04-19"
series = ["DSP Learning Path"]
categories = ["tech"]
tags = ["dsp", "matlab", "filter", "group-delay", "linear-phase"]
+++

> **DSP_lab code:** [Ch5 — Transform Analysis of LTI Systems](https://github.com/abc1199281/DSP_lab/tree/master/Ch5_TransformAnalysisOfLTISystem)

## Overview

This article explores the concept of **general linear phase (GLP)** in digital signal processing filters, emphasizing its relationship to constant group delay.

## Key Concepts

### General Linear Phase Definition

General Linear Phase (GLP) means maintaining a **constant group delay**. Group delay is defined as the negative derivative of a filter's phase response:

$$\tau(\omega) = -\frac{d\phi(\omega)}{d\omega}$$

A linear phase response necessarily produces constant group delay, and vice versa.

### What Does Group Delay Mean?

> "The group delay of a frequency represents the delay (in samples) that a filter applies to that frequency component."

Without constant group delay, a filter introduces different delays for different frequency components — potentially reordering signal components in time. For example, a musical phrase "do re mi" could come out as "re mi do".

## Practical Example

Using an IIR low-pass filter with three input frequency components:

| Component | Frequency | Result |
|---|---|---|
| A | 0.8π | Filtered out (stop band) |
| B | 0.4π | ~6.39 sample delay |
| C | 0.2π | ~153 sample delay |

The 0.2π component arrives significantly later than the 0.4π component — confirming the non-constant group delay of an IIR filter.

## Key Takeaways

1. Linear phase ↔ constant group delay (they are equivalent)
2. GLP FIR filters prevent frequency-dependent reordering of signal components
3. IIR filters cannot achieve true GLP, but may achieve lower overall delay while meeting specifications

## Reference

Alan V. Oppenheim and Ronald W. Schafer. *Discrete-Time Signal Processing* (3rd ed., 2009), Chapter 5.
