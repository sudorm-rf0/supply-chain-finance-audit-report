# C-1 — 初始化未绑定 ProgramData，可伪造 upgrade authority 抢跑夺权（Critical）

**位置：** `initialize_pool` / `initialize_registry`（trade-finance 与 supply-chain 两个程序）

**严重性：** Critical（系统接管）

## 描述

两个程序的初始化指令在创建 `PoolState` / `Registry` 账户时，未将传入的 `program_data` 账户与**本程序自身的 ProgramData PDA** 绑定校验。攻击者可构造一笔抢占初始化交易，传入**自己控制程序的 ProgramData**，从而让合约记录的 `upgrade authority` 指向攻击者账户。

一旦 `upgrade authority` 被替换为攻击者地址：

- 攻击者可升级（替换）合约程序逻辑，植入恶意指令；
- 或结合管理员权限路径，直接接管资金池与注册中心，挪走全部托管 / 池内资金。

这是典型的 **upgrade-authority 接管 + 初始化抢跑** 组合，定级 Critical。

## 影响

- 资金池管理员与程序升级权双双落入攻击者之手；
- 可导致全部在管资产（USDC / LP / 托管中的贸易订单资金）被提取或合约被替换为恶意逻辑；
- 属于"整个系统被接管"级别。

## 修复

初始化前强制：

```
program_data.key() == find_program_address(["program", <本程序 ID>], BPF_LOADER_UPGRADEABLE)
```

- 仅接受本程序自身的 ProgramData PDA，拒绝任何外部 / 伪造的 `program_data`；
- 新增 Rust 单元测试锚定该 PDA，断言传入错误 `program_data` 被拒绝；
- 与 **H-1** 协同：`initialize_pool` / `initialize_registry` 同时绑定部署方白名单或程序 upgrade authority，杜绝抢跑与伪造。

## 状态

✅ **已修复并复测** — Anchor 集成测试 66/66 通过（含初始化守卫回归）。

## 来源

`AUDIT-REMEDIATION.md` §〇 C-1；`AUDIT-KNOWN-RISKS.md` C-01（LP 铸币权链下，同轮整改）。

---

> 关联发现 **H-1（High）**：初始化抢跑 —— `initialize_pool` / `initialize_registry` 在修复前未绑定部署方或 upgrade authority，允许攻击者抢先初始化并占据管理员。修复方式：初始化者必须匹配部署方白名单或程序 upgrade authority。与 C-1 共同构成"初始化阶段"的纵深防御。
