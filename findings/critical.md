# C-1 — Init Not Bound to ProgramData; Forgeable Upgrade Authority Enables Front-run Takeover (Critical)

**Location:** `initialize_pool` / `initialize_registry` (both trade-finance and supply-chain programs)

**Severity:** Critical (system takeover)

## Description

The two programs' init instructions created the `PoolState` / `Registry` accounts **without binding the passed `program_data` account to the program's own ProgramData PDA**. An attacker could craft a front-running init transaction passing a `program_data` belonging to an attacker-controlled program, causing the contract to record an attacker address as the `upgrade authority`.

Once `upgrade authority` is replaced by an attacker:

- The attacker can upgrade (replace) the contract program logic and implant malicious instructions;
- Or, via the admin-privilege path, take over the pool and registry and drain all escrow / pool funds.

This is a classic **upgrade-authority takeover + init front-running** combination — rated Critical.

## Impact

- Both pool admin and program upgrade authority fall under attacker control;
- All managed assets (USDC / LP / trade-order escrow) can be extracted or the contract replaced with malicious logic;
- "Full system takeover" class.

## Fix

Before init, enforce:

```
program_data.key() == find_program_address(["program", <this program ID>], BPF_LOADER_UPGRADEABLE)
```

- Only accept the program's own ProgramData PDA; reject any external / forged `program_data`;
- Added a Rust unit test anchoring the PDA, asserting rejection of a wrong `program_data`;
- Paired with **H-1**: `initialize_pool` / `initialize_registry` also bind the deployer whitelist or program upgrade authority, preventing front-running and forgery.

## Status

✅ **Fixed and re-tested** — Anchor integration tests 66/66 pass (including init-guard regression).

## Source

`AUDIT-REMEDIATION.md` §〇 C-1; `AUDIT-KNOWN-RISKS.md` C-01 (off-chain LP mint authority, fixed same round).

---

> Related **H-1 (High)**: Init front-running — before the fix, `initialize_pool` / `initialize_registry` did not bind the deployer or upgrade authority, allowing an attacker to initialize first and seize admin. Fix: the initializer must match the deployer whitelist or the program upgrade authority. Together with C-1 this forms defense-in-depth at the init stage.
