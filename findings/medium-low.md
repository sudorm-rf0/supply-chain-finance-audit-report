# Medium / Low Severity Findings Summary

> All findings below come from multiple audit rounds (internal assessment + third-party DFR four-round re-test + independent re-test 2026-08-08) and **are fixed at the code level**. Summary only; high-risk paths described in more detail.

## Medium (M series, from DFR remediation)

| ID | Issue | Fix |
|----|-------|-----|
| M-01 | Invariant broken: `fund_deal` double-counted `active_capital` | Added `escrow_funded`; invariant `total_assets = vault + escrow + active` holds |
| M-02 | Insurance double-counted: `default_deal` unreleased path inflated `total_assets` | Unreleased path no longer inflates insurance payout |
| M-03 | Cash/bookkeeping decoupled | `fund_deal` adds `vault_after >= insurance_fund` check |
| M-04 | NAV / redeem price inconsistent | Added `redemption_price` field, disclosed separately; redeem event carries both prices |
| M-05 | No redeem window | Added `redeem_window_epoch/used`, 50% cumulative cap per window |
| M-06 | Notarization not isolated | `document` PDA includes buyer seed; `deal` made required |
| M-07 | Self-financing loop | `create_deal` adds `seller != buyer`, `seller != default` checks |
| M-08 | Concentration base biased | Switched to available liquidity (`vault - insurance - pending`) |
| M-09 | Mint not migratable | Added `set_lp_mint` (requires pause + no in-flight + old supply=0 + attr checks) |
| M-10 | Divide-by-zero DoS | `redeem_lp` adds `total_before > 0` check |
| M-11 | supply not rotatable | supply-chain added `transfer_admin` |
| M-12 | supply-chain one-step admin transfer | Changed to two-step rotation + lock + `set_registry_admin_delay` |

## Medium (N series, from DFR re-test / independent re-test)

| ID | Issue | Fix |
|----|-------|-----|
| N-01 | NAV vs share mint/redeem price inconsistent | Mint/redeem unified to pure cash-equity basis `equity_base = vault - first_loss` |
| N-02 | Admin-transfer timelock zero | `pending_admin_delay_secs` parameterized, default 172_800 (48h) |
| N-03 | Single admin, no multisig / role split | `admin` can point to Squads multisig PDA (code supported, **needs deployment**) |
| N-04 | `sku_seed` 8-byte collision | Switched to full SHA-256 (32-byte) seed |
| N-05 | Hardcoded `DEPLOYER` test wallet | Init rule requires upgrade authority to match (cold wallet); falls back only when UA frozen |
| N-06 | First-loss counted into LP equity | Mint/redeem/`redemption_price` all based on `equity_base`; first-loss not redeemable by LP |
| L-1 | `repay_deal` forced `fee>0` → micro-order DoS | Allow `fee=0` pure-principal path |
| M-3 | Freeze UA then init → brick deployment | `deploy-mainnet.sh` verifies PDA on-chain before freeze, else aborts |

## Low (L series)

- **L-02** no terminal `close` → added `close_deal`; `set_status` also terminal-locks `DEFAULTED`
- **L-03** missing events → added events to `create_deal` / `deposit_pool` / `default_deal`
- **L-06** `attest_document` ignored pause → added pause guard
- **L-08** `authorize_supplier` non-idempotent → changed to `init_if_needed`
- **L-11** free dividend recipient → only LP holders, capped per pro-rata share
- **L-12** governance security test gap → added H-06 attack-path regression
- **L-13** init timelock floor → production build enforces `>= 86_400s`
- Others (L-01 tenor units, L-04 overdue interest, L-05 LP decimals [done with C-01], L-07 liquidation path, L-09 product-revoke flag) listed as later-version / design decisions

## Early专项 Audit (CONTRACT-AUDIT.md, 2026-08-06)

4 High fund/ledger errors, fixed and guarded by 49→66 integration tests:

- **H1** repayment fee shortfall → LP dividends unfunded (fixed: full fee into pool before allocation)
- **H2** 70% advance stuck in escrow after default (fixed: escrow returned to pool on default)
- **H3** unreleased collateral not deducted → phantom assets (fixed: `total_assets -= down_payment` on release)
- **H4** 100% not-yet-funded counted as total assets on create (fixed: only `down_payment` added; `fund_deal` internal transfer)

---

*Post Medium/Low remediation tests: Rust 22/22, Anchor 66/66, backend Jest 155/155, frontend Vitest 48/48.*
