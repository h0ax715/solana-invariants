# Solana DeFi Invariants

Formal specification of protocol-level invariants for Solana DeFi security analysis.

Every DeFi protocol can be reduced to a set of mathematical properties that must hold invariant across all state transitions. When an invariant is violated, a vulnerability exists. This repository documents the invariants I use when auditing Solana lending protocols, AMMs, vaults, and oracle integrations.

---

## Invariants by Protocol Type

### Lending (Compound-style)

| ID | Invariant | Description |
|----|-----------|-------------|
| I1 | Pool Solvency | `vault_balance + total_borrows - uncollected_fees ≥ 0` |
| I2 | No Mint Inflation | New shares must not inflate total value beyond deposited tokens |
| I3 | Deposit No Instant Profit | `tokens_deposited ≥ shares_minted × rate_before / PRECISION` |
| I4 | Withdraw No Overdraw | `shares_burned × rate_before / PRECISION ≤ vault_balance` |
| I5 | Interest Bounded | Accrued interest ≤ `principal × rate × time / YEAR` |
| I6 | No First-Depositor Dilution | First deposit must not enable donation-based share manipulation |

### Lending (Share-Value / Aave V3)

| ID | Invariant | Description |
|----|-----------|-------------|
| SI1 | Accounting Identity | `liquidity_vault + liabilities ≈ assets + fees` |
| SI2 | Deposit No Instant Profit | Shares received reflect current `asset_share_value` |
| SI3 | Withdraw No Overdraw | Redemption ≤ `shares × asset_share_value / PRECISION` |
| SI4 | Share Value Monotonic | `asset_share_value` and `liability_share_value` never decrease |
| SI5 | Interest Bounded | Accrued ≤ `principal × rate × time` |
| SI6 | No First-Depositor Dilution | Donation attack must not create disproportionate vault/shares ratio |

### AMM (Constant-Product)

| ID | Invariant | Description |
|----|-----------|-------------|
| K1 | Constant Product (swaps) | `x × y ≥ k_before` during swaps only |
| K2 | LP Share Value | LP shares redeemable for ≥ proportional reserves |
| K3 | Swap No Free Tokens | `output_amount > 0` for any nonzero input |
| K4 | Add Liquidity Proportional | Shares minted = `min(dx/x, dy/y) × total_shares` |
| K5 | Remove Liquidity Proportional | Tokens returned proportional to share fraction |

### Vault (ERC-4626)

| ID | Invariant | Description |
|----|-----------|-------------|
| V1 | Vault Solvency | `total_shares > 0 → total_assets > 0` |
| V2 | No Free Tokens | Deposit ratio ≥ withdraw ratio |
| V3 | Accounting Equation | `shares × rate ≈ total_assets` |
| V4 | Fees Bounded | `collected_fees ≤ accrued_yield` |

### Liquidation

| ID | Invariant | Description |
|----|-----------|-------------|
| LIQ-1 | Solvency Decrease ≤ Seized | Pool solvency loss bounded by collateral seized |
| LIQ-2 | No Self-Liquidation Profit | Liquidator profit ≤ legitimate bonus |
| LIQ-3 | Bonus Bounded | Bonus ≤ `(1 - threshold) × MAX_BPS` |
| LIQ-4 | Healthy Position Immune | `health_factor ≥ threshold → no liquidation` |
| LIQ-5 | No Dust Griefing | Debt covered ≥ minimum liquidatable amount |
| LIQ-6 | No Cascading Profit | One liquidation must not create new liquidation opportunities |
| LIQ-7 | Bonus Stability | Bonus independent of liquidation split order |
| LIQ-8 | Dust Immunity | Liquidated position leaves no residual dust |
| LIQ-9 | Oracle Sandwich | Profit from oracle manipulation ≤ cost of manipulation |
| LIQ-10 | Gas Griefing | Liquidation bonus > transaction cost |

### Oracle (TWAP / Pyth / Switchboard)

| ID | Invariant | Description |
|----|-----------|-------------|
| ORA-1 | Spot/TWAP Deviation → Breaker | Excessive deviation triggers circuit breaker |
| ORA-2 | TWAP Convex Interval | TWAP ∈ `[min(old_TWAP, midpoint), max(old_TWAP, midpoint)]` |
| ORA-3 | Stale Price → Revert | Operations using stale price must revert |
| ORA-4 | Circuit Breaker → Freeze | No operations allowed during circuit breaker |
| ORA-5 | Manipulation Cost > Profit | Oracle manipulation must be uneconomical |

### Cross-Protocol

| ID | Invariant | Description |
|----|-----------|-------------|
| CROSS-1 | Oracle Manipulation Cost | Cost to manipulate oracle via AMM > profit from manipulated borrow |
| CROSS-2 | Liquidation Price Manipulation | Oracle price drop must not enable profitable self-liquidation |

---

## CPI-Specific Checks (Solana/Anchor)

| ID | Check | Method |
|----|-------|--------|
| CPI-1 | State-modifying instructions require signer on authority | IDL analysis |
| CPI-2 | PDA seeds verified with canonical bump | Seed extraction + on-chain verification |
| CPI-3 | `close = payer` → closed account emptied | Before/after state comparison |
| CPI-4 | `init` → account must not exist before | Before-state check |
| CPI-5 | Account type validation (Token ≠ Mint ≠ Program) | On-chain data size check |
| CPI-6 | CPI to non-whitelisted programs | Static CPI call analysis |

---

## Usage

These invariants serve as a checklist during manual code review and as specifications for automated verification.

For any new Solana lending protocol:
1. Identify the accounting model (Compound cash-based vs share-value accumulator)
2. Map the protocol's state variables to the relevant invariants
3. Verify each invariant against the source code
4. Test edge cases: empty pool, full withdrawal, maximum utilization, oracle staleness

---

## References

- [Compound Protocol Whitepaper](https://compound.finance/documents/Compound.Whitepaper.pdf)
- [Uniswap V2 Core](https://uniswap.org/whitepaper.pdf)
- [Aave V3 Technical Paper](https://github.com/aave/aave-v3-core/blob/master/techpaper/Aave_V3_Technical_Paper.pdf)
- [EIP-4626: Tokenized Vaults](https://eips.ethereum.org/EIPS/eip-4626)
- [Pyth Network Documentation](https://docs.pyth.network/)
- [Anchor Framework](https://www.anchor-lang.com/)
