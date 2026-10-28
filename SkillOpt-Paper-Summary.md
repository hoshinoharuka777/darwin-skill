# SkillOpt 论文总结

> **SkillOpt: Executive Strategy for Self-Evolving Agent Skills**
> Microsoft + 上海交大 + 同济 + 复旦,15 位作者(一作 Yifan Yang)· 2026-05-22(v2 修订 05-25)
> [arXiv:2605.23904](https://arxiv.org/abs/2605.23904) · 代码:[aka.ms/SkillOpt](https://aka.ms/SkillOpt)(`pip install skillopt`)

---

## 研究问题

Agent skill 目前有三种生产方式:人工手写、LLM 一次性生成、松散的自迭代进化——**没有一种像深度学习优化器那样对待 skill,也没有一种能在反馈下保证单调变好**。

论文核心主张:

> skill 应当被当作**冻结 agent 的外部可训练状态**来「训练」,复用使权重空间优化可复现的那套纪律。

与相邻工作的差异:GEPA/ABSTRAL/EvoTest 等优化的是 prompt、系统设计或完整配置;Trace2Skill、EvoSkill 等强调 skill 的发现、仓库增长与进化搜索。SkillOpt 研究的是一个更窄的问题:**如何用训练式控制手段(train/selection/test 划分、文本学习率、验证门控、拒绝缓冲、慢速动量)把一份紧凑的领域 skill 训练到最优,并导出为可移植、可审计的 `best_skill.md`。**

## 方法:把 DL 优化器逐组件搬进文本空间

SkillOpt 是首个系统化的**可控文本空间优化器**。skill 文档 `s` 是插入 agent 上下文的自然语言策略;目标模型 `M` 冻结不动,由一个额外的前沿优化器模型驱动训练循环。每个 DL 概念都有严格的文本对应物:

| 深度学习组件 | SkillOpt 对应机制 |
|---|---|
| 前向传播 | 目标模型带当前 skill 跑 **rollout batch**,产生轨迹 + 评分 |
| 反向传播/梯度 | 优化器模型对成功/失败轨迹做 **minibatch reflection**,提出结构化的 **add/delete/replace 编辑** |
| 学习率 | **textual learning-rate budget**:限制单次编辑后 skill 版本能移动多远 |
| 验证集门控 | **held-out validation gate**:编辑只有在**严格提升 held-out 选择集分数时才被接受** |
| 梯度截断/拒绝样本 | **rejected-edit buffer**:被拒编辑保留为负反馈,供后续轮次参考 |
| 动量 | **epoch-wise slow/meta update**:跨 epoch 保留长期稳定的编辑方向 |
| 部署开销 | **零额外推理调用**——只导出一个 300~2000 token 的 `best_skill.md` |

作者强调这个类比是**操作性的而非装饰性**:只有当相邻版本的编辑距离足够近、方向一致,「优化历史」(哪些编辑成功/失败)才对后续优化器有可学习的意义。

**数据划分与优化器状态**:

- train(产生经验)/ selection(门控更新)/ test(仅最终报告)
- 优化器状态包含:当前 skill、最佳已验证 skill、缓存哈希、epoch 内拒绝缓冲、可选 slow/meta 更新状态;只导出最佳已接受 skill

## 实验结果

**设置**:6 个基准(QA/表格/文档/数学/具身决策)× 7 个目标模型(GPT 到 Qwen)× 3 种执行环境(direct chat / Codex / Claude Code)= 52 个组合。

**① 全面最优**:52 个 (model, benchmark, harness) 组合**全部 best 或 tied-best**,平均超过每格最强对手(人工 skill、一次性 LLM、Trace2Skill、TextGrad、GEPA、EvoSkill)**+5.4 分**。

**② 增益巨大**(GPT-5.5,direct chat,no-skill → with skill):

| 基准 | 提升 |
|---|---|
| SearchQA | 77.7 → 87.3 |
| SpreadsheetBench | 41.8 → **80.7** |
| OfficeQA | 33.1 → **72.1** |
| DocVQA | 78.8 → 91.2 |
| LiveMathematicianBench | 37.6 → 66.9 |
| ALFWorld | 83.6 → 95.5 |

平均 **+23.5 分**;agentic 环境下 Codex **+24.8**、Claude Code **+19.1**(对 EvoSkill 分别 +14.0 / +3.2)。

**③ 三种迁移全部成立**(skill 优化一次,到处复用,不改权重):

- **跨模型规模**:GPT-5.4 上训练的 SpreadsheetBench skill 让所有更小的 GPT 变体受益
- **跨执行环境**:Codex 训练的 skill 迁到 Claude Code,增益 **+59.7 分**
- **跨任务**:OlympiadBench skill 迁到 Omni-MATH 仍有正增益

**④ 消融验证每个组件都必要**:

- 有界文本更新优于不受控重写
- held-out 门控阻止有害编辑累积
- 拒绝缓冲把失败编辑转化为负反馈
- slow/meta 更新支持长程优化而不使 skill 膨胀

**⑤ 产物紧凑可审计**:仅需 **1~4 次被接受的编辑**即收敛,最终 skill 是 300~2000 token 的过程性知识(而非针对具体题目的补丁),人类可全文审读。

## 一句话总结

**SkillOpt 把 skill 从「写出来的文档」变成「训练出来的产物」:冻结模型、用优化器模型提出有界编辑、只接受严格提升验证分的修改、用学习率预算/拒绝缓冲/慢速动量保证训练稳定——在 52 个组合上全面击败人工与所有自动化基线,且优化后的 skill 可跨模型、跨环境、跨任务迁移,部署零开销。**

## 与 darwin-skill 的关系

SkillOpt 提供给达尔文的是 **validation-gated edits 形式化框架**——「keep/revert 棘轮」机制的直接来源:每轮单维改动必须通过盲评胜出才保留,否则回滚,分数只升不降。

达尔文与 SkillOpt 的关键差异:SkillOpt 全自动,达尔文坚持 human-in-the-loop,在基线评估、单维优化、回归测试三处强制人审——skill 质量比 validation loss 更微妙,judge(即使准确率 73.8%)仍每 4 次可能错 1 次。

2026-06-03 微软在 SkillOpt 官方仓库将 darwin-skill 列入集成名单(`"gbrain, gbrain-evals, and darwin-skill have all integrated SkillOpt."`),形成双向印证。

---

*本文件由达尔文 skill 会话中整理生成,基于 arXiv:2605.23904v2 摘要与 HTML 全文。姊妹篇总结见 [SkillLens-Paper-Summary.md](SkillLens-Paper-Summary.md)。*
