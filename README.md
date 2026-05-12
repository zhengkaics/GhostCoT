<div align="right">
  <strong>Read this in other languages:</strong>
  <br>
  <a href="README.md">🇺🇸 English</a> | <a href="README.zh-CN.md">🇨🇳 简体中文</a>
</div>

# 👻 GhostCoT: Zero-Shot Evasion for Fast-DetectGPT

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Status: Experimental](https://img.shields.io/badge/Status-Experimental-success.svg)]()

This open-source project introduces a Zero-Shot text rewriting architecture based purely on Prompt Engineering. By introducing an **Implicit Chain-of-Thought (CoT)** combined with **Forced Mutation Directives**, GhostCoT completely disrupts the homogeneous probability distribution typical of Large Language Models (LLMs). It successfully bypasses AI content detection engines like [Fast-DetectGPT](https://github.com/baoguangsheng/fast-detect-gpt) that rely on log-probability curvature—all while maintaining absolute logical coherence in the text.

## 📊 Benchmark Results

In real-world adversarial testing with zero external RAG data injection, a single API call yielded the following results:

| Metric | Before (Baseline AI) | After (GhostCoT) | Change |
| :--- | :--- | :--- | :--- |
| **Average AI Probability** | 0.9546 | **0.4284** | **- 55.12% 📉** |
| **Average Curvature (Crit)** | 3.7239 | **1.1903** | **- 68.04% 📉** |
| **LCS Skeleton Similarity** | — | **0.5482** | *(Perfect Sweet Spot)* |

> **💡 Data Insight:** 
> Standard LLM-generated text usually scores a Curvature (Crit) > 3.0, guaranteeing a 100% AI flag by detectors. GhostCoT crushes the Crit down to **1.19** (safely below the human threshold of 1.5). The 0.54 LCS (Longest Common Subsequence) score proves that the core logic and plot are perfectly preserved, with only the "syntactic flesh" being entirely reconstructed.

---

## ✂️ Prerequisite: Chunking Strategy & Sliding Window

**This is the core prerequisite for GhostCoT to reach its maximum potential.**
When LLMs process long-form generation, they suffer from "Attention Dilution" and prompt forgetting, causing evasion rules to fail midway. **Never input an entire article at once.** You must preprocess the text into chunks in your pipeline:

1. **Hard Split Without Overlap**: Split by paragraphs. Keep each chunk around **300-500 words**. **Absolutely DO NOT** use traditional RAG overlap, as it causes plot repetition.
2. **Sliding Window Context**: When processing a chunk, dynamically extract the last 150 words of the *previously rewritten chunk* as `previous_chunk_tail`, and the first 50 words of the *next chunk* as `next_chunk_head`.
3. **End-Anchor Extraction**: Programmatically extract the very last sentence of the current chunk draft. This acts as a forced physical limitation for the LLM to conclude its rewriting, eliminating logical gaps and plot drift between chunks.

---

## 🧠 The Mechanics: How it Breaks Zero-Shot Detection

### 1. The Achilles' Heel: Log-Probability Curvature
Advanced detectors like Fast-DetectGPT rely on mathematical curvature: LLMs tend to use **greedy sampling** (picking the highest probability next-token). Thus, AI text forms a **smooth, high-probability curve**. Human writing, full of leaps, counter-intuitive vocabulary, and uneven sentence structures, forms a jagged, low-probability curve full of "statistical noise."

### 2. The Straitjacket Paradox
If you simply prompt an LLM to "use short sentences and avoid cliches" while simultaneously imposing strict contextual logic constraints, the model experiences **compute overload**. To ensure logical accuracy, the model instinctively retreats to its safest, highest-probability neural pathways (default AI tone), rendering evasion prompts useless.

### 3. The Breakthrough: Implicit CoT
Before outputting the final text, GhostCoT forces the model to generate a `<thought_process>` block. In this hidden domain, the model must:
*   **Autonomously sniff the scene's tone.**
*   **Create a blacklist**: Actively identify and list high-frequency "AI-flavored" words in the draft.
*   **Plan syntactic destruction**: Explicitly plan where to break sentences and omit subjects.

---

## ⚙️ Transformer-Level Exploitation

GhostCoT isn't magic; it exploits two core physical traits of the Transformer architecture:

### Exploit A: Prefix Conditioning & KV Cache Dominance
When the `<thought_process>` (containing the rejected AI words, the rugged synonyms, and the syntactic destruction plan) is generated *physically right above* the final output, it is written into the model's latest **KV Cache**. In the Attention mechanism, this highly proximal, low-probability implicit prefix aggressively pollutes the model's sampling trajectory, forcing it off the greedy path.

### Exploit B: Compute Shifting
If an LLM has to simultaneously think about "how to replace words" and "how to write a coherent story," its attention heads dilute, resulting in mediocre output. By using `<thought_process>`, we shift all the complex "strategic planning" upstream. By the time the model starts generating the actual text, its "brain" is clear, and its compute is fully liberated to act as a strict executor of its own rugged blueprint. This completely shatters the smooth Markov chain.

---

## 🚀 Core Prompt Template Example

Inject your data dynamically using the **Chunking Strategy** mentioned above. 

*(Note: Translate this prompt to your target language if you are rewriting non-English text.)*

👉 **[English)](./ghostCot_template_en.jinja)**
👉 **[Chinese)](./ghostCot_template_cn.jinja)**