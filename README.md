<div align="center">

# Blockchain Supply-Chain Finance

## Independent Security Audit Report

**Target** · Solana / Anchor 0.31.1 · Trade Finance & Supply Chain Contracts

![Solana](https://img.shields.io/badge/Solana-purple?logo=solana&logoColor=white)
![Anchor](https://img.shields.io/badge/Anchor-0.31.1-512BD4?logo=solana&logoColor=white)
![Status](https://img.shields.io/badge/Status-Fixed%20%26%20Re--tested-2ea44f)
![Severity](https://img.shields.io/badge/Finding-1%20Critical%20%2B%206%20High-dc3545)

![GitHub last commit](https://img.shields.io/github/last-commit/sudorm-rf0/supply-chain-finance-audit-report)

</div>

---

> Independent Security Audit Report
> Target: Blockchain Supply-Chain Finance (Solana / Anchor 0.31.1)
> Nature: Independent security audit (not a licensed third-party deliverable; methodology follows Trail of Bits / CertiK)
> Report version: v1.0 — re-test baseline 2026-08-08
> Auditor: sudorm-rf0

---

## 0. Disclaimer & Responsible Disclosure

- This report is an independent assessment by the auditor. It is **not a deliverable from a licensed third-party audit firm** and does not constitute a warranty of the project's security, compliance, or commercial viability.
- All listed **Critical / High vulnerabilities have been fixed at the code level and re-tested** before publication. Exploit descriptions are kept conceptual; **no directly reproducible attack scripts are provided.**
- Program IDs and deployment parameters are **redacted / parameterized** in this public version. The full audit package (contract source, Anchor tests, deployment info without private keys) is shared with the project owner and regulators under responsible-disclosure process.
- The project is currently in a **pre-mainnet** stage. Any modification to core contract logic (business constants, account layout, instruction signatures) should trigger a re-audit.

---

## 1. Executive Summary

An independent security audit was performed on the Solana / Anchor smart contracts of Blockchain Supply-Chain Finance — a system covering the full trade lifecycle (create–fund–logistics–repay), LP pool deposit/redeem, on-chain document notarization, and supplier authorization, intended to carry real-money transactions.

**Core conclusions:**

- Across multiple independent audit rounds (internal assessment + third-party DFR four-round re-test + independent re-test on 2026-08-08), **1 Critical and 6 High** severity vulnerabilities were identified and fixed.
- **All Critical / High issues are fixed at the code level and re-tested.** There are currently **no unfixed Critical / High** issues.
- Remaining items are **pre-launch governance / deployment decisions** (multisig admin, ≥48h admin timelock, fee/allocation/first-loss parameter sign-off, final licensed third-party report) — **not unfixed vulnerabilities.**
- After the §8 governance gates are met, the contract layer is fit for mainnet deployment.

> ⚠️ Important correction: an early internal self-assessment (`AUDIT-REPORT.md`) gave an optimistic **B+ (no Critical/High)** conclusion due to limited scope. The third-party audit **DFR-SC-2026-0143 issued a negative opinion (1 Critical + 4 High)** and noted that the internal assessment under-rated `C-01` (LP mint authority) and `H-03` (admin centralization) — recorded as DFR finding `I-04`. This report reflects the true post-remediation state.

---

## 2. Scope

| Component | Path | Lang | Size |
|-----------|------|------|------|
| trade-finance contract | `programs/trade-finance/src/` | Rust (Anchor 0.31.1) | ~1345 LoC |
| supply-chain contract | `programs/supply-chain/src/` | Rust (Anchor 0.31.1) | ~337 LoC |
| Contract integration tests | `programs/tests/` | TypeScript (Anchor TS) | 46 → 66 cases |
| Backend / microservices | `packages/backend/` | TypeScript (NestJS 11) | sampled |
| Frontend | `packages/frontend/` | TypeScript (Next.js 15) | sampled |
| Infra | `docker-compose.yml` / `k8s/` / `infra/` | YAML/Docker | sampled |

**High-value paths:** LP mint / redeem, fee allocation & dividends, default liquidation insurance path, admin privilege & timelock, init guards, off-chain secrets & network policy.

---

## 3. Methodology

1. **Static analysis** — line-by-line review of all contracts; focus on arithmetic safety (`checked_*`), access control, state-machine integrity.
2. **Dynamic verification** — Rust unit tests + Anchor TS integration tests (boundaries, pause, admin rotation, audit-fix regressions).
3. **Ledger invariant reasoning** — maintain core invariant `INV-1: total_assets = vault + escrow + active_capital`; assert delta on every fund path.
4. **Architecture / threat modeling** — attack-vector lists for default, repay, redeem, init, admin rotation.
5. **Off-chain config review** — JWT/Webhook secrets, NetworkPolicy, webhook replay protection, TOTP 2FA.

---

## 4. Severity Model
## 4. Severity Model

| Level | Definition |
|-------|------------|
| **Critical** | Directly causes permanent fund loss, unbounded mint, or full system takeover |
| **High** | May cause fund loss or system failure under specific conditions |
| **Medium** | Affects security assumptions or data consistency, no direct fund loss |
| **Low** | Code quality / edge-case incompleteness |
| **Informational** | Optimization suggestion, not a threat |

---

## 5. Findings Summary

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| **C-1** | Init not bound to ProgramData → forge upgrade authority, front-run takeover | Critical | ✅ Fixed & re-tested |
| **H-1** | Init front-running (no deployer / upgrade-authority binding) | High | ✅ Fixed & re-tested |
| **H-2** | `test-deployer` feature backdoor (hardcoded DEPLOYER init backdoor) | High | ✅ Eradicated |
| **H-3** | `deposit_pool` real-time balance pricing → donation-pump draining | High | ✅ Fixed & re-tested |
| **H-4** | Fee not parameterized (first-loss layer / allocation hardcoded) | High | ⚠️ Contract layer done; params pending business sign-off |
| **H-5** | `set_admin_delay` timelock self-abolition backdoor | High | ✅ Fixed (hard floor 86400s) |
| **H-06** | supply-chain admin timelock zeroable (same class as H-5) | High | ✅ Fixed (hard floor 86400s) |
| M-01~M-12 | Invariant / insurance / NAV / redeem window / concentration | Medium | ✅ Fixed |
| N-01~N-06 | NAV basis / timelock param / multisig / seed collision / DEPLOYER / first-loss | Medium/Low | ✅ Fixed |
| L-01~L-13 | Units / events / idempotency / pause guard / terminal state | Low | ✅ Fixed |

> An earlier专项 audit (`CONTRACT-AUDIT.md`, 2026-08-06) identified 4 additional High fund/ledger errors (H1 fee shortfall, H2 stuck advance, H3 unreleased collateral, H4 phantom assets) — all fixed and guarded by 49→66 integration tests.

---

## 6. Detailed Findings

- [Critical — C-1 init takeover](findings/critical.md)
- [High — H-1 / H-2 / H-3 / H-4 / H-5 / H-06](findings/high.md)
- [Medium / Low summary](findings/medium-low.md)

---

## 7. Remediation Timeline

| Date | Milestone | Result |
|------|-----------|--------|
| 2026-08-06 | Internal专项 audit (CONTRACT-AUDIT) | Fixed H1–H4 fund/ledger errors |
| 2026-08-07 | Third-party DFR round 1 (SC-2026-0143) | Negative opinion (1C+4H); under-rating flagged |
| 2026-08-07 | DFR 4-round re-test (0143→0144→0145→0146) | All C/H fixed + re-tested |
| 2026-08-08 | Independent re-test (SECURITY-AUDIT-REPORT) | Fixed C-1/H-2/H-3 etc.; Anchor 66/66 pass |

**Current state: no unfixed Critical / High.**

---

## 8. Pre-launch Gates (not vulnerabilities)

Decisions required before mainnet ramp-up:

1. **Multisig admin** — `pool.admin` / `registry.admin` point to a Squads multisig PDA (≥3/5), enforced by `precheck-mainnet-deploy.sh` and MAINNET-MIGRATION.md.
2. **Admin timelock ≥ 48h** — `ADMIN_TRANSFER_DELAY_SECS` / `MIN_ADMIN_DELAY_SECS` set to ≥ 86400s at launch (hard floor already in code).
3. **Fee / allocation / first-loss params** — target default rate, first-loss funding size, final fee values signed off by business & risk (H-4 contract layer already parameterized).
4. **Final licensed third-party report** — issued by a licensed firm as the hard launch gate.

---

## 9. Verdict

> **Conditional Go**: after all §8 governance gates are met, the contract layer is fit for mainnet deployment. Recommend the licensed third-party final report as the hard launch gate, and retain daily on-chain/DB reconciliation (`scripts/reconcile`) and monitoring alerts post-launch.

---

## 10. References

- `AUDIT-REPORT.md` — internal security assessment (B+, noted as outdated)
- `CONTRACT-AUDIT.md` — 2026-08-06 trade-finance专项 audit
- `AUDIT-KNOWN-RISKS.md` — DFR remediation tracking (SC-2026-0143 → 0146)
- `AUDIT-REMEDIATION.md` — 2026-08-08 independent re-test remediation
- `AUDIT-DELIVERY.md` — audit status & package notes

---

*This audit report is based on the code version provided during the audit period. Any core-logic change should trigger a re-audit.*

**中文版 (Chinese):** see [`zh/`](zh/).
