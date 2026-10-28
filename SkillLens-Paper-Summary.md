# SkillLens 论文总结

> **From Raw Experience to Skill Consumption: A Systematic Study of Model-Generated Agent Skills**
> 复旦大学 + Microsoft Research + 上海交大,16 位作者(一作 Zisu Huang)· 2026-05-22
> [arXiv:2605.23899](https://arxiv.org/abs/2605.23899) · 代码:[aka.ms/SkillLens](https://aka.ms/SkillLens) · License: CC BY 4.0

---

## 研究问题

Agent 领域流行让模型从自己的执行轨迹中自动提取 skill(而不是人工手写),提取方法层出不穷——但**没人系统回答过:这样生成的 skill 到底有没有用?什么时候有用?成败由什么决定?**

现有基准(SkillsBench、SWE-Skills-Bench、Skills-in-the-Wild)都只测「装上 skill 后涨不涨分」,把提取环节排除在外;SkillCraft 虽涉及提取,但把 skill 限制为可执行函数组合,泛化性存疑。

## 评估框架:覆盖 skill 全生命周期

论文提出首个贯穿三个阶段的 utility-grounded 评估框架:

```
Stage 1 经验生成        Stage 2 skill 提取           Stage 3 skill 消费
目标模型跑训练任务  →  提取器从轨迹池蒸馏 skill  →  装回同一模型,在 held-out
   产生经验池          (固定 schema + 预算)         测试集测 Δ = 有skill − 无skill
```

- **5 个域**:具身规划(ALFWorld)、办公软件(SpreadsheetBench)、软件工程(SWE-bench-Verified)、网络搜索(SEAL-0)、工具调用(BFCL-v4);**6 个目标模型 × 5 个提取器**全交叉
- **刻意极简的提取框架**:逐轨迹提取成功/失败 pattern → 树状分组归并(借鉴 Trace2Skill 但剥掉子代理编队、冲突消解、skill 深化等所有工程技巧),保证差异可归因于提取器本身能力
- **skill 表示**:遵循 Agent Skills 开放标准(name / description / body / references / scripts)
- **两个解耦指标**:
  - **EE**(Extraction Efficacy):固定提取器,跨不同目标产出有用 skill 的可靠度
  - **TE**(Target Evolvability):固定目标,从自己的经验中被不同提取器提炼后的受益程度

## 核心结论

### RQ1:skill 平均有用,但「负迁移」不可忽视

- 约 **75% 的组合里 skill 带来正收益,25% 出现负迁移**——加了 skill 比不加还差
- 提取器和目标模型的表现都不均匀:**同一个模型可以是强提取器却同时是弱消费者**,反之亦然(如 Gemini-3.1-FL 在提取效率上反超 GPT-5.4)
- skill 的效用与模型规模、基线任务能力**均不相关**——不能靠「换更强的模型」来解决

### RQ2:什么决定 skill 的效用?

逐阶段解剖发现:

- **经验构成塑造 skill 质量**——经验池的多样性/失败轨迹占比直接影响提取结果
- **有用的 skill 有共同特征**:显式编码失败模式、指令具体可执行、包含风险动作黑名单;反之,泛泛的成功经验总结价值很低
- **同一 skill 跨消费者迁移时表现分化**——skill 不是普适的,消费端特性决定吸收效果

### RQ3:发现可以变成药方——meta-skill

把 RQ2 的发现固化为一个 **meta-skill**(指导「如何写 skill」的 skill),要求提取出的 skill 必须满足三个维度:

| 维度 | 要求 |
|---|---|
| **Failure-mechanism encoding** | 显式写出「如果 X 失败 → Y」的分支 |
| **Actionable specificity** | 禁止"考虑/可能/视情况"等软化措辞 |
| **Risk-action blacklist** | 有明确的"不要做什么"反例清单 |

效果:

- 用 meta-skill 约束提取后,**所有 domain 一致提升约 +1.55pp**——不是某个领域的特例
- **大幅减少负迁移**
- 附带发现:用 LLM 直接评判两份 skill 哪份更好,准确率仅 **46.4%(低于随机的 50%)**;加入 meta-skill 三维度作为评分 rubric 后升到 **73.8%**——但仍每 4 次错 1 次

## 一句话总结

**模型自动生成的 skill 平均有价值但四分之一是负资产,且好坏与模型强弱无关;决定 skill 效用的是可操作的写作特征(失败编码 + 指令具体 + 反例黑名单),把这些特征做成 meta-skill 药方,就能稳定提升 skill 质量并抑制负迁移——但 LLM 对 skill 质量的自判仍不可靠,重要决策需要外部评估框架。**

## 与 darwin-skill 的关系

本文是 darwin-skill v2.0 的 9 维评分卡中三个维度的直接来源:

| 论文 meta-skill 维度 | 达尔文评分维度 | 权重 |
|---|---|---|
| Failure-mechanism encoding | dim3 失败模式编码 | 12 |
| Actionable specificity | dim5 可执行具体性 | 18 |
| Risk-action blacklist | dim9 反例与黑名单 | 6 |

同时,「LLM 自评仅 46.4%」是达尔文「独立 judge 子 agent + paired 盲评 + 人工守关」设计的实证依据。

---

*本文件由达尔文 skill 会话中整理生成,基于 arXiv:2605.23899 摘要、HTML 全文及本仓库 [references/skilllens-evidence.md](references/skilllens-evidence.md)。*
