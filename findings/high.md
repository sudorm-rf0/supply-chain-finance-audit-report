# High Severity Findings (H-1 ~ H-06)

> All High issues are fixed at the code level and re-tested. No unfixed High remains.

---

## H-1 — Init Front-running (High)

**Location:** `initialize_pool` / `initialize_registry`
**Description:** The init instruction did not bind the deployer or program upgrade authority, allowing an attacker to initialize first and seize admin.
**Fix:** The initializer must match the deployer whitelist or the program upgrade authority (paired with C-1 for defense-in-depth at init).
**Status:** ✅ Fixed & re-tested.

---

## H-2 — `test-deployer` Feature Backdoor (High)

**Location:** Build config (`test-deployer` feature / `DEPLOYER` constant)
**Description:** If the `test-deployer` feature was compiled into the mainnet build, the hardcoded `DEPLOYER` test wallet became an init backdoor — anyone knowing the constant could initialize as "deployer" and take over admin.
**Fix (eradicated):**
- Removed the `test-deployer` feature and the `DEPLOYER` constant;
- Upgrade-authority check unified to "**initializer must equal the program upgrade authority**"; `None` (frozen) UA is always rejected, with no fallback path;
- Tests use `test-build` (only relaxes the `initial_delay` floor, no whitelist).
**Status:** ✅ Eradicated (N-05 / L-13 same-round fix).

---

## H-3 — `deposit_pool` Real-time Balance Pricing → Donation-pump Draining (High)

**Location:** `deposit_pool` (and all 8 fund-path pricing entrypoints)
**Description:** Share pricing used the **real-time vault balance** as the basis. An attacker could donate USDC directly to the vault after the first deposit, inflating `equity_base` / share price so later depositors receive fewer shares at the inflated price — the first depositor + donor can drain later depositors' value.
**Fix:**
- `PoolState` gained an authoritative ledger `tracked_vault`, **updated only by program in/out paths**;
- All 8 fund instructions maintain `tracked_vault`;
- `deposit` / `redeem` entrypoints assert `real-time balance == tracked_vault`, else `VaultMismatch` rejects external donations;
- Added H-3 donation regression test.
**Status:** ✅ Fixed & re-tested.

---

## H-4 — Fee / First-loss Not Parameterized (High; params pending business sign-off)

**Location:** `repay_deal` fee allocation, first-loss layer
**Description:** Fee rate, platform/LP/buyer allocation, and first-loss reserve size were mostly hardcoded business constants, lacking governance adjustability and a first-loss isolation layer — creating systemic-bias / economic-model risk under extreme parameters.
**Fix (contract layer done):**
- `fee_apy_bps` parameterized (default 670 = 6.7% APR);
- allocation ratios governable (`set_fee_params`);
- `first_loss_reserve` layer (`deposit_first_loss` / `withdraw_first_loss`);
- single-fee cap `MAX_SINGLE_FEE_BPS` (5%);
- NAV basis excludes first-loss; `repay_deal` daily-accrual accounting fixed (dust bias).
**Pending (business / governance decision, not a vulnerability):** target default rate (default 2%), first-loss funding size, final fee values — to be signed off by business & risk using real default data.
**Status:** ⚠️ Contract layer done; params pending (see §8 pre-launch gates).

---

## H-5 — `set_admin_delay` Timelock Self-abolition Backdoor (High)

**Location:** `set_admin_delay` (trade-finance admin-rotation timelock)
**Description:** The admin-rotation timelock (delay) could be set to 0, letting an attacker who had taken over admin **immediately** complete the in-timelock transfer, defeating the "two-step rotation + timelock" guard.
**Fix:** `set_admin_delay` enforces a hard floor `MIN_ADMIN_DELAY_SECS = 86400` (1 day), not lowerable; initial timelock injected by `initialize_pool` (48h in production).
**Status:** ✅ Fixed.

---

## H-06 — supply-chain Admin Timelock Zeroable (High, same class as H-5)

**Location:** `set_registry_admin_delay` (supply-chain Registry admin-rotation timelock)
**Description:** Same root cause as H-5 — the supply-chain admin timelock could be zeroed, defeating the Registry admin two-step rotation guard.
**Fix:** `set_registry_admin_delay` enforces hard floor `MIN_REGISTRY_ADMIN_DELAY_SECS = 86400`; initial timelock injected by `initialize_registry` (48h in production).
**Status:** ✅ Fixed.

---

## Sources

- `AUDIT-REMEDIATION.md` §〇 (C-1 / H-2 / H-3 / M-3 / N-01)
- `AUDIT-KNOWN-RISKS.md` (H-01~H-06, N-01~N-06, L-13 remediation tracking)
- `CONTRACT-AUDIT.md` (H1–H4 fund/ledger errors, fixed)
