# Unicorn Harness

**一个人，百人团队的工程判断力。**

从生产级系统（Claude Code v2.1.88）中提炼的 7 条原则，泛化为适用于任何产品领域的思维框架——SaaS、CLI/DevTools、Agent 编排、AIGC、电商、内容平台、产业数字化。

> "我们不仅要孵化独角兽，更要构建一台能够持续裂变独角兽的母体引擎。"
> — Nebutra 宣言

## 这是什么

一个 [Claude Code SKILL](https://docs.anthropic.com/en/docs/claude-code)，升级你的工程决策能力。不是教程，不是检查清单，是思维操作系统。

"能写代码"和"能做出独角兽级产品"之间的差距不在技术能力，在思维方式。这个 SKILL 将 10 亿美金级产品背后的思维模式编码成 7 条可操作的原则。

## 7 条原则

| # | 原则 | 一句话 |
|---|------|--------|
| 1 | **最小干预** | 永远先用最轻的手段，只在便宜的手段失败后才升级 |
| 2 | **边界即产品** | 你画的每条线都在塑造用户体验 |
| 3 | **生命周期优先** | 先设计事物如何诞生、存活、死亡，再写代码 |
| 4 | **渐进赢得信任** | 永远不要一上来就要求全部信任 |
| 5 | **约束即燃料** | 限制是设计的起点，不是妥协的借口 |
| 6 | **策略写进代码** | 写在 wiki 里的规则会被违反，写进编译器的不会 |
| 7 | **先有灵魂再谈规模** | 产品人格是地基，不是装饰 |

## 快速开始

### 作为 Claude Code SKILL 使用

```bash
# 作为 git submodule 添加到项目
git submodule add https://github.com/Nebutra/claude-code-harness.git .claude/skills/claude-code-harness

# 或直接复制到 skills 目录
cp claude-code-harness/SKILL.md ~/.claude/skills/claude-code-harness/SKILL.md
```

### 直接阅读原则

每条原则都有一个深度参考文档：

- [01 — 最小干预](references/01-minimum-intervention.md)
- [02 — 边界即产品](references/02-boundary-is-product.md)
- [03 — 生命周期优先](references/03-lifecycle-not-function.md)
- [04 — 渐进赢得信任](references/04-earn-trust-progressively.md)
- [05 — 约束即燃料](references/05-constraint-as-fuel.md)
- [06 — 策略写进代码](references/06-policy-in-code-not-wiki.md)
- [07 — 先有灵魂再谈规模](references/07-soul-before-scale.md)

每个参考文档覆盖：原则解释、为什么重要、Claude Code 如何实现、如何迁移到你的产品、反模式、自检问题。

**补充深度参考**（专项领域）：

- [08 — 缓存感知架构](references/08-cache-aware-architecture.md) — Prompt Cache工程、前缀共享、缓存优先设计
- [09 — 多Agent协调](references/09-multi-agent-coordination.md) — 3层协作架构、权限传递、通信模式
- [10 — 流式与实时系统](references/10-streaming-and-realtime.md) — AsyncGenerator、背压、渐进渲染

## 使用场景

| 你在做什么 | 重点关注 |
|-----------|---------|
| 从 0 到 1 搭新产品 | 灵魂 → 约束 → 边界 |
| 设计 API / 系统架构 | 边界 → 生命周期 → 策略 |
| 性能 / 成本优化 | 最小干预 → 约束 |
| 权限 / 安全设计 | 信任 → 边界 → 策略 |
| 构建 Agent / AI 产品 | 全部，特别是干预 + 生命周期 |
| 识别技术债 | 全部——用自检问题作为审计清单 |
| 上线前审查 | 7 条全过一遍 |

## 基准测试

4 轮测试、6 个 subagent、2 个真实任务（Agent 架构设计 + 生产代码库技术债审计），对比：无 SKILL / v1 / v2。

### Agent 架构设计任务

| 指标 | 无 SKILL | v2 SKILL |
|------|----------|----------|
| Tokens | 23K | 33K (+43%) |
| 边界推理 | 列出组件职责 | 解释每条边界的语义差异 + 驳斥3种常见错误切法 |
| 错误处理 | 决策树（分类→路由） | 5级干预阶梯（retry→replan→human→fail） |
| 类型安全 | 标准接口 | 不透明 `BudgetToken` 强制编译期预算检查 |
| 崩溃恢复 | 提及checkpoint | 每个状态回答"进程在这里crash怎么办" + two-phase commit |
| 反模式 | 无 | 5个反模式 + 为什么看起来对但实际错 |

### 技术债审计任务（真实生产 monorepo）

| 指标 | 无 SKILL | v2 SKILL |
|------|----------|----------|
| Tokens / tool 调用 | 116K / 113 | 87K / 52 (-25% / -54%) |
| 独特发现 | Zod 版本分裂、幽灵 Schema | S2S HMAC 断裂、mock 支付路径、硬编码 plan limit |
| 修复建议 | 方向性描述 | 内联代码修复 + "今天就修" vs "下个sprint" 分级 |
| 洞察质量 | 描述现状 | "半完成的好意图比没开始更危险——制造安全幻觉" |

### 诚实的局限

- **+43% token 成本**（架构任务）——高stakes决策值得，CRUD 不需要
- **Sonnet/Opus 效果最好**——小模型可能退化
- **不替代领域知识**——提升架构判断力，不替代业务理解
- **简单任务边际收益低**——写工具函数不需要7条原则

### 何时使用 / 何时跳过

| 使用 | 跳过 |
|------|------|
| 从 0 到 1 架构设计 | 简单 bug 修复 |
| 技术债审计 | 写单元测试 |
| Agent/AI 产品设计 | CRUD 接口 |
| 上线前系统 Review | 日常 coding |
| Demo 升级为生产系统 | 文档更新 |

## Nebutra 生态中的位置

```
Nebutra 宣言（哲学层）    → "无限裂变，而非无限扩张"
Unicorn Harness（认知层）  → 怎么想     ← 本仓库
Nebutra Sailor（实现层）   → 按这 7 条原则构建的 53 个模块
Sleptons（社区层）         → OPC 生态
```

## 贡献

欢迎提交新领域的迁移示例。每条原则的参考文档都有"迁移到你的产品"段落——如果你的领域没有被覆盖，欢迎提 PR。

## 许可证

MIT — 思维框架应该无摩擦传播。
