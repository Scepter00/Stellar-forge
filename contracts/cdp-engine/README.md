# CDP Engine — Collateralised Debt Positions on Soroban

The CDP Engine lets a user lock collateral in a per-address *vault* and mint a
synthetic debt token against it, as long as the vault stays above the configured
minimum collateralisation ratio. If the oracle price falls and a vault drops
below the lower liquidation threshold, any keeper can repay part of its debt and
take discounted collateral in return; shortfalls that cannot be covered are
booked to a bad-debt reserve instead of being left in the vault.

## How it works

Each vault stores exactly two numbers, both in Stellar 7-decimal token units
(`1` whole token = `10_000_000`):

| Field | Meaning |
| --- | --- |
| `collateral` | Raw units of the collateral token (e.g. wXLM) locked in the vault |
| `debt` | Raw units of the synthetic debt token (e.g. sUSD) issued against it |

The collateralisation ratio (CR) is derived from those two numbers and the
oracle price:

```text
CR = (collateral * oracle_price * PRECISION) / (debt * PRICE_PRECISION)
```

`PRECISION` is `1e18`, so a CR of `1e18` is 100%, and `PRICE_PRECISION` is `1e9`
(the oracle quotes `1e9` = $1.00). Thresholds are configured in basis points:
`15_000` bps = 150%, `12_000` bps = 120%. A vault with no debt reports
`i128::MAX` (infinitely safe). `initialize` rejects a liquidation ratio that is
not strictly below the minimum CR, and `borrow`, `withdraw_collateral` and
`liquidate` all re-check the ratio on chain.

When a keeper liquidates, the debt they cover is burned and they receive
`debt_covered * (10_000 + liq_discount_bps) / 10_000` collateral units. If the
vault cannot cover that seizure, the keeper takes all remaining collateral, the
uncovered debt is added to the `ReserveBadDebt` counter, and the vault is zeroed
out.

## Public functions

Contract entry points (callable on chain through `CdpEngineClient`):

| Function | What it does |
| --- | --- |
| `initialize` | Sets the admin, oracle, collateral and debt token addresses, minimum CR, liquidation ratio and liquidation discount once; errors if already initialised or if the parameters are inconsistent. |
| `deposit_collateral` | Pulls `amount` of the collateral token from `user` into the contract and adds it to the user's vault; no ratio check is needed because the CR can only rise. |
| `withdraw_collateral` | Sends `amount` of collateral back to `user`, failing with `BelowMinCollateralRatio` if the resulting vault with outstanding debt would fall below the minimum CR. |
| `borrow` | Mints `amount` of the debt token to `user`, adding it to the vault's debt and to the global total, and only if the resulting CR is at least the minimum CR. |
| `repay` | Burns `amount` of the debt token from `user` (clamped to the outstanding debt, so overpaying is safe), reducing the vault's debt and the global total. |
| `liquidate` | Lets any keeper cover up to `max_debt_to_cover` of an under-threshold vault's debt, receiving discounted collateral, and returns `(debt_covered, collateral_seized)`; records any shortfall as bad debt. |
| `get_vault` | Returns the vault's `VaultState` (`collateral` and `debt`), defaulting to zeroes if the user has no vault. |
| `get_collateral_ratio` | Returns the vault's current CR scaled by `PRECISION` (`1e18` = 100%), or `i128::MAX` when there is no debt. |
| `is_liquidatable` | Returns `true` if the vault has debt and its CR is below the configured liquidation ratio. |
| `get_total_debt` | Returns the total outstanding debt tracked across all vaults. |
| `get_reserve_bad_debt` | Returns the cumulative bad debt absorbed by the protocol reserve. |
| `set_min_cr` | Admin only: updates the minimum collateralisation ratio in basis points, provided it stays above the liquidation ratio. |
| `set_oracle` | Admin only: points the contract at a new oracle address. |

Public helper functions (usable from Rust, not separate contract entry points):

| Function | What it does |
| --- | --- |
| `get_oracle_price` | Calls `get_price` on the configured oracle and returns the price, rejecting zero or negative values with `InvalidOraclePrice`. |
| `compute_cr` | Computes a CR from explicit `collateral`, `debt` and `price` arguments using checked `u128` arithmetic, returning `i128::MAX` for zero debt. |
| `bps_to_precision` | Converts a basis-point threshold to a `PRECISION`-scaled ratio, e.g. `15_000` bps → `1_500_000_000_000_000_000`. |

## Errors

Failures are reported through the `CdpError` enum: `AlreadyInitialized`,
`NotInitialized`, `Unauthorized`, `InvalidAmount`, `InsufficientCollateral`,
`BelowMinCollateralRatio`, `VaultNotLiquidatable`, `InvalidOraclePrice`,
`Overflow`, `InvalidParameter` and `InsufficientDebt`.

## Build and test

The crate is a standalone Soroban workspace and is independent of the root
Stellar-forge workspace. Run these commands from the repository root with Rust
and Cargo installed:

```sh
# Unit tests for CR arithmetic, vault lifecycle, liquidation and admin guards
cargo test --manifest-path contracts/cdp-engine/Cargo.toml

# Build the deployable Wasm artifact
rustup target add wasm32v1-none
cargo build --manifest-path contracts/cdp-engine/Cargo.toml --release --target wasm32v1-none
```

If you have the Stellar CLI installed, `stellar contract build` run inside
`contracts/cdp-engine` produces the same Wasm artifact.
