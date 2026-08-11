# High 级别发现（H-1 ~ H-06）

> 全部 High 均已代码级修复并通过复测，当前无未修复 High。

---

## H-1 — 初始化抢跑（High）

**位置：** `initialize_pool` / `initialize_registry`
**描述：** 初始化指令未绑定部署方或程序 upgrade authority，攻击者可在项目方之前抢先初始化并占据管理员。
**修复：** 初始化者必须匹配部署方白名单或程序 upgrade authority（与 C-1 协同，构成初始化阶段纵深防御）。
**状态：** ✅ 已修复并复测。

---

## H-2 — `test-deployer` 特性后门（High）

**位置：** 构建配置（`test-deployer` feature / `DEPLOYER` 常量）
**描述：** 若 `test-deployer` 特性被编译进主网构建，硬编码的 `DEPLOYER` 测试钱包将成为初始化后门——任何知道该常量的人都能以"部署方"身份初始化并接管管理员。
**修复（根除）：**
- 删除 `test-deployer` feature 与 `DEPLOYER` 常量；
- upgrade authority 校验统一为「**初始化者必须等于程序 upgrade authority**」，UA 为 `None`（冻结）时一律拒绝，无任何回退路径；
- 测试改用 `test-build`（仅放宽 `initial_delay` 下限，无白名单）。
**状态：** ✅ 已根除（N-05 / L-13 同轮整改）。

---

## H-3 — `deposit_pool` 实时余额定价 → 捐赠抬价抽水（High）

**位置：** `deposit_pool`（及全部 8 个资金指令的定价入口）
**描述：** 份额定价以**实时金库余额**为基准。攻击者可在首存后向金库直接捐赠 USDC，抬高 `equity_base` / 份额定价基准，使后续存款人按虚高价格获得更少份额——首存者 + 捐赠者可抽干后续存款人的价值。
**修复：**
- `PoolState` 新增权威记账 `tracked_vault`，**仅由程序出入金路径更新**；
- 全部 8 个资金指令同步维护 `tracked_vault`；
- `deposit` / `redeem` 等入口校验 `实时余额 == tracked_vault`，不一致则 `VaultMismatch` 拒绝外部捐赠；
- 新增 H-3 捐赠回归测试。
**状态：** ✅ 已修复并复测。

---

## H-4 — 费率 / 首损未参数化（High，参数待业务确认）

**位置：** `repay_deal` 费用分配、首损层
**描述：** 费率、平台 / LP / 买方分成比例、首损准备金规模此前多为硬编码业务常量，缺乏治理可调性与首损隔离层，极端参数下存在系统性偏差 / 经济模型风险。
**修复（合约层已完成）：**
- `fee_apy_bps` 参数化（默认 670 = 6.7% APR）；
- 分配比例可治理（`set_fee_params`）；
- `first_loss_reserve` 首损层（`deposit_first_loss` / `withdraw_first_loss`）；
- 单笔费率上限 `MAX_SINGLE_FEE_BPS`（5%）；
- NAV 口径剔除首损；`repay_deal` 按日计息记账修正（dust 偏差）。
**待办（业务 / 治理决策，非漏洞）：** 目标违约率（默认 2%）、首损池实际注资规模、费率最终值，需业务与风控基于真实违约数据确认。
**状态：** ⚠️ 合约层完成，参数待业务确认（见 §8 上线前门槛）。

---

## H-5 — `set_admin_delay` 时锁自废后门（High）

**位置：** `set_admin_delay`（trade-finance 管理员轮换时锁）
**描述：** 管理员轮换的时锁（delay）可被设置为 0，攻击者在接管管理员后可**立即**完成时锁内的转移，使 "两步轮换 + 时锁" 防护失效。
**修复：** `set_admin_delay` 设硬下限 `MIN_ADMIN_DELAY_SECS = 86400`（1 天），不得下调；初始时锁由 `initialize_pool` 注入（生产 48h）。
**状态：** ✅ 已修复。

---

## H-06 — supply-chain 管理员时锁可置零（High，同类 H-5）

**位置：** `set_registry_admin_delay`（supply-chain Registry 管理员轮换时锁）
**描述：** 与 H-5 同源问题——supply-chain 的管理员时锁可被置零，使 Registry 管理员两步轮换防护失效。
**修复：** `set_registry_admin_delay` 设硬下限 `MIN_REGISTRY_ADMIN_DELAY_SECS = 86400`；`initialize_registry` 初始时锁由部署方注入（生产 48h）。
**状态：** ✅ 已修复。

---

## 来源

- `AUDIT-REMEDIATION.md` §〇（C-1 / H-2 / H-3 / M-3 / N-01）
- `AUDIT-KNOWN-RISKS.md`（H-01~H-06、N-01~N-06、L-13 整改追踪）
- `CONTRACT-AUDIT.md`（H1–H4 资金账本错误，已修复）
