---
name: semantic-trap-detector
description: "Scan Claude Skill instructions for ambiguous word choices that cause LLM hallucinations, identify high-risk phrasing using a semantic trap lexicon, and rewrite problematic sections with narrow-boundary replacements. Use when debugging skill prompts that produce incorrect outputs, reducing hallucination in skill responses, auditing skill wording for semantic drift, or hardening skill constraints against output leakage."
---

# 语义陷阱检测器 (Semantic Trap Detector)

## 概述

检测 Skill 中导致 LLM 幻觉的词汇选择问题。某些词汇对在日常语境中含义相近，但在 LLM 语义空间中激活范围差异巨大（如 "漏洞" vs "风险" 导致 27个百分点的准确率差异）。本检测器通过四步工作流识别高风险词汇和结构模式，提供替换建议和边界锚定策略。

完整词典和锚定策略见 [semantic-trap-lexicon.md](semantic-trap-lexicon.md)。测试用例见 [test-cases.md](test-cases.md)。

---

## 检测工作流

本检测器采用严格的四步工作流，**不得跳过或重新排序任何步骤**。

### STEP 1: 加载词典

**目标**：加载语义陷阱词典，建立检测知识基础

**执行动作**：
1. 读取 `semantic-trap-lexicon.md` 文件
2. 确认词典包含以下内容：
   - 17+ 组中文词汇对
   - 10+ 组英文词汇对
   - 语义敏感度矩阵
   - 边界锚定策略

**验证标准**：
- 词典文件成功加载
- 记录词典中的词汇对数量

---

### STEP 2: 提取关键词

**目标**：从待检测文本中提取四类高风险关键词

**执行动作**：

#### 2.1 核心术语提取
提取任务类型名词，如：
- 漏洞、风险、问题、缺陷、错误、异常
- Vulnerability, Risk, Issue, Defect, Error, Anomaly

#### 2.2 动作指令提取
提取指导大模型执行的动词或动宾短语：
- 检查、审查、分析、评估、探索、理解
- Check, Review, Analyze, Evaluate, Explore, Understand

#### 2.3 约束词提取
提取限定范围的词：
- 要求、建议、原则、规则、必须、应该
- Requirement, Suggestion, Must, Should, Principle

#### 2.4 输出描述词提取
提取描述期望输出的词：
- 列出、描述、总结、报告、洞察
- List, Describe, Summarize, Report, Insight

**验证标准**：
- 四类关键词全部提取完毕
- 每个关键词标注其在文本中的位置（行号、上下文）

---

### STEP 3: 逐词匹配与上下文分析

**目标**：对每个关键词执行两层判定，识别语义陷阱

#### 3.1 第一层判定：词典匹配

对每个提取的关键词：
1. 检查是否在宽边界词列表中
2. 如果匹配，记录陷阱ID、风险等级、推荐替换词

**风险等级分类**：
- **高危**：语义宽度差极大（如 漏洞 vs 风险）
- **中危**：语义宽度差大（如 检查 vs 审查）
- **低危**：语义宽度差中等

#### 3.2 第二层判定：上下文角色分析

同一个词在不同位置的危害程度不同：

**高风险上下文**：
- 核心约束语句中："只分析 X 类型"
- 任务主描述中："评估 X"
- 输出格式定义中："生成 X 报告"

**低风险上下文**：
- 辅助描述中："以分析性的视角"
- 示例说明中："以下是一个分析示例"

**判定规则**：
```
IF 词在宽边界词列表 AND 在高风险上下文
   → 风险等级提升一级
IF 词在宽边界词列表 AND 在低风险上下文
   → 风险等级降低一级
```

#### 3.3 疑似陷阱识别

即使词典中未收录，如果词汇满足以下宽边界特征，也应标记为疑似陷阱：

**宽边界词的共性特征**：
1. **程度性而非二元性**：如"风险"（高/中/低）vs"漏洞"（存在/不存在）
2. **带展望性**：如"分析"指向潜在可能性
3. **触发主观评价**：如"审查"包含价值判断
4. **关联网络松散**：如"问题"可关联到任何不够完美的事物

#### 3.4 结构性风险模式识别

识别四种额外的结构性陷阱：

| 模式类型 | 示例 | 检测方法 |
|---------|------|---------|
| 开放式动词 + 无范围限定 | "分析代码" | 检测动词后是否跟随具体对象 |
| 形容词化目标 | "评估安全性" | 检测抽象名词（安全性、质量性） |
| 情态动词降级 | "应该"而非"必须" | 检测约束条件中的情态动词 |
| 缺少否定清单 | 使用宽边界词但无排除列表 | 检测是否有"不包括"或"排除"章节 |

**验证标准**：
- 所有关键词完成两层判定
- 识别出所有结构性风险模式
- 生成完整的风险清单

---

### STEP 4: 生成检测报告

**目标**：输出结构化的检测报告和可执行的建议

#### 4.1 报告结构

```markdown
# 语义陷阱检测报告

## 1. 检测概览
- 检测文件：[文件名]
- 提取关键词数：[数量]
- 发现陷阱数：[数量]
  - 高危：[数量]
  - 中危：[数量]
  - 低危：[数量]
- 结构性风险：[数量]

## 2. 检测详情表

| 原词 | 陷阱ID | 风险等级 | 上下文 | 推荐替换 | 理由 |
|------|--------|---------|--------|---------|------|
| 风险 | T01 | 高危 | 核心术语 | 漏洞 | 语义宽度极大，导致范围溢出 |
| 审查 | T11 | 中危 | 动作指令 | 检查 | 触发主观评价性意见 |

## 3. 替换建议

Per trap: original sentence, identified traps with IDs, replacement sentence, reasoning.
For non-replaceable words, apply anchoring strategies from [semantic-trap-lexicon.md](semantic-trap-lexicon.md) (前置否定清单, 输出格式硬约束, 反例强化).

## 4. 优先级

1. 立即修复：所有高危陷阱
2. 尽快修复：中危陷阱
3. 可选优化：低危陷阱和疑似陷阱
```

#### 4.2 输出要求

**Final Answer 格式**：
```json
{
  "summary": {
    "total_keywords": 0,
    "traps_found": 0,
    "high_risk": 0,
    "medium_risk": 0,
    "low_risk": 0,
    "structural_risks": 0
  },
  "details": [
    {
      "word": "原词",
      "trap_id": "T01",
      "risk_level": "高危|中危|低危",
      "context": "上下文片段",
      "line_number": 行号,
      "replacement": "推荐替换词",
      "reason": "替换理由"
    }
  ],
  "structural_patterns": [
    {
      "pattern_type": "模式类型",
      "line_number": 行号,
      "example": "示例内容",
      "fix_suggestion": "修复建议"
    }
  ],
  "suggestions": [
    {
      "original": "原句",
      "fixed": "替换后",
      "reason": "替换理由"
    }
  ]
}
```

**验证标准**：
- 报告包含所有四个部分
- 每个陷阱都有具体的替换建议
- 无法替换的陷阱提供锚定策略
- Final Answer 符合 JSON 格式

---

---

## 约束与规则

1. **强制执行顺序**：STEP 1 → 2 → 3 → 4，不得跳过或重新排序
2. **完整性要求**：每步必须完成验证标准才能进入下一步
3. **词典依赖**：所有判定必须基于词典，不得凭空推断
4. **上下文敏感**：必须结合上下文角色判定风险等级
5. **可操作性**：所有建议必须具体可执行，不能是模糊的"优化建议"

---

## 质量保证

### 自检清单

使用本检测器时，应确认：
- [ ] 词典文件完整加载
- [ ] 四类关键词全部提取
- [ ] 每个词完成两层判定
- [ ] 识别所有结构性风险模式
- [ ] 生成符合格式的 Final Answer
- [ ] 每个陷阱都有可执行建议
