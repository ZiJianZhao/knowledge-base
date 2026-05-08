---
title: "Rethinking On-Policy Distillation of Large Language Models: Phenomenology, Mechanism, and Recipe"
method: OPD-Rethinking
authors: [Yaxuan Li, Yuxin Zuo, Bingxiang He, Jinqian Zhang, Chaojun Xiao, Cheng Qian, Tianyu Yu, Huan-ang Gao, Wenkai Yang, Zhiyuan Liu, Ning Ding]
published: 2026-04
venue: arXiv
tags: [knowledge-distillation, on-policy-distillation, LLM, reasoning, GRPO, token-level-dynamics]
created: 2026-05-08
---

## 核心贡献

本文系统研究了 On-Policy Distillation（OPD）在 LLM 推理场景中的成功与失败条件，形成「现象学—机制—方法」三段论框架：

1. **现象学（Phenomenology）**：识别出 OPD 成功的两个必要条件——思维模式一致性（Thinking-Pattern Consistency）和真正的新知识（New Knowledge）。
2. **机制（Mechanism）**：通过 token 级动态指标揭示成功 OPD 的内在签名：高概率 token 重叠率持续上升（72% → 91%），共享 token 集中了 97–99% 的概率质量。
3. **方法（Recipe）**：提出两种实用修复策略——Off-Policy Cold Start 和 Teacher-Aligned Prompts——在失败配置下恢复 OPD 效果。

与之前工作相比，本文首次从 token 级动态视角解释了"为何更强的 teacher 有时反而效果更差"，并给出了可操作的诊断指标和修复路径。

---

## 方法

### 整体框架

OPD 的标准目标是在 student 自采样的轨迹上最小化 student 与 teacher 分布的 KL 散度：

$$
\mathcal{L}_{\mathrm{OPD}}(\theta) = \mathbb{E}_{x \sim \mathcal{D}_x} \left[ D_{\mathrm{KL}}\left(\pi_\theta(\cdot|x) \| \pi_T(\cdot|x)\right) \right]
$$

token 级展开（student 自采样轨迹 $\hat{y}$）：

$$
\mathcal{L}_{\mathrm{OPD}}(\theta) = \mathbb{E}_{x \sim \mathcal{D}_x, \hat{y} \sim \pi_\theta(\cdot|x)} \left[ \sum_{t=1}^{T} D_{\mathrm{KL}}(p_t \| q_t) \right]
$$

其中 $p_t$ 为 student 在第 $t$ 步的分布，$q_t$ 为 teacher 对应分布。

论文对比了三种 OPD 变体：

- **Sampled-Token OPD**（最轻量）：只在 student 采样的单个 token 上计算损失
- **Full-Vocabulary OPD**：对全词表计算 KL
- **Top-k OPD**：在 student/teacher 各自 top-k token 的并集 $S_t$ 上计算截断 KL

$$
\mathcal{L}_{\mathrm{OPD}}^{\mathrm{top\text{-}k}}(\theta) = \mathbb{E}\left[\sum_{t=1}^{T} D_{\mathrm{KL}}\left(\bar{p}_t^{(S_t)} \| \bar{q}_t^{(S_t)}\right)\right]
$$

Sampled-Token OPD 简化为：

$$
\mathcal{L}_{\mathrm{OPD}}^{\mathrm{sample}}(\theta) = \mathbb{E}\left[\sum_{t=1}^{T} \left(\log p_t(\hat{y}_t) - \log q_t(\hat{y}_t)\right)\right]
$$

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x1.png" width="600">
  <br>
  <em>Figure 1：OPD 整体概览，展示 JustRL-1.5B 和 Skywork-OR1-Math-7B 作为 teacher 的蒸馏路径。</em>
</p>

---

### 关键组件

#### 1. 思维模式一致性（Thinking-Pattern Consistency）

**是什么**：student 和 teacher 是否使用相似的推理链格式（如 `<think>...</think>` 结构），以初始 token 重叠率衡量。

**为什么重要**：如果 teacher 是 Non-thinking 模式而 student 是 thinking 模式（或反之），两者在高概率 token 上的分布天然错位，OPD 从一开始就在优化错误的方向。

**诊断方式**：训练开始时的 overlap ratio（$\mathcal{M}_{\text{overlap}}$）是预测 OPD 成功与否的早期指标。

$$
\mathcal{M}_{\text{overlap}} \triangleq \mathbb{E}_t\left[\frac{|S_t^{(p)} \cap S_t^{(q)}|}{k}\right]
$$

其中 $S_t^{(p)}$ 和 $S_t^{(q)}$ 分别为 student 和 teacher 在第 $t$ 步的 top-k token 集合。

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x2.png" width="600">
  <br>
  <em>Figure 2：GRPO teacher（thinking 模式）vs Non-thinking teacher 的 OPD 效果对比，GRPO teacher 具有更高的初始重叠率和更强的蒸馏效果。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x3.png" width="600">
  <br>
  <em>Figure 3：两种 teacher（Qwen3-4B Non-thinking vs Qwen3-4B-Base-GRPO）在 AIME 2024、AIME 2025、AMC 2023 上的验证性能。</em>
</p>

#### 2. 新知识条件（New Knowledge Beyond Same Pipeline）

**是什么**：teacher 必须具备 student 在训练过程中未曾接触过的能力。如果 student 和 teacher 使用相同数据和训练流程（只是规模不同），两者收敛到相近分布，可迁移信号极少。

**关键实验**：Reverse Distillation——用 student 的前 RL 版本作为 teacher，结果表明蒸馏效果退化至 baseline，证明"更大的模型"不等于"有新知识"。

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x4.png" width="600">
  <br>
  <em>Figure 4：有/无 RL post-training 的 teacher 在 DeepSeek 和 Qwen 两个家族上的 OPD 性能及 gap recovery rate 对比。</em>
</p>

**Gap Recovery Rate 定义**：

$$
\text{Gap Recovery Rate} = \frac{\text{Acc}_{\text{after OPD}} - \text{Acc}_{\text{before OPD}}}{\text{Acc}_{\text{teacher}} - \text{Acc}_{\text{before OPD}}}
$$

经过 RL post-training 的 teacher 显著提升 gap recovery rate。

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x5.png" width="600">
  <br>
  <em>Figure 5：Reverse Distillation 实验——以 JustRL-1.5B 为 student，同家族 teacher 均导致性能退化。</em>
</p>

#### 3. Token 级动态机制

**成功 OPD 的签名**：

- Overlap ratio 从训练初期 ~72% 持续上升至 ~91%
- 共享 token 集中了 97–99% 的总概率质量
- 仅优化重叠 token 即可恢复完整性能；非重叠 token 贡献极小

**失败配置的签名**：overlap ratio 从训练初期就停滞，entropy gap 持续存在。

熵差指标：

$$
\Delta H_t = |H(q_t) - H(p_t)|
$$

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x6.png" width="600">
  <br>
  <em>Figure 6：成功 vs 失败 OPD 的 overlap ratio、advantage、entropy gap 训练动态对比。</em>
</p>

**消融实验**：将 OPD 损失拆分为"重叠 token"和"非重叠 token"两部分，分别训练：

- Overlap Top-k：性能与完整 Top-k OPD 持平
- Non-Overlap Top-k：性能显著下降

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x7.png" width="600">
  <br>
  <em>Figure 7：重叠 vs 非重叠 token 消融实验，证明重叠 token 是 OPD 效果的核心来源。</em>
</p>

#### 4. 修复策略 A：Off-Policy Cold Start

在 OPD 之前，先用 teacher 生成的轨迹做 SFT（监督微调），缩小初始思维模式差距，再开始 OPD。

效果：SFT 初始化的 student 在整个 OPD 训练过程中均优于 base 初始化。

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x8.png" width="600">
  <br>
  <em>Figure 8：Off-Policy Cold Start 效果——SFT 初始化 student 优于 base 初始化。</em>
</p>

#### 5. 修复策略 B：Teacher-Aligned Prompts

使用来自 teacher post-training 分布的 prompt，而非通用数学题库，可以锐化高概率 token 的对齐，但需注意：

- Prompt 格式对齐（template alignment）提升 accuracy 和 overlap 增长速度
- Prompt 内容对齐（content alignment）进一步提升性能，但过度对齐会导致 entropy collapse
- 建议混合使用 aligned 和 out-of-distribution prompts

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x9.png" width="600">
  <br>
  <em>Figure 9：Teacher-aligned template vs 通用 template 的 accuracy 和 overlap 增长对比。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x10.png" width="600">
  <br>
  <em>Figure 10：Teacher-aligned prompts vs 通用 prompts 的性能和概率质量集中度对比。</em>
</p>

---

### 核心公式汇总

| 公式名称 | LaTeX | 含义 |
|----------|-------|------|
| OPD 序列级目标 | $\mathcal{L}_{\mathrm{OPD}}(\theta) = \mathbb{E}_{x}[D_{\mathrm{KL}}(\pi_\theta \| \pi_T)]$ | 最小化 student 与 teacher 分布的 reverse KL |
| OPD token 级展开 | $\mathbb{E}[\sum_t D_{\mathrm{KL}}(p_t \| q_t)]$ | 按 student 采样轨迹分解到每个时间步 |
| Sampled-Token OPD | $\mathbb{E}[\sum_t (\log p_t(\hat{y}_t) - \log q_t(\hat{y}_t))]$ | 最轻量变体，只在采样 token 上计算 |
| Top-k OPD | $\mathbb{E}[\sum_t D_{\mathrm{KL}}(\bar{p}_t^{(S_t)} \| \bar{q}_t^{(S_t)})]$ | 在 top-k 并集上计算截断 KL |
| Overlap Ratio | $\mathcal{M}_{\text{overlap}} = \mathbb{E}_t[|S_t^{(p)} \cap S_t^{(q)}| / k]$ | 衡量 student/teacher top-k token 集的重叠程度 |
| Entropy Gap | $\Delta H_t = \|H(q_t) - H(p_t)\|$ | 衡量 student/teacher 在第 t 步的熵差 |
| Gap Recovery Rate | $(Acc_{after} - Acc_{before}) / (Acc_{teacher} - Acc_{before})$ | 衡量 OPD 弥合 student-teacher 性能差距的比例 |

---

## 实验结果

### 实验配置

- **Student 模型**：Qwen3-1.7B-Base、DeepSeek-R1-Distill-1.5B
- **Teacher 模型**：Qwen3-4B（Non-thinking/GRPO）、DeepSeek-R1-Distill-7B、Skywork-OR1-Math-7B、JustRL-1.5B
- **训练数据**：DAPO-Math-17K
- **评测 Benchmark**：AIME 2024、AIME 2025、AMC 2023（avg@16）

### 主要发现

| 条件 | 结论 |
|------|------|
| Teacher 有 RL post-training | Gap Recovery Rate 显著更高 |
| Teacher 与 student 同家族无 RL | 效果接近 baseline，甚至退化 |
| Off-Policy Cold Start（SFT 预热） | 整个训练过程均优于 base 初始化 |
| Teacher-Aligned Prompts | 更高 accuracy + 更快 overlap 增长 |
| Top-k OPD（k=4,16,64） | 性能相当；Top-1（argmax）明显更差 |
| Sampled-Token OPD | 与 Top-k 性能相当，计算代价更低 |

### Response Length 分析

最优响应长度区间为 **3K–7K tokens**：

- 超过 10K–15K tokens：性能持平或下降
- 不稳定性起源于序列末尾（back-to-front instability）
- Teacher 的 continuation advantage 随 prefix depth 增加而减弱

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x11.png" width="600">
  <br>
  <em>Figure 11：不同 max response length 下的性能分析，3K-7K 为最优区间。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x12.png" width="600">
  <br>
  <em>Figure 12：不同 max response length 下的训练动态。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x13.png" width="600">
  <br>
  <em>Figure 13：15K max length 下各解码位置的 student entropy，展示 back-to-front 不稳定模式。</em>
</p>

### Reward 信号分析

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x14.png" width="600">
  <br>
  <em>Figure 14：正确 vs 错误 rollout 的序列平均 reward 分布，两者 AUROC 相当，说明序列级 reward 难以区分局部质量。</em>
</p>

### Top-k 支持集大小消融

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x15.png" width="600">
  <br>
  <em>Figure 15：k ∈ {1, 4, 16, 64} 的 Top-k OPD 性能对比，Top-1 明显最差，其余相当。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x16.png" width="600">
  <br>
  <em>Figure 16：不同 k 值下的训练动态。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2604.13016v2/x17.png" width="600">
  <br>
  <em>Figure 17：附录补充图，训练动态或额外消融（见原文附录）。</em>
</p>

### 局限性

**论文自认的局限**：
- 所有实验仅在数学推理任务上，其他领域适用性未验证
- Teacher-Aligned Prompts 存在 entropy collapse 风险，需要混合策略
- "Learnability Gap"：当推理复杂度超出 student 能力上限时，OPD 根本无法传递知识

**我认为的局限**：
- Gap Recovery Rate 的分母（teacher-student 初始差距）本身受 teacher 质量影响，指标设计可能掩盖绝对性能差异
- 实验规模偏小（1.5B/7B），在更大规模（70B+）下的结论是否成立未知
- Off-Policy Cold Start 的 SFT 数据来源于 teacher，引入了额外的 teacher 推理成本

---

## 与现有方案对比

| 方法 | 核心思路 | 与本文关系 |
|------|----------|------------|
| MiniLLM | 形式化 reverse KL OPD 目标 | 本文在其基础上分析成功/失败条件 |
| GKD (Generalized KD) | 统一 on-policy/off-policy 框架 | 本文聚焦 on-policy 场景的机制分析 |
| Hinton et al. (2015) | 知识蒸馏奠基工作，soft labels | 本文扩展到 LLM 推理的 token 级分析 |
| Kim & Rush (2016) | 序列级 KD | 本文进一步到 token 级动态 |
| Qwen3/MiMo/GLM-5 | 工业界 OPD 应用 | 本文提供理论解释和诊断工具 |
| Capacity Gap 研究 | 容量差距导致蒸馏退化 | 本文细化为"思维模式一致性"和"新知识"两个维度 |

**本文核心差异**：不只是说"OPD 有时失败"，而是给出了可量化的诊断指标（overlap ratio、entropy gap）和可操作的修复策略。

---

## 业务适用性

**适合的场景**：
- 有较强 teacher 模型（经过 RL post-training）可用，需要蒸馏到更小模型
- Student 和 teacher 在推理格式上兼容（同为 thinking 或同为 non-thinking）
- 数学推理、代码生成等有明确 reward 信号的任务
- 响应长度可控在 3K–7K tokens 的场景

**不适合的场景**：
- Teacher 只是参数量更大但训练流程相同（没有 RL post-training），蒸馏收益有限
- Student 和 teacher 思维模式差异大（如 thinking vs non-thinking），需要先做 Off-Policy Cold Start
- 需要超长推理链（>10K tokens）的任务，reward 信号退化问题未解决
- 非数学/推理类任务，论文结论的适用性未经验证

**落地难度**：中

原因：核心方法（Sampled-Token OPD + Off-Policy Cold Start）实现难度不高，但需要一个经过 RL post-training 的强 teacher，且 teacher 推理成本不可忽视。诊断指标（overlap ratio）需要在训练中实时监控，有一定工程成本。

---

## 模型评价

本文最有价值的贡献不是方法本身，而是**诊断工具**——overlap ratio 和 entropy gap 这两个指标让 OPD 的成败从黑盒变成了可观测的训练动态。这对实际工程落地很有价值：在训练早期就能判断当前配置是否有效，而不是等到训练结束才发现白跑了。

Off-Policy Cold Start 的逻辑也很直觉：OPD 失败的根本原因是 student 和 teacher「说不同的语言」，先 SFT 预热对齐语言格式，再做 OPD 传递知识，是合理的两阶段拆分。

局限在于实验范围偏窄（数学推理 + 小规模模型），且依赖一个经过 RL post-training 的强 teacher，在没有现成强 teacher 的场景下适用性有限。Gap Recovery Rate 这个指标的设计也值得商榷——分母是 teacher-student 初始差距，当 teacher 本身不强时，指标会虚高。

## 你的评价

>

---

## 参考

- arXiv: https://arxiv.org/abs/2604.13016
- 代码: https://github.com/thunlp/OPD
- 训练数据: DAPO-Math-17K
