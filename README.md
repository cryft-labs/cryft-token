# Cryft Token (V3) — `CryftV3` (Upgradeable ERC-20 / BEP-20)

**Contract:** `CryftV3`  
**Name / Symbol:** Cryft / CRYFT  
**Compiler:** Solidity `^0.8.2`  
**License:** MIT  
**Pattern:** Upgradeable (OpenZeppelin `Initializable`) — deploy behind a proxy and call `initialize()` once.

> Note: The ASCII header in your source says “AUDITED”. Treat that as a claim until you verify the audit report and the exact deployed bytecode yourself.

---

## What this token is

CryftV3 is a fee-on-transfer token with:

- **Reflections** (redistribution using reflected accounting)
- **Liquidity + buyback + vault + gasstation fee buckets** (tracked on the contract and used by swap logic)
- **Burns** (sent to the dead address)
- **Anti-LP-sniper / trading gate** (blocks early buys before trading opens)
- **Owner-managed configuration** (fees, wallets, tx limits, LP pool list)
- **Gas reward distribution** (the token contract can distribute native gas currency via `distribute()`)

The fee logic depends on whether a transfer is a **buy**, **sell**, or normal **transfer**, using LP pool detection.

---

## Token parameters (defaults from the provided code)

- **Total supply:** `2,800,000,000 CRYFT` (2.8B)  
  `_tTotal = 2800 * 10**6 * 10**18`
- **Decimals:** `18`
- **Max tx amount:** `3,000,000 CRYFT`  
  `_maxTxAmount = 3 * 10**6 * 10**18`
- **Swap threshold (minTokenSpendAmount):** `500,000 CRYFT`  
  `minTokenSpendAmount = 500 * 10**3 * 10**18`
- **Trading gate:** starts closed; owner must call `openTrading()`
- **Anti-sniper:** enabled by default (`antiSniperEnabled = true`)

---

## Fee system

The contract maintains three fee configs:

- `buyFees`
- `sellFees`
- `transferFees`

### Default buy/sell fees (divisor = 1000)

The code sets buy/sell fees to:

- reflection: `5`
- liquidity: `5`
- gasstation: `5`
- vault: `3`
- burn: `2`
- buyback: `10`
- divisor: `1000`

That totals **30 / 1000 = 3.0%** on buys and sells by default.

Transfers default to **0** across the board (divisor `0` in the snippet you provided), meaning “no transfer fee” unless updated later.

> Important: The effective fee depends on LP pool classification and whether either party is excluded from fees.

---

## How reflections work (high level)

CryftV3 uses a reflection model:

- `_rOwned` stores reflected balances
- `_tOwned` stores “real” balances for accounts excluded from rewards
- `_isExcludedFromReward[account]` decides which balance path applies

`balanceOf()` returns:
- `_tOwned[account]` if excluded from reward
- otherwise converts reflected to token amount via `tokenFromReflection(_rOwned[account])`

---

## Liquidity / buyback / vault / gasstation buckets

Fees are accumulated into `tokenTracker`:

- `tokenTracker.liquidity`
- `tokenTracker.buyback`
- `tokenTracker.gasstation`
- `tokenTracker.vault`

Then the contract can execute swaps (via inherited swap support utilities). The internal `selectSwapEvent()` prioritizes actions in this order:

1. **buyback-and-liquify** path (if enabled & balance threshold met)
2. swap **buyback** tokens for currency
3. swap **liquidity** tokens for currency
4. swap **gasstation** tokens for currency and send to `gasstationWallet`
5. swap **vault** tokens for currency and send to `vaultWallet`

Exact swap/router mechanics live in `LPSwapSupportUpgradeable.sol` and related utilities.

---

## Burn behavior

Burn fee is sent to the dead address and emits:

- `Burn(deadAddress, tBurn)`
- `Transfer(address(this), deadAddress, tBurn)`

---

## Anti-sniper + trading gate behavior

During `_transfer()`:

- If the transfer is a **buy** (LP pool is `from`) and `tradingOpen == false` and `antiSniperEnabled == true`,
  the recipient can be “hammered” (`banHammer(to)`), and the tokens are redirected to the token contract.

Otherwise, if not excluded, trading must be open:
- `require(tradingOpen, "Trading not open");`

---

## Max transaction limit

If both sides are not excluded from tx limit:
- `amount <= _maxTxAmount` is enforced

Owner can update with:
- `updateMaxTxSize(uint256 maxTransactionAllowed)` (value is multiplied by `10**decimals` internally)

---

## Batch airdrops

`batchAirdrop(address[] airdropAddresses, uint256[] airdropAmounts)`:
- callable by `owner()` OR any account excluded from fees
- amounts are interpreted in whole tokens and scaled by `10**decimals` inside `_batchAirdrop`

---

## Gas reward distribution (native currency)

The contract includes:

- `event GasRewardDistributed(address recipient, uint256 amount);`
- `function distribute(address[] recipients, uint256[] amounts) external nonReentrant`

Rules:
- callable only by `gasStation` OR `owner()`
- uses native currency transfers:
  - `payable(recipients[i]).transfer(amounts[i]);`
- tracks:
  - `gasRewardsBalance` (defined in inherited utilities)
  - `totalGasDistributed` (stored in this contract)

Also includes:
- `receive() external payable {}` so it can accept native currency.

---

## Key addresses / roles

- **Owner** (Upgradeable Ownable via inherited logic): controls configuration & trading state
- **Router / Pair**: managed via `updateRouterAndPair(_routerAddress)`
- **gasstationWallet**: receives swapped-out gasstation fee proceeds
- **vaultWallet**: receives swapped-out vault fee proceeds
- **gasStation**: address allowed to call `distribute()` (in addition to owner)
- **buybackEscrowAddress / deadAddress / LP pool list**: referenced in inherited utilities

---

## Owner / admin functions (from provided contract)

Fee & limits:
- `updateBuyFees(...)`
- `updateSellFees(...)`
- `updateTransferFees(...)`
- `updateMaxTxSize(uint256 maxTransactionAllowed)`

Wallets / addresses:
- `updategasstationWallet(address)`
- `updatevaultWallet(address)`
- `updategasStation(address newgasStation)`

Exclusions:
- `excludeFromFee(address account, bool exclude)`
- `excludeFromReward(address account, bool shouldExclude)`
- `excludeFromMaxTxLimit(address account, bool exclude)`

Trading:
- `openTrading()` (enables trading + swaps)
- `pauseTrading()` (toggles trading)

LP pool classification:
- `updateLPPoolList(address newAddress, bool isPoolAddress)`

Swaps:
- `pushSwap()` (manual trigger when allowed)

Recovering unrelated tokens:
- `freeStuckTokens(address tokenAddress)`  
  (cannot withdraw this token itself)

---

## Repository / dependency notes

This contract imports custom upgradeable utilities:

- `./utils/LPSwapSupportUpgradeable.sol`
- `./utils/AntiLPSniperUpgradeable.sol`

and OpenZeppelin upgradeable packages, including:

- `@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol`
- `@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol`
- upgradeable ERC20 interfaces

Make sure your repo includes the `utils/` contracts and that they compile under the same Solidity version range.

---

## Deployment checklist (practical)

1. **Deploy router + LP environment** (on your target chain)
2. **Deploy proxy + implementation**
   - Implementation: `CryftV3`
   - Proxy: your chosen proxy type (Transparent is common)
3. **Call `initialize(router, tokenOwner, gasstationWallet, vaultWallet)`**
4. **Verify exclusions** are correct (owner, tokenOwner, this contract, wallets, escrow)
5. **Confirm LP pool address classification** (`updateLPPoolList(pair, true)`) if not done in utilities
6. **Fund contract with native currency** if you intend to use gas rewards distribution
7. **Open trading**
   - call `openTrading()` when ready
8. **Post-launch monitoring**
   - watch swap events, fee bucket growth, blacklist/anti-sniper status, maxTx enforcement

---

## Operational gotchas (read before you ship)

- **Upgradeable storage layout:** any upgrades must preserve storage ordering across `CryftV3` and all inherited upgradeable contracts.
- **Transfer fee divisor = 0:** your snippet sets transfer divisor to `0`. If your fee math divides by divisor without guarding, updating transfer fees must be done carefully. (Verify the fee calculation path in `takeFees()` and any inherited logic.)
- **Anti-sniper behavior can surprise users:** before trading opens, buys from LP can get redirected and/or blacklisted. Plan your launch timing and messaging accordingly.
- **Native currency `transfer()` limits:** Solidity `transfer()` forwards 2300 gas and can fail for some smart wallets/contracts. If recipients include contracts, consider whether this is acceptable for your use case.
- **Swap behavior is in utilities:** the real “tokenomics engine” lives in `LPSwapSupportUpgradeable`. Read and document that file too (router calls, slippage handling, liquidity add behavior, buyback logic, etc.).

---

## Security notes

This README is a summary of the behavior visible in the `CryftV3` file you provided. The true safety profile depends heavily on:

- proxy admin controls (who can upgrade)
- inherited utility contracts (`LPSwapSupportUpgradeable`, `AntiLPSniperUpgradeable`)
- router interactions and swap settings
- blacklist / sniper logic and how it can be abused
- access control correctness for wallet updates and swap triggers

If you have an audit report, link it here and match:
- chain + contract address
- commit hash / bytecode hash
- proxy implementation address at the audited time

---

## License

MIT — see SPDX header in source.

---

## Maintainer

Deployed by: **CryftCreator**  
Contract: **CryftV3 (Version 3.x)**

