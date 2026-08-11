# Medium / Low 级别发现汇总

> 以下发现均来自多轮审计（内部评估 + 第三方 DFR 四轮复测 + 2026-08-08 独立复测），**当前均已代码级修复**。本节为汇总，详略以高风险路径为主。

## Medium（M 系列，来自 DFR 整改）

| 编号 | 问题 | 修复 |
|------|------|------|
| M-01 | 不变式失效：`fund_deal` 重复计入 `active_capital` | 新增 `escrow_funded`；恒等式 `total_assets = vault + 托管 + active` 成立 |
| M-02 | 保险重复计入：`default_deal` 已释放路径虚增 `total_assets` | 已释放路径不再虚增保险赔付 |
| M-03 | 现金与记账脱钩 | `fund_deal` 增加 `vault_after >= insurance_fund` 校验 |
| M-04 | NAV / 赎回价不一致 | 新增 `redemption_price` 字段分别披露；赎回事件携带两价 |
| M-05 | 赎回无窗口 | 新增 `redeem_window_epoch/used`，周期内累计上限 50% |
| M-06 | 存证未隔离 | `document` PDA 纳入 buyer 种子；`deal` 改为必选 |
| M-07 | 自融资闭环 | `create_deal` 增加 `seller != buyer`、`seller != default` 校验 |
| M-08 | 集中度基数偏差 | 改为基于可用流动性（`vault - insurance - pending`） |
| M-09 | mint 不可迁移 | 新增 `set_lp_mint`（需暂停 + 无在途 + 旧 supply=0 + 属性校验） |
| M-10 | 除零 DoS | `redeem_lp` 增加 `total_before > 0` 校验 |
| M-11 | supply 不可轮换 | supply-chain 新增 `transfer_admin` |
| M-12 | supply-chain 一步管理员转移 | 改为两步轮换 + 锁定期 + `set_registry_admin_delay` |

## Medium（N 系列，来自 DFR 复测 / 独立复测）

| 编号 | 问题 | 修复 |
|------|------|------|
| N-01 | NAV 与份额铸造 / 赎回用价不一致 | 铸造 / 赎回统一为纯现金权益基准 `equity_base = vault - first_loss` |
| N-02 | 管理员转移时锁为 0 | `pending_admin_delay_secs` 参数化，默认 172_800（48h） |
| N-03 | 单管理员无多签 / 角色分离 | `admin` 可指向 Squads 多签 PDA（代码已支持，**需上线部署**） |
| N-04 | `sku_seed` 8 字节碰撞 | 改用完整 SHA-256（32 字节）种子 |
| N-05 | `DEPLOYER` 硬编码测试钱包 | 初始化规则改为 upgrade authority 必须匹配（冷钱包），仅 UA 冻结时回退 |
| N-06 | 首损计入 LP 权益 | 铸造 / 赎回 / `redemption_price` 全部基于 `equity_base`，首损不可被 LP 赎回 |
| L-1 | `repay_deal` 强制 `fee>0` 致微单 DoS | 允许 `fee=0` 纯还本路径 |
| M-3 | 先冻结 UA 后初始化 → 部署变砖 | `deploy-mainnet.sh` 冻结前校验 PDA 已在链上，否则中止 |

## Low（L 系列）

- **L-02** 终态无 close → 新增 `close_deal`；`set_status` 对 `DEFAULTED` 同样终态锁定
- **L-03** 事件缺失 → `create_deal` / `deposit_pool` / `default_deal` 补事件
- **L-06** `attest_document` 不受暂停 → 增加暂停守卫
- **L-08** `authorize_supplier` 非幂等 → 改为 `init_if_needed`
- **L-11** 分红接收方自由指定 → 仅允许向 LP 持有者发放，单次不超其按比例应得
- **L-12** 治理安全控制测试覆盖缺口 → 补 H-06 攻击路径回归测试
- **L-13** 初始化时锁下限 → 生产构建强制 `>= 86_400s`
- 其余 L-01（tenor 单位）、L-04（逾期罚息）、L-05（LP decimals，已随 C-01 完成）、L-07（清盘路径）、L-09（商品撤销标记）列为后续版本 / 设计决策

## 早期专项审计（CONTRACT-AUDIT.md, 2026-08-06）

4 项 High 资金 / 账本错误，已修复并纳入 49→66 集成测试守护：

- **H1** 还款未收足费用 → LP 分红无资金支撑（改为全额费用进池后分配）
- **H2** 违约后 70% 垫付款永久卡死托管（违约时托管整笔回池）
- **H3** 释放后未扣减已付卖方抵押金 → 账面虚增（释放时 `total_assets -= down_payment`）
- **H4** 创建订单把未放款 100% 计入总资产（仅增 `down_payment`，`fund_deal` 内部划转）

---

*全部 Medium / Low 整改后测试：Rust 22/22、Anchor 66/66、后端 Jest 155/155、前端 Vitest 48/48。*
