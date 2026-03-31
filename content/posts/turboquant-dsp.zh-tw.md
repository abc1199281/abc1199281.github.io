+++
title = "TurboQuant 的 DSP 視角：訊號處理如何揭示 KV Cache 量化的本質"
date = "2026-03-31"
categories = ["tech"]
tags = ["dsp", "quantization", "llm", "kv-cache", "signal-processing", "transformer"]
+++

*一位半導體效能建模工程師對 Google TurboQuant 的分析，以及揭示量化 KV Cache 中令人意外的噪聲結構的實測數據。*

---

Google Research 在 2026 年 3 月發布了 TurboQuant — 一種兩階段向量量化演算法，能以每座標 3 bits 的壓縮率壓縮 LLM 的 key-value cache，且精度損失幾乎為零。一週之內，開源社群就在 PyTorch、llama.cpp、vLLM 和 Triton 上產出了超過十幾個實作版本。

但這些實作全都從 ML 的角度切入：失真邊界、資訊理論、rate-distortion 取捨。沒有人從**訊號處理與硬體工程**的角度分析它——而這個領域已解決量化問題數十年。

我是一位半導體公司的 Staff Model Engineer，專責建構影像與顯示處理流程的效能模型。我同時擁有電腦視覺博士學位，以及有實際 ADC 和 DSP 實驗室經驗的電機工程背景。當我閱讀 TurboQuant 論文時，我看到的不是新穎的 ML 技術，而是變換編碼（transform coding）、Lloyd-Max 量化器以及減法式抖動（subtractive dithering）——這些都是 DSP 教科書裡的概念，只是應用在新的領域。

本文用 DSP 工具重新詮釋 TurboQuant，並分享在 Qwen2.5-1.5B 上進行的實測結果——揭示了論文中未曾討論的事實：**量化噪聲並非白噪聲，它是有色的，而且恰好在最關鍵的地方最嚴重。**

所有程式碼與 notebooks 均收錄於 [https://github.com/abc1199281/AI_Labs](https://github.com/abc1199281/AI_Labs)。

---

## TurboQuant 30 秒速覽

LLM 在生成文字時，會為每一層的每個 token 儲存 Key 和 Value 向量——即 KV cache。這個 cache 隨 context 長度線性增長，成為長 context 推論的主要記憶體瓶頸。

TurboQuant 以兩個階段壓縮 KV cache：

1. **旋轉**：以隨機正交矩陣對每個向量做旋轉，使每個座標都服從可預測的 Gaussian 分布，再對每個座標套用最佳化的 **Lloyd-Max 純量量化器**。
2. **校正**：利用 1-bit 的 **QJL（Quantized Johnson-Lindenstrauss）**變換對殘差做內積偏差修正。

在每座標 3 bits 下，可達到約 5 倍壓縮率，且 attention 保真度達 99.5%。完整細節請參閱[原始論文（arXiv:2504.19874）](https://arxiv.org/abs/2504.19874)。

以下是理解其*為何*有效——以及何時失效——的另一種方式。

---

## DSP 重新詮釋

### 就是每 bit 6 dB，如此而已

均勻量化的經典 DSP 法則：SQNR ≈ 6.02b + 1.76 dB，其中 b 為位元數。

TurboQuant 定理 1 的 MSE 失真為 D_mse ≈ C / 4^b。轉換為 SQNR：

```
SQNR = 10·log₁₀(signal_power / D_mse) = 20b·log₁₀(2) - 10·log₁₀(C) ≈ 6.02b - 10·log₁₀(C)
```

這個失真率**就是** 6 dB/bit 法則，只是常數項不同。TurboQuant 在 Gaussian 分布上使用 Lloyd-Max（非均勻）量化，比均勻量化多增益約 1.4 dB——這同樣是任何 DSP 教科書的經典結果，論文並未提及這個連結。

### 距 Shannon 4.3 dB：一句話總結整篇論文

論文最自豪的宣稱：TurboQuant 的失真在資訊理論下界 2.7 倍以內。

換算為分貝：10·log₁₀(2.7) ≈ **距 Shannon 極限 4.3 dB。**

對於曾設計 ADC 或通訊系統的人來說，這個數字立即傳達了論文的貢獻。許多實用 ADC 設計距 Shannon 極限操作於 3–6 dB。TurboQuant 在向量量化上達到了相當的效率。

### 旋轉 = 變換編碼

TurboQuant 的隨機正交旋轉在 DSP 中並非新概念，它就是**變換編碼（transform coding）**：

| 系統 | 變換 | 之後 |
|---|---|---|
| JPEG | DCT | 對每個係數獨立量化 |
| 音訊編解碼器（MP3/AAC） | MDCT | 對每個頻率 bin 獨立量化 |
| TurboQuant | 隨機正交矩陣 | 對每個座標獨立量化 |

目的完全相同：將相關輸入轉換為去相關/獨立的表示，使獨立的純量量化趨近最佳。JPEG 使用 DCT 是因為自然影像的統計特性是已知的。TurboQuant 使用隨機旋轉是因為它需要**資料無關性（data-oblivious）**——旋轉保證無論輸入向量為何，都能獲得 Gaussian 分布。

### QJL = 減法式抖動（以及為何社群發現它有害）

TurboQuant 的兩階段架構——MSE 量化器加上對殘差的 1-bit QJL 校正——在 DAC 設計中有直接的類比：**減法式抖動（subtractive dithering）**。

經典抖動：量化、計算殘差、將殘差的表示發送給接收端以消除量化偏差。代價是噪聲底板升高（變異數增加）。

QJL 正是如此。它將量化殘差投影至隨機矩陣並只儲存正負號（1 bit），產生無偏的內積估計器。代價是變異數增加。

llama.cpp 社群獨立發現，在相同總 bit 預算下，k-bit 純 MSE 優於 (k-1)-bit MSE + 1-bit QJL。在抖動理論中，這是已知的取捨：**抖動消除諧波失真（偏差）但升高噪聲底板（變異數）。當下游系統對噪聲底板比對諧波失真更敏感時，跳過抖動更好。**

Attention 的 softmax 正是這樣的系統——attention scores 中單一的高變異數離群值可能扭曲整個分布。純 MSE 量化的偏差小而系統性；QJL 的變異數是隨機且潛在毀滅性的。抖動框架預測了這個結果，實測數據也印證了它。

---

## 我的量測：三個前所未有的發現

我在 **Qwen2.5-1.5B-Instruct**（28 層、2 KV heads、12 Q heads、head_dim=128）上進行系統性量測，以 WikiText-2 和 Python 程式碼作為輸入。目標是用 DSP 指標刻畫量化噪聲結構。完整 notebooks 收錄於 repo。

### 發現一：SQNR 基線——TurboQuant 驚人地均勻

在所有 28 層的 3-bit 量化下：

- **K 平均 SQNR：35.8 dB**
- **V 平均 SQNR：35.8 dB**
- 分布範圍：僅 34.7–36.6 dB（所有層不到 2 dB 的差距）
- K 與 V 幾乎相同——此模型無需不對稱的 K/V bit 配置

6 dB/bit 關係成立：3 bits × 6.02 ≈ 18 dB 量化 SQNR，加上 Lloyd-Max 增益與訊號功率貢獻，落在約 36 dB。Shannon gap ≤ 2.7 倍（4.3 dB）在 d=1536 的 10 萬 DBpedia 向量上得到確認。

**結論**：TurboQuant 已在理論最佳值附近運作，純量量化器層級沒有太多改善空間。

![SQNR vs bit-width](https://raw.githubusercontent.com/abc1199281/AI_Labs/main/experiments/turboquant-dsp/figures/sqnr-tooling/figure3_reproduction.png)
*D_prod 與 D_mse vs bit-width，復現論文 Figure 3。6 dB/bit 的斜率清晰可見。*

### 發現二：噪聲不是白噪聲——早期層的有色噪聲

這是最有趣的發現。我定義了一個**等向性比（isotropy ratio）**，衡量量化誤差是否均勻分布於各方向，或偏向 query 方向：

```
isotropy_ratio = ||P_q(e)||² / (||e||² / d)

= 1.0 → 誤差為等向性（均勻分布於所有方向）
> 1.0 → 誤差偏向 Q 方向（對 attention 有害）
< 1.0 → 誤差自然迴避 Q 方向（有益）
```

量測結果：

| 層群 | 等向性比 | 解讀 |
|---|---|---|
| Layer 0 | ~1.96 | 誤差在 Q 方向集中約 2 倍 |
| Layers 1–9 | ~1.4 | 顯著的 Q 方向偏差 |
| Layers 10–27 | ~0.97–1.17 | 接近等向性 |

![每層等向性比](https://raw.githubusercontent.com/abc1199281/AI_Labs/main/experiments/turboquant-dsp/figures/isotropy/isotropy_ratio.png)
*28 層的等向性比（mean ± std）。Layer 0 是明顯的離群值。*

**以 DSP 術語表達：量化噪聲是有色的，不是白噪聲。噪聲功率譜密度恰好在「訊號頻帶」達到峰值——也就是 attention 最在意的 query 方向。**

這是最糟糕的情況，類比於噪聲指數在訊號頻率最高的 ADC。

**為何發生？** K 與 Q 都是相同隱藏狀態的線性投影：K = W_k·h，Q = W_q·h。當 h 具有強烈的非等向性（在早期層常見，因為嵌入還很新鮮，layer norm 尚未完全正規化），K 與 Q 的主要方向是相關的——兩者都受 h 的主成分驅動。

TurboQuant 的旋轉應該消除這種非等向性，而且確實如此——漸近地。但在 d=128 時，收斂是不完整的。這在 Notebook 01 中得到驗證：在 d=1536 時，旋轉後的 Beta 分布與 Gaussian 幾乎完全吻合，但在 d=128 時有明顯落差。輸入向量的殘餘非等向性「洩漏」過有限維度的旋轉，又因為 K 與 Q 共享它們的非等向性（透過 h），量化誤差便與 Q 相關。

**含意**：任何假設等向性誤差的 KV cache 量化方法（包括 TurboQuant 的理論分析）都會**低估**其對早期層 attention 精度的影響，最多達 2 倍。

### 發現三：視頻風格的時序預測是死路

作為訊號處理背景的人，我的第一直覺是：相鄰 token 的 K 向量應該很相似（時序冗餘），所以像視頻壓縮（I-frame 和 P-frame）一樣儲存差值。我定義了**差值變異比（Delta Variance Ratio，DVR）**：

```
DVR = Var(K[t] - K[t-1]) / Var(K[t])

< 0.5 → delta coding 至少節省 3 dB（~0.5 bits）
< 0.25 → 節省 6 dB（~1 bit）
≈ 1.0 → 無時序冗餘可利用
> 1.0 → delta 比原始更差（不值得嘗試）
```

量測結果：

| 輸入 | 平均 DVR | DVR < 0.5 的層數 |
|---|---|---|
| WikiText-2 | 1.16 | 0 / 28 |
| Python code | 1.25 | 0 / 28 |
| **整體** | **1.21** | **0 / 28** |

![每層 DVR](https://raw.githubusercontent.com/abc1199281/AI_Labs/main/experiments/turboquant-dsp/figures/kvcache-stats/delta_var_ratio.png)
*28 層的 Delta Variance Ratio。0.5 門檻（虛線）從未被突破。*

**DVR > 1 意味著相鄰 token 之間的差值比 token 本身更具變異性。** 視頻風格的 DPCM（差分脈衝碼調製）不只無用——它比直接儲存原始值更糟。

但等等——相鄰 K 向量的 cosine similarity 約為 0.8。相似度高，差值變異數怎麼也可能高？

**答案是維度詛咒。** 在 d=128 維空間中，cosine similarity 為 0.8 的兩個向量夾角約為 37°。這聽起來很小，但在高維空間中，差值錐的「體積」是巨大的。在 d=128 時，20% 的方向差異換算成差值，其量級與向量本身相當。

這是一個具體、實驗驗證的例子，說明低維訊號處理的直覺（音訊：1D，視頻：2D 空間）為何在 transformer 隱藏狀態的高維設定中會徹底失效。

**對實務工程師的結論**：不要嘗試 token 間的 KV cache 壓縮。token 序列中存在的時序冗餘，在 d=128 的向量層面並不成立。每篇現有的 KV cache 量化論文（TurboQuant、KIVI、KVQuant）都正確地將每個 token 視為獨立處理。

### 補充：噪聲整形上限

借用 sigma-delta ADC 理論，d=128 下噪聲整形的理論上限為：

```
理論上限：10·log₁₀(128) ≈ 21.07 dB
考慮等向性比後的實際上限：20.02 dB
差距：1.06 dB
```

20 dB 的上限意味著，假設性的完美噪聲整形可讓 3-bit 量化在 attention 精度上表現得如同 ~6.5-bit。但實現這個上限存在根本性障礙——attention 不是理想低通濾波器（它往往是尖峰型的），而且啟用噪聲整形的回饋機制引入了與 GPU 平行化衝突的序列依賴。

---

## 實務結論

**對推論系統工程師：**

3-bit 的 TurboQuant 已距 Shannon 極限 4.3 dB 以內。純量量化器不是瓶頸。如果需要更好的品質，提升至 4-bit（增益 ~6 dB）比任何 3-bit 噪聲整形方案更實際。

**對 KV cache 量化研究者：**

等向性比是一個有用的診斷工具。如果你在開發新的量化方法，請確認量化誤差是等向性的，還是偏向 query 方向。早期層值得特別關注——它們始終表現出最差的噪聲特性。

DVR 量測應作為嘗試任何時序/差值壓縮方案前的標準把關。在 d=128 時，答案一致是「別費力了」。

**對設計 LLM 推論 NPU/GPU 的硬體架構師：**

有色噪聲的發現暗示，未來的量化方案可能受益於每層或每 head 的 bit 配置。支援混合精度 KV cache 的硬體設計（例如早期層 4-bit、深層 2-bit）可捕捉顯著的效率增益。SQNR 框架直接映射到硬體設計參數——記憶體頻寬需求隨 bit 寬度縮放，SQNR 精確告訴你在交換什麼品質。

---

## 開放問題與未來方向

**能否不經校準修正有色噪聲？** 早期層的等向性比 > 1 是有限維度旋轉造成的系統性偏差。一個方向是解碼時校正：一個逐座標的 Wiener 濾波器，利用解析計算的 bin-variance 統計量對高噪聲座標降權，同時保持資料無關性。我的估計：早期層增益 1–3 dB，是否足夠需要進一步評估。

**子頻帶 bit 配置是否值得校準成本？** query 向量的 PCA 顯示早中層有效秩 r/d = 0.31–0.36，暗示 query-aware bit 配置有 4–5 dB 的潛在增益。但這需要校準資料，犧牲了 TurboQuant 的核心賣點——資料無關性。這個取捨是否值得取決於部署情境。

**attention-aware 噪聲整形能否突破上限？** 20 dB 噪聲整形上限存在，但受到根本性障礙保護：attention 不是低通濾波器。尖峰型 attention 模式（attention sinks、needle-in-haystack 檢索）在不衰減的情況下傳遞高頻量化噪聲。解決這個問題需要重新思考 sigma-delta 類比——或許是區塊式噪聲整形，或是對 attention 模式本身作出響應的自適應方法。

**為何 d=128 影響如此大？** 我發現的許多限制都源於同一個根本原因：d=128 的維度不夠大，漸近保證無法完全發揮。隨著模型演進至更大的 head 維度（某些架構已使用 d=256），等向性比應會改善，噪聲也會趨近白噪聲。測量這些發現如何隨 d 變化是重要的下一步。

---

## 關於作者

我是 Arm 英國的 Staff Model Engineer，專責建構影像與顯示處理流程的週期精確效能模型。我的背景涵蓋半導體驗證（SystemC/SystemVerilog）、電腦視覺（博士）及電機工程（交通大學，台灣——含 PCB 設計與 ADC 實驗室經驗）。我對橋接半導體效能建模方法論與 AI 推論優化感興趣。

如果您覺得這個分析有用，非常歡迎與我聯繫——特別是如果您研究推論系統、LLM 硬體或量化。請在 [LinkedIn/Twitter/GitHub] 找到我。

---

## 參考文獻

1. Zandieh, A., Daliri, M., Hadian, M., & Mirrokni, V. (2025). TurboQuant: Online Vector Quantization with Near-optimal Distortion Rate. ICLR 2026. [arXiv:2504.19874](https://arxiv.org/abs/2504.19874)
2. Zandieh, A., Han, I., Daliri, M., & Karbasi, A. (2024). QJL: 1-Bit Quantized JL Transform for KV Cache Quantization with Zero Overhead. AAAI 2025. [arXiv:2406.03482](https://arxiv.org/abs/2406.03482)
3. Zandieh, A., Hadian, M., & Mirrokni, V. (2025). PolarQuant: Quantizing KV Caches with Polar Transformation. AISTATS 2026. [arXiv:2502.02617](https://arxiv.org/abs/2502.02617)
4. ggml-org/llama.cpp Discussion #20969: TurboQuant — Extreme KV Cache Quantization. [Link](https://github.com/ggml-org/llama.cpp/discussions/20969)
5. scos-lab/turboquant: Reference implementation with K/V ratio engineering insights. [Link](https://github.com/scos-lab/turboquant)

---

*本分析為業餘專案。所有量測在 Google Colab（免費版，CPU）上使用 Qwen2.5-1.5B-Instruct 執行。三個 Jupyter notebooks 與所有圖片均收錄於[配套 repository](https://github.com/abc1199281/AI_Labs)。*
