# 🤖 AgentFlow-Reimplementation: 可訓練的流程中代理系統

## 🌟 專案簡介 (Project Overview)

本專案旨在重現和探索 [論文](https://arxiv.org/abs/2510.05592#) 中提出的 **AgentFlow** 架構與 **Flow-GRPO** 強化學習演算法。核心目標是構建一個可訓練的、模組化的 Agent 系統，以有效解決稀疏獎勵下的長時序規劃和工具使用問題。

**核心貢獻 (Core Contributions to Implement):**
1. 實作四模組 Agent 框架 (Planner, Executor, Verifier, Generator)。
2. 實現「演進記憶體 ($\mathbf{M}$)」來管理長時序上下文。
3. 實現 $\text{On-Policy}$ 的 $\text{Flow-GRPO}$ 強化學習迴路。

---

## 🛠️ 方法論與實作步驟 (Methodology & Implementation Steps)

我們將專案分解為四個迭代階段，確保每個核心組件都能獨立驗證，從而降低複雜性。

### 階段 I: 基礎框架與零樣本基線 (Baseline Setup)

**目標:** 建立最小可行系統 (MVS) 並驗證四模組結構的流程可行性。

| 步驟編號 | 說明 | 交付成果 (Deliverable) |
| :---: | :--- | :--- |
| **I-1** | **定義任務與工具集：** 選擇初始的 2-3 個基準任務 (e.g., HotpotQA, GameOf24)。確定並整合核心工具 (e.g., Google Search, Python Coder)。 | `config/tools.yaml`，基座 LLM 選擇 (e.g., Qwen-2.5-7B-Instruct)。 |
| **I-2** | **設計模組 Prompt：** 為 Planner, Executor, Verifier, Generator 撰寫結構化 Prompt，並確保輸出格式 (e.g., Verifier 必須輸出 'CONTINUE'/'STOP')。 | `prompts/` 資料夾，包含四個模組的 Prompt 模板。 |
| **I-3** | **建立凍結基線：** 實作完整的 AgentFlow 流程迴路 (但不進行訓練)。Planner 僅使用 Zero-shot Prompt 進行決策。 | `agentflow/pipeline.py`，能執行並產出完整軌跡 (w/o Flow-GRPO) 的 MVS。 |

### 階段 II: 記憶體與流程迴路 (Memory & Flow Loop)

**目標:** 實現 AgentFlow 的核心創新：狀態壓縮與流程管理。

| 步驟編號 | 說明 | 交付成果 (Deliverable) |
| :---: | :--- | :--- |
| **II-1** | **實現演進記憶體 ($\mathbf{M}$)：** 實作記憶體更新函數 $f_{mem}(\cdot)$。使用 Regex 或小型 LLM 進行結果解析、壓縮，將 $\text{Action}$ ($\mathbf{a}^t$)、$\text{Result}$ ($\mathbf{e}^t$) 和 $\text{Status}$ ($\mathbf{v}^t$) 結構化存入 $\mathbf{M}^{t+1}$。 | `agentflow/memory.py`，具有**確定性**的記憶體讀寫和更新邏輯。 |
| **II-2** | **實現終止機制：** 根據 Verifier 的 $\text{STOP}$ 信號或達到 $T_{\text{max}}$ (訓練 $T_{\text{max}}=3$) 來可靠地終止 $\text{Rollout}$ 迴圈。 | 驗證：執行 5 步任務，確認 $\mathbf{M}$ 增長速度可控，且迴圈正常結束。 |
| **II-3** | **實現獎勵函數：** 引入 $\text{LLM-as-a-Judge}$ (e.g., GPT-4o API) 判斷最終答案 $\mathbf{o}$ 的正確性，返回 $\text{Trajectory-level}$ 的二元獎勵 $\overline{R}(\tau) \in \{0, 1\}$。 | `reward/judge.py`。 |

### 階段 III: Flow-GRPO 強化學習 (RL Optimization)

**目標:** 將 $\text{Planner}$ $\pi_{\theta}$ 納入 $\text{On-Policy}$ 訓練。

| 步驟編號 | 說明 | 交付成果 (Deliverable) |
| :---: | :--- | :--- |
| **III-1** | **策略 $\text{Rollout}$：** 實作 $\text{Rollout}$ 採集器。使用 $\pi_{\theta_{old}}$ 策略生成 $G$ 條完整軌跡 $\{\tau_i\}_{i=1}^G$，並記錄每一步的 $\text{Token}$ 級別機率。 | `rl/rollout_collector.py`，輸出包含 $\mathbf{s}^t, \mathbf{a}^t, \log \pi_{\theta_{old}}(\mathbf{a}^t|\mathbf{s}^t)$ 的 $\text{Batch}$ 數據。 |
| **III-2** | **獎勵廣播與優勢計算：** 實作 $\text{Flow-GRPO}$ 的信用分配機制：將 $\overline{R}(\tau)$ 廣播給軌跡中所有 $t$ 步驟。計算**組歸一化優勢** $A_i^t$ (Eq. 7)。 | `rl/advantage_calculator.py`。 |
| **III-3** | **實現 $\text{PPO/GRPO}$ 目標：** 實作 $\text{Token-level}$ 的 $\text{Flow-GRPO}$ 損失函數 (Eq. 5)，包括 $\text{Importance Ratio}$、$\text{Clipping}$ 和 $\text{KL}$ 懲罰項。 | `rl/ppo_loss.py`。 |
| **III-4** | **優化器與訓練迴路：** 使用 $\text{AdamW}$ 或類似優化器，以論文設定的超參數 ($\text{LR}=10^{-6}, \beta=0.001$) 訓練 $\text{Planner}$ 模組。 | `rl/trainer.py`，能穩定運行 $32$ $\text{Rollout}$ $\text{Batch}$ 的訓練迴圈。 |

### 階段 IV: 驗證與分析 (Verification & Analysis)

**目標:** 驗證 $\text{Flow-GRPO}$ 帶來的性能提升和 $\text{Agent}$ 行為變化。

| 步驟編號 | 說明 | 關鍵方法論 |
| :---: | :--- | :--- |
| **IV-1** | **性能比較：** 評估 $\text{AgentFlow}$ (w/o Flow-GRPO) 基線和 (w/ Flow-GRPO) 的最終性能差異。 | 報告：至少在 **4 個**主要基準任務上的 $\text{Accuracy}$ 提升百分比。 |
| **IV-2** | **工具使用分析：** 繪製類似 $\text{Figure 5}$ 的圖表，分析訓練前後 $\text{Planner}$ 對不同工具 (e.g., $\text{Google Search}$ vs. $\text{Wikipedia Search}$) 的使用頻率變化。 | 報告：工具呼叫比率 (Tool Call Ratio) 和呼叫錯誤率 (Calling Error Rate) 的趨勢分析。 |
| **IV-3** | **定性案例研究：** 仔細分析訓練前後**完整軌跡 $\tau$** 的對比。觀察 $\text{Planner}$ 是否學會了從失敗中自我修復、更高效的規劃路徑 (e.g., 減少步驟長度)。 | 報告：提供至少兩個**成功案例對比**，證明 $\text{Flow-GRPO}$ 增強了**規劃和工具協調能力**。 |

---

## 🔗 參考資料 (References)

- **AgentFlow Paper**: [https://arxiv.org/abs/2510.05592#]
- **Qwen2.5 Model**: 選擇的基座 $\text{LLM}$。