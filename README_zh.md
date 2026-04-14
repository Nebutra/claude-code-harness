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
git submodule add https://github.com/user/unicorn-harness.git .claude/skills/unicorn-harness

# 或直接复制到 skills 目录
cp unicorn-harness/SKILL.md ~/.claude/skills/unicorn-harness/SKILL.md
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

## 经过 TDD 验证

这个 SKILL 使用真实 prompt 进行了 A/B 对比测试（Agent 架构设计、生产代码库技术债审计）。

结果：
- 架构设计展现了更深的边界推理和干预成本阶梯
- 技术债审计找到了更多可操作的 bug（HMAC 不匹配、硬编码限额、生产路径中的 mock 代码）
- 原则被内化，而非鹦鹉学舌——输出中没有"应用原则 X"的标签
- Token 效率逐迭代提升（同一审计任务 52 次 tool 调用 vs 113 次）

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
