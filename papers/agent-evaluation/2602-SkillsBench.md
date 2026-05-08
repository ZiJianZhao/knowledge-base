---
title: "SkillsBench: Benchmarking How Well Agent Skills Work Across Diverse Tasks"
method: SkillsBench
authors: [Xiangyi Li, Yimin Liu, Wenbo Chen, Shenghan Zheng, Xiaokun Chen, Yifeng He, Yubo Li, Bingran You, Haotian Shen, Jiankai Sun, Shuyi Wang, Binxu Li, Qunhong Zeng, Di Wang, Xuandong Zhao, Yuanli Wang, Roey Ben Chaim, Zonglin Di, Yipeng Gao, Junwei He, Yizhuo He, Liqiang Jing, Luyang Kong, Xin Lan, Jiachen Li, Songlin Li, Yijiang Li, Yueqian Lin, Xinyi Liu, Xuanqing Liu, Haoran Lyu, Ze Ma, Runhui Wang, Tianyu Wang, Wengao Ye, Yue Zhang, Hanwen Xing, Yiqi Xue, Steven Dillmann, Han-chung Lee]
published: 2026-02
venue: arXiv
tags: [agent, benchmark, skills, llm-agent, evaluation, procedural-knowledge, claude-code, gemini-cli, codex]
created: 2026-05-08
---

## 核心贡献

SkillsBench 是第一个将 Agent Skills 作为一等评测对象的 benchmark，系统回答了「Skills 到底有没有用、什么情况下有用」这个问题。之前的 agent benchmark（SWE-bench、AgentBench 等）只测模型的原始能力，不测增强手段的效果。

核心贡献：

1. **首个 Skills 效果 benchmark**：84 个任务跨 11 个领域，每个任务在三种条件下评测（无 Skills / 精心编写的 Skills / 自生成 Skills），配备确定性验证器，消除 LLM-as-judge 的随机性。
2. **大规模实证**：7 种 agent-模型组合，7,308 条轨迹，是迄今最系统的 Skills 效果研究。
3. **反直觉发现**：精心编写的 Skills 平均提升 +16.2pp，但自生成 Skills 平均 -1.3pp——模型无法可靠地生成自己能有效利用的过程性知识。
4. **设计原则**：2-3 个模块的 Skills 优于大而全的文档；小模型 + Skills 可以超过大模型无 Skills 的性能。

## 方法

### 整体框架

SkillsBench 的评测流程分三个阶段：

**Phase 1：Benchmark 构建**
- 从三个来源聚合 Skills：开源 GitHub 仓库（12,847）、Claude Code 生态（28,412）、企业合作伙伴（5,891）
- 去重后保留 47,150 个唯一 Skills
- 105 名贡献者提交 322 个候选任务

**Phase 2：质量过滤**
- 自动化检查：结构验证、Oracle 100% 通过率、AI 生成检测（GPTZero）、泄漏审计
- 人工审核：数据有效性、任务真实性、Oracle 质量、Skill 质量、防作弊
- 最终录取 84 个任务（接受率 26.7%）

**Phase 3：评测**
- 三种条件：无 Skills、精心编写的 Skills、自生成 Skills
- 三种商业 agent harness：Claude Code、Gemini CLI、Codex CLI
- 确定性 pytest 验证器，每任务 5 次 trial，取平均

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x1.png" width="750">
  <br>
  <em>Figure 1：Agent 架构栈与 7 种 agent-模型配置在 84 个任务上的 resolution rate。精心编写的 Skills（米色）平均提升 +16.2pp；自生成 Skills（琥珀色）几乎没有收益甚至负收益。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x2.png" width="750">
  <br>
  <em>Figure 2：SkillsBench 完整流程图。Phase 1 聚合 47,150 个 Skills，Phase 2 质量过滤产出 84 个任务，Phase 3 跨 7 种配置执行 7,308 条轨迹。</em>
</p>

### 关键组件

#### Skill 的定义

一个合格的 Skill 必须满足四个标准：

- **过程性内容**：包含 how-to 指导（流程、SOP、领域惯例），而非事实检索
- **任务类适用性**：适用于一类问题，而非单个实例
- **结构化组件**：包含 SKILL.md 文件，可选资源（脚本、模板、示例）
- **可移植性**：完全基于文件系统，易于编辑、版本控制、跨 harness 使用

明确排除：system prompt（缺乏结构和资源）、few-shot 示例（声明性非过程性）、RAG 检索（事实性非过程性）、工具文档（描述能力而非过程）。

#### Task 规范

每个任务是一个自包含模块，包含四个组件：

```
tasks/<task-id>/
  instruction.md          # 人工编写的任务描述
  task.toml               # 元数据和资源配置
  environment/
    Dockerfile
    skills/               # 精心编写的 Skills（无 Skills 条件下缺失）
      <skill-name>/
        SKILL.md          # 必需
        scripts/          # 可选
        references/       # 可选
  solution/
    solve.sh              # Oracle 解法（必须 100% 通过）
  tests/
    test.sh               # 运行 pytest
    test_outputs.py       # 程序化断言
```

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x3.png" width="750">
  <br>
  <em>Figure 3：SkillsBench 的 11 个领域分布。Software Engineering（16 个）和 Office & White Collar（14 个）任务最多。</em>
</p>

#### 任务难度分层

| 难度 | 任务数 | 人工完成时间 |
|------|--------|-------------|
| Core | 17 (19.8%) | < 60 分钟 |
| Extended | 43 (50.0%) | 1-4 小时 |
| Extreme | 26 (30.2%) | > 4 小时 |

#### 三种评测条件

- **No Skills**：agent 只收到 instruction.md，环境中无 Skills
- **With Skills**：完整的 environment/skills/ 目录，包含所有示例、代码片段和资源
- **Self-Generated Skills**：无 Skills 提供，但 agent 被提示在解题前先生成相关过程性知识，然后用自己生成的 Skills 解题

自生成条件仅在 Claude Code（全部四个模型）和 Codex 上评测；Gemini CLI 因其显式 skill 激活接口不支持该条件。

#### Skill 注入机制

Skills 通过 Docker COPY 指令注入到各 agent 的特定路径：

```dockerfile
COPY skills /root/.claude/skills    # Claude Code
COPY skills /root/.codex/skills     # Codex CLI
COPY skills /root/.gemini/skills    # Gemini CLI
```

指令中从不提及使用哪个 Skill——agent 必须自主发现并应用。

### 核心公式

**Pass Rate**（主要指标）：对每个任务，在 5 次 trial 中平均二元奖励，再以固定分母 84 对所有任务求平均。

**Normalized Gain**（归一化增益）：

$$g = \frac{pass_{skill} - pass_{vanilla}}{1 - pass_{vanilla}}$$

- $pass_{skill}$：有 Skills 时的通过率
- $pass_{vanilla}$：无 Skills 时的基线通过率
- $g$：衡量向完美性能的比例提升

注意：高 $g$ + 低绝对提升可能是天花板效应；高 $g$ + 高绝对提升才表明真实的脚手架效果。

## 实验结果

### 主要结果（Table 3）

| Harness | 模型 | 无 Skills | 有 Skills | g (%) | 自生成 | g (%) |
|---------|------|-----------|-----------|-------|--------|-------|
| Gemini CLI | Gemini 3 Flash | 31.3% | 48.7% | 25.3 | - | - |
| Claude Code | Opus 4.5 | 22.0% | 45.3% | 29.9 | 21.6% | -0.5 |
| Codex | GPT-5.2 | 30.6% | 44.7% | 20.3 | 25.0% | -8.1 |
| Claude Code | Opus 4.6 | 30.6% | 44.5% | 20.0 | 32.0% | +2.0 |
| Gemini CLI | Gemini 3 Pro | 27.6% | 41.2% | 18.8 | - | - |
| Claude Code | Sonnet 4.5 | 17.3% | 31.8% | 17.5 | 15.2% | -2.5 |
| Claude Code | Haiku 4.5 | 11.0% | 27.7% | 18.8 | 11.0% | 0.0 |
| **平均** | | **24.3%** | **40.6%** | **21.5** | **21.0%** | **-1.8** |

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x4.png" width="750">
  <br>
  <em>Figure 4：通过率 vs. 成本的 Pareto 前沿。有 Skills 的配置（实心）整体上移。Gemini 3 Flash 比 Pro 便宜 44%（$0.55 vs $0.98/任务），因为其 4x 更低的 token 单价抵消了更高的 token 消耗。</em>
</p>

### 领域分析（Table 4）

| 领域 | 有 Skills | 无 Skills | 绝对提升 |
|------|-----------|-----------|----------|
| Healthcare | 86.1% | 34.2% | +51.9pp |
| Manufacturing | 42.9% | 1.0% | +41.9pp |
| Cybersecurity | 44.0% | 20.8% | +23.2pp |
| Natural Science | 44.9% | 23.1% | +21.9pp |
| Energy | 47.5% | 29.5% | +17.9pp |
| Office & White Collar | 42.5% | 24.7% | +17.8pp |
| Finance | 27.6% | 12.5% | +15.1pp |
| Media & Content Production | 37.6% | 23.8% | +13.9pp |
| Robotics | 27.0% | 20.0% | +7.0pp |
| Mathematics | 47.3% | 41.3% | +6.0pp |
| Software Engineering | 38.9% | 34.4% | +4.5pp |

Healthcare 和 Manufacturing 的巨大提升（+51.9pp、+41.9pp）与预训练覆盖不足高度相关；Math 和 Software Engineering 提升小，因为模型已有强先验。

### Skills 数量效应（Table 5）

| Skills 数量 | 有 Skills | 无 Skills | 绝对提升 |
|------------|-----------|-----------|----------|
| 1 个 | 42.2% | 24.4% | +17.8pp |
| 2-3 个 | 42.0% | 23.4% | +18.6pp |
| 4+ 个 | 32.7% | 26.9% | +5.9pp |

2-3 个 Skills 是最优点；4+ 个 Skills 因认知负担或冲突指导而效果下降。

### Skills 复杂度效应（Table 6）

| 复杂度 | 通过率 | 绝对提升 | N |
|--------|--------|----------|---|
| Detailed（详细） | 42.7% | +18.8pp | 1165 |
| Compact（精简） | 37.6% | +17.1pp | 845 |
| Standard（标准） | 37.1% | +10.1pp | 773 |
| Comprehensive（全面） | 39.9% | -2.9pp | 140 |

大而全的 Skills 实际上**有害**（-2.9pp）。Detailed 和 Compact 效果最好。

### 模型规模效应

Claude Haiku 4.5 + Skills（27.7%）> Claude Opus 4.5 无 Skills（22.0%）。Skills 可以部分替代模型容量，在过程性任务上让小模型超越大模型。

### 任务级极端案例

**Skills 帮助最大的任务**：
- `mario-coin-counting`：+85.7pp（2.9% → 88.6%）
- `sales-pivot-analysis`：+85.7pp
- `flood-risk-analysis`：+77.1pp
- `sec-financial-report`：+74.3pp

**Skills 有害的任务（16/84 任务出现负 delta）**：
- `taxonomy-tree-merge`：-39.3pp
- `energy-ac-optimal-power-flow`：-14.3pp
- `trend-anomaly-causal-inference`：-12.9pp
- `exoplanet-detection-period`：-11.4pp

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x11.png" width="750">
  <br>
  <em>Figure 11：有 Skills 时各模型的任务级通过率热力图。顶部（深蓝）为所有模型都能解的简单任务，底部（深红）为即使有 Skills 也无法解的极难任务。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x12.png" width="750">
  <br>
  <em>Figure 12：无 Skills 基线时各模型的任务级通过率热力图。与 Figure 11 相比，蓝色区域明显收缩，确认 Skills 的整体提升效果。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x13.png" width="750">
  <br>
  <em>Figure 13：Skills 提升量热力图（有 Skills - 无 Skills）。蓝色表示正提升，红色表示 Skills 有害。大多数任务呈正提升，但存在明显的负提升任务。</em>
</p>

### 消融实验关键发现

1. **自生成 Skills 失败的两种模式**：(1) 模型识别到需要领域知识但生成不精确或不完整的过程（如"用 pandas 处理数据"而不给具体 API 模式）；(2) 对于高领域知识任务（制造、金融），模型往往完全没意识到需要专门 Skills，直接用通用方法尝试。

2. **Harness 差异**：
   - Claude Code：Skills 利用率最高，所有 Claude 模型一致受益（+13.9pp 到 +23.3pp）
   - Gemini CLI：原始性能最高（48.7%）；提升范围 +13.6pp 到 +17.4pp
   - Codex CLI：竞争性原始性能（44.7%）；但经常忽略提供的 Skills，agent 承认 Skills 内容却独立实现解法

### 局限性

- **覆盖范围**：仅限终端容器化任务，结果可能不直接迁移到 GUI agent、多 agent 协调或超长任务
- **因果归因**：Skills 注入增加了上下文长度，部分收益可能来自"更多上下文"而非过程性结构——需要长度匹配基线（随机文本、仅检索文档）来隔离
- **生态系统代表性**：benchmark 使用顶部四分位 Skills（质量分 ≥ 9/12），而生态系统平均质量仅 6.2/12，真实部署效果会更差
- **确定性**：容器化提供状态隔离但不能完全消除非确定性和训练集泄漏

## Skills 生态系统分析

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x5.png" width="750">
  <br>
  <em>Figure 5：136 天内 Skill 创建的时间动态。2025 年底前每日新增平稳，2026 年 1 月激增至峰值 18,904，累计达 84,192 个 Skills。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x6.png" width="750">
  <br>
  <em>Figure 6：SKILL.md 的 token 分布（n=36,338）。大多数 Skills 轻量，中位数约 1,569 tokens（~1.5k tokens）。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x7.png" width="750">
  <br>
  <em>Figure 7：Skills 总大小分布（n=37,078）。中位数约 2,296 tokens（~2.5k tokens），分布高度偏向精简工件。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x8.png" width="750">
  <br>
  <em>Figure 8：Skill 类别分布。前 10 类占 79.6%，Documentation（11.9%）、Git/Version Control（11.8%）、Code Quality（9.0%）领先。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x9.png" width="750">
  <br>
  <em>Figure 9：每个 Skill 的文件数分布。大多数 Skills 包含 1-5 个文件，中位数为 1。</em>
</p>

<p align="center">
  <img src="https://arxiv.org/html/2602.12670v1/x10.png" width="750">
  <br>
  <em>Figure 10：文件扩展名分布。Markdown 文件占绝对主导，表明 Skills 优先自然语言指令而非可执行实现。</em>
</p>

生态系统质量评估：平均质量分 6.2/12（SD=2.8），说明实际部署中 Skills 质量参差不齐，benchmark 使用的高质量 Skills（均分 10.1/12）代表乐观场景。

## 与现有方案对比

| | SWE-bench | AgentBench | Terminal-Bench | **SkillsBench** |
|--|-----------|------------|----------------|-----------------|
| 评测目标 | 模型原始能力 | 模型原始能力 | 模型+harness能力 | **Skills 增强效果** |
| 评测方式 | 单条件 | 单条件 | 单条件 | **配对（有/无 Skills）** |
| 验证方式 | 程序化 | 混合 | 程序化 | **确定性 pytest** |
| 领域覆盖 | 软件工程 | 多领域 | 通用 | **11 个专业领域** |
| Skills 作为变量 | 否 | 否 | 否 | **是（核心设计）** |

SkillsBench 的核心差异：它不是在问"模型能做什么"，而是在问"Skills 让模型多做了什么"。这是一个增量效果 benchmark，而非能力 benchmark。

## 业务适用性

**适合的场景**：
- 需要评估 agent Skills / RAG 文档 / system prompt 等增强手段效果的团队
- 开发 agent harness 并想了解 Skills 集成优化方向的工程师
- 研究过程性知识对 LLM 的影响

**不适合的场景**：
- 评测模型原始能力（用 SWE-bench、Terminal-Bench 等）
- GUI 或多模态 agent 场景（benchmark 仅覆盖终端环境）
- 需要长上下文或多 agent 协调的任务

**落地难度**：中等。开源 benchmark 和评测框架（skillsbench.ai），但需要自行搭建 Docker 容器化环境和 Harbor 框架。对于想直接复用 benchmark 数据验证自己 Skills 效果的团队，成本较低；想贡献新任务的成本较高（接受率 26.7%）。

## 模型评价

这篇论文的价值主要在于**填补了一个显而易见但从未被系统解决的空白**：大家都在用 Skills，但没人知道它们到底有没有用、什么时候有用。从这个角度看，这是一篇扎实的实证工作。

**真实价值**：
- 自生成 Skills 无效这个发现很有价值，打破了"让模型自己生成知识再用"的直觉
- 2-3 个 Skills 最优、大而全 Skills 有害的发现对实践很有指导意义
- 小模型 + Skills ≥ 大模型无 Skills 的发现有商业价值（降本）

**潜在问题**：
- 评测的 Skills 质量远高于生态系统平均（10.1/12 vs 6.2/12），结论过于乐观
- 自生成条件的 prompt 设计可能不是最优的——更好的 prompt 工程是否能改变结论？
- Harness 本身的差异（Claude Code 对 Skills 有原生优化）使跨 harness 比较存在混淆变量
- 上下文长度的混淆问题作者自己也承认，但没有提供长度匹配基线

**与现有工作的关系**：
- 建立在 Terminal-Bench 框架上，任务格式高度相似，迁移成本低
- 与 Harbor 框架深度绑定，生态系统依赖较强

**值不值得复现**：对于做 agent 基础设施的团队，值得关注其 benchmark 数据集和评测方法；对于做 LLM 能力研究的团队，结论本身的参考价值高于复现价值。

## 你的评价

>

## 参考

- arXiv: https://arxiv.org/abs/2602.12670
- 项目主页: https://www.skillsbench.ai
- 代码/数据: https://github.com/laude-institute/harbor（Harbor 框架）
