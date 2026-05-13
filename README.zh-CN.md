<div align="right">
  <strong>选择语言：</strong>
  <br>
  <a href="README.md">🇺🇸 English</a> | <a href="README.zh-CN.md">🇨🇳 简体中文</a>
</div>

# 👻 GhostCoT: Zero-Shot Evasion for Fast-DetectGPT

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Status: Experimental](https://img.shields.io/badge/Status-Experimental-success.svg)]()

本开源项目提出了一种纯基于 Prompt 工程的零样本（Zero-Shot）文本重写架构。通过引入**隐式思维链（Implicit Chain-of-Thought, CoT）**与**强制变异指令**，GhostCoT 能够在保持文本逻辑连贯性的前提下，彻底破坏大语言模型（LLM）默认的均质概率分布，成功绕过以 [Fast-DetectGPT](https://github.com/baoguangsheng/fast-detect-gpt) 为代表的基于概率曲率（Probability Curvature）的 AI 内容检测引擎。

## 📊 Benchmark 测试数据

在没有任何外部 RAG 数据注入的情况下，单次 Prompt 调用的真实对抗测试结果如下：

| 指标 | 重写前 (Baseline AI) | 重写后 (GhostCoT) | 变化 |
| :--- | :--- | :--- | :--- |
| **平均 AI 概率 (AI Prob)** | 0.9546 | **0.4284** | **- 55.12% 📉** |
| **平均曲率 (Crit)** | 3.7239 | **1.1903** | **- 68.04% 📉** |
| **LCS 骨架相似度** | — | **0.5482** | *(完美重写甜点位)* |

> **💡 数据解读：** 
> 传统大模型生成的文本，其 Crit（曲率）通常大于 3.0，被检测器 100% 判定为 AI。本方法成功将 Crit 压低至 **1.19**（低于通用的人类判定阈值 1.5），同时 0.54 的 LCS 相似度证明了原文的核心逻辑与信息被完美保留，仅在“句法皮肉”上进行了重塑。

---

## ✂️ 前置准备：文本切片与滑动窗口 (Chunking Strategy)

**这是 GhostCoT 发挥极限威力的核心前提。**
大模型在处理超长文本生成时，会产生“指令遗忘”，导致防检测规则在文本中后段失效。因此，**切勿将整篇文章直接输入**。你必须在工程代码侧对长文本进行分块（Chunking）预处理：

1. **无重叠硬切分 (Hard Split without Overlap)**：按自然段切分，建议每个 Chunk 长度控制在 **500 字**左右。**绝对不要**保留传统 RAG 的文本重叠区（Overlap），否则会导致剧情复读。
2. **构建滑动窗口 (Sliding Window)**：在遍历改写每个 Chunk 时，动态提取上一个已改写 Chunk 的末尾 150 字作为 `previous_chunk_tail`，提取下一个待改写 Chunk 的开头 50 字作为 `next_chunk_head`。
3. **提取末句锚点 (End-Anchor)**：用代码提取当前待改写 Chunk 的最后一句话，作为大模型收尾的强制物理限制，彻底消除 Chunk 之间的逻辑断层与剧情偏移。

---

## 🧠 原理解析：为什么它能击穿 0 样本检测？

### 1. 传统检测器的软肋：概率曲率 (Log-Probability Curvature)
Fast-DetectGPT 等先进检测器的核心数学原理是：AI 在生成文本时，总是倾向于**贪心采样**（选择概率最高的下一个 Token）。因此，AI 生成的文本在统计学上表现为一条**极其平滑、持续处于高概率区间的曲线**。而人类写作充满跳跃性、反直觉词汇和参差的句式，表现为充满“统计学噪声”的锯齿状低概率曲线。

### 2. 拘束衣悖论 (The Straitjacket Paradox)
如果在普通的 Prompt 中要求 AI “多用短句、少用套话”，并同时施加严格的上下文逻辑约束时，大模型的算力会发生**过载**。为了保证逻辑不出错，模型会本能地退回到它最安全的底层语法区（即最高概率的均质词汇），导致“防检测指令”完全失效。

### 3. 本项目的破局点：隐式思维链 (Implicit CoT)
我们在输出最终文本前，强制模型生成一个 `<thought_process>` 标签。在这个思考域中，模型必须完成以下任务：
*   **自主定调**：判断语境氛围。
*   **黑名单清洗**：主动找出并列举原文本中的高频“AI 味”词汇。
*   **句法破坏策略**：明确规划在哪里截断句子、吃掉主语。

---

## ⚙️ Transformer 架构层面的降维打击

这套 Prompt 之所以有效，并非玄学，而是极其精妙地利用了 Transformer 底层架构的两个核心物理特性：

### 特性 A：前缀条件化统治 (Prefix Conditioning & KV Cache)
Transformer 模型本质上是一个“Next-Token Predictor”。
当 `<thought_process>`（包含了要抛弃的 AI 词、要替换的生僻词、要打碎的句法计划）在物理位置上被放置在正文输出的正上方时，这些内容被写入了模型最新的 **KV Cache（键值缓存）**。
在 Attention（注意力机制）中，距离最近的 Context 拥有绝对的统治级权重。这段包含了“低概率、反套路”策略的隐式前缀，强行污染了模型的采样轨迹，逼迫它偏离最高概率的预测路径。

### 特性 B：计算复杂度前置转移 (Compute Shifting)
如果让模型一边构思如何替换词汇，一边生成连贯正文，注意力头的算力分配会导致输出平庸化。
通过引入 `<thought_process>`，我们将所有复杂的“策略规划”和“特征提取”任务前置。当模型真正开始输出正文时，它的大脑已经被清空，算力被彻底解放，变成了一个只需严格执行刚刚写下的“图纸”的填字机器人。这种**“想写分离”**彻底打破了模型的平滑马尔可夫链，人为制造出了人类特有的结构突发性 (Burstiness) 和困惑度 (Perplexity)。

---

## 🚀 Core Prompt Template Example:

👉 **[Prompt Template](./ghostCot_template_cn.jinja)**

---

## 🚀 进阶：GhostCoT 两步思考版 (Two-Stage Reasoning)

除了针对文学创作优化的标准版，我们还提供了 **GhostCoT 两步思考增强版**。该版本引入了“元认知”机制，将思考过程解耦为**领域自适应分析**与**战术执行规划**，具备更强的泛化能力。

### 核心优势 (Key Advantages)

* **全体裁泛化 (Cross-Domain Adaptability)**：不再局限于小说场景。通过 Phase 1 的体裁识别，它能自动为技术博客、商业报告、学术摘要等不同领域匹配最佳的“反 AI 指纹”策略。
* **自适应扰动 (Adaptive Perturbation)**：模型会先自主标记当前体裁下的“高概率陈词滥调（AI Fingerprints）”，并在生成正文前主动压制这些 Token 的权重。
* **深度解耦 (Decoupled Logic)**：将“写什么”与“AI 习惯怎么写”彻底分开，强制模型在 `<thought_process>` 中完成从宏观定位到微观粉碎的完整推演。

### 性能表现 (Benchmark)

根据实测数据（含 17 个思考块的长文本任务），两步思考版表现出了更具统治力的抗检测性：

| 指标 (Metrics) | 重写前 (Original) | 重写后 (GhostCoT v2) | 变化 (Delta) |
| :--- | :--- | :--- | :--- |
| **平均 AI 概率 (Avg. Prob)** | 0.9730 | **0.4153** | **-57.31%** |
| **平均曲率 (Avg. Curvature)** | 3.9832 | **1.0477** | **-73.70%** |
| **LCS 相似度 (LCS Similarity)** | — | 0.4269 | — |

## 🚀 Template Example:

👉 **[Prompt Template](./ghostCot_two_steps_thinking_cn.jinja)**


---

