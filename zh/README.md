# Blockchain Supply-Chain Finance — 独立安全审计报告

> Independent Security Audit Report
> 审计对象：Blockchain Supply-Chain Finance（Solana / Anchor 0.31.1）
> 审计性质：独立安全审计（非持牌机构交付物；方法论参考 Trail of Bits / CertiK）
> 报告版本：v1.0 — 2026-08-08 复测基线
> 审计师：sudorm-rf0

---

## 0. 免责声明与负责任披露（Responsible Disclosure）

- 本报告为审计师独立评估，**非持牌第三方审计机构交付物**，不构成对项目安全性、合规性或商业可行性的保证。
- 所有列出的 **Critical / High 漏洞均已代码级修复并通过复测后**才公开；exploit 描述保持概念级，**不提供可直接复现的攻击脚本**。
- Program ID 与部署参数在公开版中做脱敏 / 参数化处理。完整审计包（合约源码、Anchor 测试、部署信息、无私钥）按负责任披露流程提供给项目方与监管机构。
- 项目当前处于 **主网前（pre-mainnet）** 阶段。任何对核心合约逻辑（业务常量、账户结构、指令签名）的修改都应触发重新审计。

---

## 1. 执行摘要（Executive Summary）

对 Blockchain Supply-Chain Finance 的 Solana / Anchor 智能合约进行了独立安全审计。系统包含贸易订单全生命周期（创建-放款-物流-还款）、LP 资金池存取赎回、链上单据存证、供应商授权等模块，将承载真实资金交易。

**核心结论：**

- 经多轮独立审计（内部评估 + 第三方 DFR 四轮复测 + 2026-08-08 独立复测），共识别并修复 **1 项 Critical 与 6 项 High** 级别漏洞。
- **全部 Critical / High 均已代码级修复并通过复测**，当前无未修复的 Critical / High。
- 剩余项为**上线前治理 / 部署决策**（多签管理员、≥48h 管理员时锁、费率 / 分成 / 首损参数业务确认、独立第三方终稿），**不构成未修补漏洞**。
- 在落实 §8 治理门槛后，合约层具备上线条件。

> ⚠️ 重要更正：早期内部自评（AUDIT-REPORT.md）因范围局限给出 **B+（无 Critical/High）** 的乐观结论；第三方审计 **DFR-SC-2026-0143 出具否定意见（1 Critical + 4 High）**，并指出内部评估对 `C-01`（LP 铸币权）与 `H-03`（管理员集中）定级偏低（DFR 发现 `I-04`）。本报告以多轮复测后的真实状态为准。

---

## 2. 审计范围（Scope）

| 组件 | 路径 | 语言 | 规模 |
|------|------|------|------|
| trade-finance 合约 | `programs/trade-finance/src/` | Rust (Anchor 0.31.1) | ~1345 行 |
| supply-chain 合约 | `programs/supply-chain/src/` | Rust (Anchor 0.31.1) | ~337 行 |
| 合约集成测试 | `programs/tests/` | TypeScript (Anchor TS) | 46 → 66 用例 |
| 后端 / 微服务 | `packages/backend/` | TypeScript (NestJS 11) | 抽样 |
| 前端 | `packages/frontend/` | TypeScript (Next.js 15) | 抽样 |
| 基础设施 | `docker-compose.yml` / `k8s/` / `infra/` | YAML/Docker | 抽样 |

**重点路径：** LP 铸币 / 赎回、费用分配与分红、违约清算保险路径、管理员权限与时锁、初始化守卫、链下密钥与网络策略。

---

## 3. 方法论（Methodology）

1. **静态分析** — 逐指令通读全部合约，重点审查算术安全（`checked_*`）、访问控制、状态机完整性。
2. **动态验证** — Rust 单元测试 + Anchor TS 集成测试（覆盖边界、暂停、管理员轮换、审计整改回归）。
3. **账本不变式推演** — 维护核心不变式 `INV-1: total_assets = vault + 托管 + active_capital`，对每条资金路径做增量断言。
4. **架构 / 威胁建模** — 针对违约、还款、赎回、初始化、管理员轮换构建攻击向量清单。
5. **链下配置复核** — JWT/Webhook 密钥、NetworkPolicy、Webhook 签名防重放、TOTP 两步验证。

---

## 4. 严重级模型（Severity Model）

| 等级 | 定义 |
|------|------|
| **Critical** | 可直接导致资金永久丢失、无限铸造、或整个系统被接管 |
| **High** | 在特定条件下可能导致资金损失或系统故障 |
| **Medium** | 影响安全假设或数据一致性，但不直接导致资金损失 |
| **Low** | 代码质量、边缘场景不完善 |
| **Informational** | 优化建议，不构成安全威胁 |

---

## 5. 发现汇总（Findings Summary）

| 编号 | 标题 | 严重性 | 状态 |
|------|------|--------|------|
| **C-1** | 初始化未绑定 ProgramData → 伪造 upgrade authority 抢跑夺权 | Critical | ✅ 已修复并复测 |
| **H-1** | 初始化抢跑（未绑定部署方 / upgrade authority） | High | ✅ 已修复并复测 |
| **H-2** | `test-deployer` 特性后门（硬编码 DEPLOYER 初始化后门） | High | ✅ 已根除 |
| **H-3** | `deposit_pool` 实时余额定价 → 捐赠抬价抽水 | High | ✅ 已修复并复测 |
| **H-4** | 费率未参数化（首损层 / 分配比例硬编码） | High | ⚠️ 合约层完成，参数待业务确认 |
| **H-5** | `set_admin_delay` 时锁自废后门 | High | ✅ 已修复（硬下限 86400s） |
| **H-06** | supply-chain 管理员时锁可置零（同类 H-5） | High | ✅ 已修复（硬下限 86400s） |
| M-01~M-12 | 不变式 / 保险 / NAV / 赎回窗口 / 集中度等 | Medium | ✅ 已修复 |
| N-01~N-06 | NAV 基准 / 时锁参数 / 多签 / 种子碰撞 / DEPLOYER / 首损 | Medium/Low | ✅ 已修复 |
| L-01~L-13 | 单位 / 事件 / 幂等 / 暂停守卫 / 终态等 | Low | ✅ 已修复 |

> 早期专项审计（CONTRACT-AUDIT.md, 2026-08-06）另识别 4 项 High 资金/账本错误（H1 费用未收足、H2 违约垫付卡死、H3 释放未扣抵押金、H4 虚增总资产），均已修复并纳入 49→66 集成测试守护。

---

## 6. 关键发现详述

- [Critical — C-1 初始化夺权](findings/critical.md)
- [High — H-1 / H-2 / H-3 / H-4 / H-5 / H-06](findings/high.md)
- [Medium / Low 汇总](findings/medium-low.md)

---

## 7. 整改进度（Remediation Timeline）

| 时间 | 里程碑 | 结果 |
|------|--------|------|
| 2026-08-06 | 内部专项审计（CONTRACT-AUDIT） | 修复 H1–H4 资金账本错误 |
| 2026-08-07 | 第三方 DFR 首轮（SC-2026-0143） | 否定意见（1C+4H）；指出内部定级偏低 |
| 2026-08-07 | DFR 四轮复测（0143→0144→0145→0146） | 全部 C/H 代码级修复 + 复测通过 |
| 2026-08-08 | 独立复测（SECURITY-AUDIT-REPORT） | 修复 C-1/H-2/H-3 等，Anchor 66/66 通过 |

**当前状态：无未修复 Critical / High。**

---

## 8. 上线前门槛（Pre-launch Gates — 非漏洞）

以下为治理 / 部署决策，需在主网灰度前落实：

1. **多签管理员**：`pool.admin` / `registry.admin` 指向 Squads 多签 PDA（≥3/5），由 `precheck-mainnet-deploy.sh` 与 MAINNET-MIGRATION.md 强制。
2. **管理员时锁 ≥ 48h**：`ADMIN_TRANSFER_DELAY_SECS` / `MIN_ADMIN_DELAY_SECS` 上线前设为 ≥ 86400s（代码已支持硬下限）。
3. **费率 / 分成 / 首损参数**：目标违约率、首损池注资规模、费率最终值由业务与风控确认（H-4 合约层已参数化）。
4. **独立第三方终稿**：由持牌机构出具最终审计报告，作为上线硬门槛。

---

## 9. 结论（Verdict）

> **Conditional Go（有条件通过）**：在落实 §8 全部治理门槛后，合约层具备主网部署条件。建议以独立第三方机构最终报告作为上线硬门槛，上线后保留链上 / DB 每日对账（`scripts/reconcile`）与监控告警。

---

## 10. 参考材料（References）

- `AUDIT-REPORT.md` — 内部安全评估（B+，已注明结论过时）
- `CONTRACT-AUDIT.md` — 2026-08-06 贸易金融专项审计
- `AUDIT-KNOWN-RISKS.md` — DFR 整改追踪（SC-2026-0143 → 0146）
- `AUDIT-REMEDIATION.md` — 2026-08-08 独立复测整改
- `AUDIT-DELIVERY.md` — 审计状态与材料包说明

---

*本审计报告基于审计期间提供的代码版本。任何核心逻辑修改后应重新评估。*
