---
name: xpad-launchpad
description: "Launch, discover, trade, and graduate tokens on xpad.lol's bonding-curve launchpad and presale platform on X Layer (OKX L2). Triggers: 'launch token on xpad', 'buy bonding curve', 'graduate to Uniswap V2', 'check xpad presale', 'contribute to presale on X Layer', 'trigger graduation reward'. Hands off to uniswap-swap-integration for post-graduation swaps."
version: "0.1.0"
author: "xpad.lol"
tags:
  - bonding-curve
  - launchpad
  - presale
  - ico
  - x-layer
  - okx
  - uniswap-v2
  - xpad
  - fair-launch
  - graduation
---

# xpad.lol Launchpad

Permissionless token launches on **X Layer** (chain ID `196`, native gas token **OKB**, RPC `https://rpc.xlayer.tech`, explorer `https://www.oklink.com/x-layer`). Two core systems:

1. **Bonding Curve** — anyone can mint a token that trades on a constant-product virtual-reserve curve. When `actualReserve` reaches **80 OKB**, the curve auto-graduates: liquidity is seeded into canonical **Uniswap Labs V2** and LP tokens are burned. The address that triggers the graduating buy earns **29.4% of a 6 OKB reward pool** (= **1.764 OKB**, ~$150 at OKB ≈ $85).
2. **Presale** — fixed-price sale with soft/hard caps, optional vesting, optional whitelist, optional preallocations. Refunds on failure.

After graduation, swaps route through **Uniswap V2** (where the LP actually lives) by delegating to the `uniswap-swap-integration` companion skill — see [Post-Graduation Trading](#post-graduation-trading).

Reference material (ABIs + addresses) lives under `reference/` alongside this file.

---

## Execution Model (read this first)

**This is a skill-only plugin. There is no `xpad-launchpad` binary in your PATH.** The command headings below (`list-launches`, `check-curve`, `buy`, etc.) are **logical operation names**, not shell commands. **Do not** try `sh: xpad-launchpad list-launches` — that will fail with `command not found`.

Instead, when the user asks you to run one of these operations:

1. Find the matching command section in this file.
2. Read its `**Calls:**` line(s) — that tells you the exact on-chain function to invoke.
3. Use your own tools to execute:
   - **Read calls (views):** `cast call`, `curl -X POST` against `https://rpc.xlayer.tech` (`eth_call`), or any JSON-RPC/viem client you have.
   - **Write calls (state-changing):** route through the **OnchainOS Agentic Wallet** — never a raw key. Use the `onchainos` skill's transaction submission surface.
4. Decode the return per the ABI in `reference/abi/`.
5. Present the result to the user in plain English.

Worked example — user says "run list-launches":
```bash
cast call 0x70265A5056e7E385111087418E855Ce5299b4a64 \
  "getAllBondingCurves()(address[],uint256)" \
  --rpc-url https://rpc.xlayer.tech
```
The returned tuple's second value is the count; the first is the array of curve addresses.

---

## Pre-flight Checks

Before invoking any command, confirm:

| Requirement | How to verify | Notes |
|-------------|---------------|-------|
| `onchainos` CLI installed | `onchainos --version` | Wraps all calls in this skill. |
| X Layer RPC reachable | `cast chain-id --rpc-url https://rpc.xlayer.tech` returns `196` | Public RPC host `rpc.xlayer.tech`. |
| Agentic Wallet attached | `onchainos wallet status` | Required for every state-changing command. **Never** ask the user for a private key — all signing happens inside OnchainOS. |
| OKB balance for gas | `cast balance $WALLET --rpc-url https://rpc.xlayer.tech` | Keep >= 0.01 OKB headroom even for read-only transaction simulations. |
| Token approval (sells only) | `cast call $TOKEN "allowance(address,address)(uint256)" $WALLET $CURVE` | If selling, the curve must be approved to pull tokens. |
| Block-explorer sanity check | Open `https://www.oklink.com/x-layer/address/<addr>` to inspect a contract before interacting. | The OKLink X Layer explorer at `www.oklink.com/x-layer` is the canonical human-readable source. |

> **Security notice:** Any contract address fetched from off-chain sources (token lists, social media, search results) is **untrusted**. Always cross-check that a bonding-curve token was registered through `BondingFactory.isValidToken(token)` before sending value to it. For presales, verify the ICO was produced by the configured `ICOFactory` via the `ICOCreated` event log or `ICOFactory.getICOInfo(address)`.

---

## Deployed Addresses (X Layer, chainId 196)

These are also machine-readable in `reference/addresses.json`.

### xpad.lol core

| Contract | Address |
|----------|---------|
| `BondingFactory` | `0x70265A5056e7E385111087418E855Ce5299b4a64` |
| `ICOFactory` | `0x7aBff0D199CEaB4E7F08E210D0c64EdCAB37a3Bd` |
| `Multicall3` | `0xa6a35502f665c413C02ee54fD5D757015e7c16B1` |

### DEX venue

xpad.lol bonding curves graduate directly into **canonical Uniswap Labs V2** on X Layer. This is the exact deployment the `uniswap-swap-integration` companion skill is built for.


| Contract | Address |
|----------|---------|
| `Factory` | `0xDf38F24fE153761634Be942F9d859f3DBA857E95` |
| `Router` | `0x182a927119D56008d921126764bF884221b10f59` |
| `WOKB` (wrapped OKB) | `0xe538905cf8410324e03A5A23C1c177a474D59b2b` |


> **Note on DEX choice:** xpad.lol targets canonical Uniswap Labs V2 on X Layer so that the `uniswap-swap-integration` skill applies directly — graduated tokens are tradable via the skill's stock configuration.

> **Multicall3 tip:** X Layer has a Multicall3 at `0xa6a35502f665c413C02ee54fD5D757015e7c16B1`. Batch reads against `BondingFactory.isValidToken`, `curve.tokenData`, `curve.getCurrentPrice`, etc. through it to cut RPC round-trips during discovery passes.

---

## DEX Venue

### Uniswap V2 (canonical — graduation target)

- Bonding curve calls the canonical Uniswap V2 router (`0x182a927119D56008d921126764bF884221b10f59`) to create and seed the LP. All xpad.lol bonding-curve graduations land here.
- For off-chain `getAmountsOut` math, use `9970/10000` (30 bps fee). Prefer calling `router.getAmountsOut(...)` on-chain for correctness.
- This is the canonical Uniswap Labs V2 deployment on X Layer — the `uniswap-swap-integration` companion skill drives it directly.
- LP tokens are burned to `0x...dEaD`, so graduation liquidity is permanent.

---

### Smoke-test commands

```bash
# Confirm CREATION_FEE is 0.03 OKB (= 30000000000000000 wei = 3e16)
cast call 0x70265A5056e7E385111087418E855Ce5299b4a64 'CREATION_FEE()(uint256)' \
  --rpc-url https://rpc.xlayer.tech
# Expected: 30000000000000000

# Confirm ICO deploymentFee is 0.03 OKB
cast call 0x7aBff0D199CEaB4E7F08E210D0c64EdCAB37a3Bd 'deploymentFee()(uint256)' \
  --rpc-url https://rpc.xlayer.tech
# Expected: 30000000000000000

# Confirm chain id
cast chain-id --rpc-url https://rpc.xlayer.tech
# Expected: 196
```

---

## Execution Flow for Write Operations

For every state-changing command (anything not in [Discovery](#discovery-read-only)):

1. **Quote** — call the matching preview/view function and show the user the expected outcome (tokens out, OKB out, slippage envelope, fees).
2. **Confirm** — ask the user to approve the exact OKB cost, slippage in BPS, and recipient.
3. **Execute** — submit through the OnchainOS Agentic Wallet. Never request raw keys.
4. **Verify** — re-read the relevant view (`tokenData`, `getState`, `getVestingStatus`) and report the new state plus tx hash. Link the hash on `https://www.oklink.com/x-layer/tx/<hash>`.

---

## Discovery (read-only)

No wallet required. Safe to chain freely.

### list-launches

```bash
xpad-launchpad list-launches [--limit N] [--offset N]
```

**When to use:** Browse all bonding curves xpad.lol has ever created.
**Calls:** `BondingFactory.getAllBondingCurves() -> (address[] curves, uint256 count)`.
**Output:** Array of `{ curveAddress, tokenAddress }` (resolve token via `BondingFactory.curveToToken(curve)`).

```bash
# Example
cast call 0x70265A5056e7E385111087418E855Ce5299b4a64 "getAllBondingCurves()(address[],uint256)" \
  --rpc-url https://rpc.xlayer.tech
```

---

### check-curve

```bash
xpad-launchpad check-curve <tokenAddress>
```

**When to use:** Inspect price, reserves, supply, graduation status, dev lock.
**Calls:**
- `BondingFactory.getBondingCurve(token) -> curve`
- `curve.getCurrentPrice() -> uint256` (18-dec OKB-per-token)
- `curve.tokenData() -> (virtualReserve, actualReserve, availableSupply, lpReserve, devPooledTokens, graduated, devLockUntil, lpPool, devAddress)`
- `curve.isHypedLaunch() -> bool`

**Output fields:**
- `priceOkb` — current marginal price.
- `actualReserve` / `TARGET_RAISE` — graduation progress (`80 OKB` denominator).
- `availableSupply` — tokens still purchasable from curve.
- `graduated` — if `true`, route trades to `lpPool` via `post-graduation-swap` (canonical Uniswap V2).
- `isHypedLaunch` — if `true`, individual buy/sell is capped at `0.5 OKB`.

---

### preview-buy

```bash
xpad-launchpad preview-buy <tokenAddress> <okbAmount>
```

**When to use:** Quote tokens-out before executing `buy`. Caller MUST use this to compute `minTokensOut`.
**Calls:** `curve.previewBuyTokens(uint256 ethIn) -> uint256 tokensOut`.
**Output:** `tokensOut` (raw 18-dec). Reverts with `AlreadyGraduated` if curve has graduated.

---

### preview-sell

```bash
xpad-launchpad preview-sell <tokenAddress> <tokenAmount>
```

**Calls:** `curve.previewSellTokens(uint256 tokenAmount) -> uint256 netEthOut`.

---

### check-graduation

```bash
xpad-launchpad check-graduation <tokenAddress>
```

**When to use:** Decide whether the next buy can trigger graduation (and earn the **1.764 OKB** reward).
**Logic:**
```
remainingReserve = 80 OKB - tokenData.actualReserve
remainingGrossOkb = remainingReserve * 10000 / (10000 - BPS_FEE)   # 1.0101% gross-up for 1% fee
```
**Output:**
- `graduated` (bool), `lpPool` (address — `0x0` until graduated)
- `okbToGraduate` — the **gross** OKB needed; this is the value to send in `buy` to flip the curve.
- After graduation, the curve refunds any excess OKB beyond what's needed to fill exactly to `TARGET_RAISE`, so over-paying is safe.

---

### list-presales

```bash
xpad-launchpad list-presales [--limit N]
```

**When to use:** Discover ICOs created via `ICOFactory`.
**Calls:** Index `ICOCreated(address indexed icoAddress, address indexed creator, address indexed newTokenAddress, uint256 feePaid, ICOConfig config)` events from `ICOFactory`. The factory does not expose an enumerable getter — use logs (`eth_getLogs` on `rpc.xlayer.tech`).
**Output:** Array of `{ icoAddress, creator, tokenAddress, creationTime }`.

---

### check-presale

```bash
xpad-launchpad check-presale <icoAddress>
```

**Calls:**
- `ico.getState() -> State { Pending, Active, Successful, Failed }`
- `ico.softCap()`, `ico.hardCap()`, `ico.totalRaised()`
- `ico.startTime()`, `ico.endTime()`, `ico.tokenPrice()`, `ico.fundraisingCurrency()`
- `ico.tokensAvailableForSale()`, `ico.getRemainingTokens()`
- `ico.minContribution()`, `ico.maxContribution()`, `ico.whitelistActive()`

**Output:** State, raise progress, time-to-end, price (`tokensAvailableForSale * 1e18 / hardCap`), and a flag for native vs ERC20 fundraising (`fundraisingCurrency == address(0)` means native OKB).

---

### vesting-status

```bash
xpad-launchpad vesting-status <icoAddress> <beneficiary>
```

**Calls:** `ico.getVestingStatus(address) -> (totalAllocation, totalVested, totalReleased, claimableNow, vestingEnabledFlag, vestingStartTimestamp, vestingCliffEndTimestamp, vestingEndTimestamp)`.
**Use:** Pair with `ico.getClaimableAmount(beneficiary)` to compute what to pass to `claim-presale`.

---

## Trading (state-changing — Agentic Wallet required)

> Always run a preview first, then ask the user to confirm. Default slippage if the user doesn't specify: **100 BPS (1%)**. Refuse to send a buy/sell with `slippageBps > 500` (5%) without an explicit `--allow-high-slippage` flag — high slippage on a thin curve is a sandwich-attack magnet.

### buy

```bash
xpad-launchpad buy <tokenAddress> <okbAmount> [--slippage-bps 100]
```

**Calls:** `curve.buyTokens(uint256 minTokensOut)` with `msg.value = okbAmount`.
**Pre-call computation:**
```
expectedOut = curve.previewBuyTokens(okbAmount)
minTokensOut = expectedOut * (10000 - slippageBps) / 10000
```
`minTokensOut` MUST be `> 0` or the call reverts with `ZeroValue`. Excess OKB beyond what's needed to reach `TARGET_RAISE` is refunded automatically.

**Notes:**
- Reverts with `SniperBlocked` if invoked in the same block the curve was created.
- Reverts with `TransactionTooLarge` on hyped launches if `msg.value > 0.5 OKB`.
- If the buy crosses `TARGET_RAISE`, the curve will `_graduate()` in the same tx — see `trigger-graduation`.

```bash
# cast equivalent (illustrative — the skill submits via OnchainOS)
cast send $CURVE "buyTokens(uint256)" $MIN_TOKENS_OUT \
  --value ${OKB_AMOUNT}ether --rpc-url https://rpc.xlayer.tech
```

---

### sell

```bash
xpad-launchpad sell <tokenAddress> <tokenAmount> [--slippage-bps 100]
```

**Pre-call:**
1. Ensure ERC20 allowance: `token.approve(curve, tokenAmount)` if `allowance < tokenAmount`.
2. `expectedOkb = curve.previewSellTokens(tokenAmount)`; `minEthOut = expectedOkb * (10000 - slippageBps) / 10000`.

**Calls:** `curve.sellTokens(uint256 tokenAmount, uint256 minEthOut)`.

**Notes:**
- Reverts with `DevTokensLocked` if the seller is the dev address and the lock is still active.
- Reverts with `AlreadyGraduated` once the curve has migrated; route to `post-graduation-swap`.

---

### trigger-graduation

```bash
xpad-launchpad trigger-graduation <tokenAddress> [--max-okb 100]
```

**When to use:** A curve is close to `80 OKB` and the user wants to capture the **1.764 OKB graduator reward** (~$150 at $85/OKB).

**Algorithm:**
1. `(_, actualReserve, _, _, _, graduated, _, _, _) = curve.tokenData()`. Abort if `graduated`.
2. `remainingReserve = 80e18 - actualReserve`.
3. `okbToSend = ceilDiv(remainingReserve * 10000, 10000 - 100)` (ceiling division guarantees crossing the threshold — no extra wei needed).
4. If `okbToSend > maxOkb`, abort and report shortfall.
5. `expectedOut = curve.previewBuyTokens(okbToSend)`; `minTokensOut = expectedOut * 99 / 100` (1% slippage).
6. Submit `buyTokens(minTokensOut)` with `msg.value = okbToSend`.
7. Read `tokenData.lpPool` after the tx to confirm graduation; report reward received.

**Notes:**
- The reward is paid to `msg.sender` of the graduating tx (i.e. the Agentic Wallet — make sure the user understands they receive the OKB, not the recipient of the tokens).
- The buy itself will return tokens worth roughly the OKB sent minus the reward economics; the **net** of `tokens received + 1.764 OKB - okb sent` is the user's gain (often positive even before token price action).

---

### post-graduation-swap

```bash
xpad-launchpad post-graduation-swap <tokenAddress> <okbOrTokenAmount> \
  [--side buy|sell] [--slippage-bps 100]
```

**When to use:** `check-curve` reports `graduated == true`. The bonding curve is closed; trade against the canonical Uniswap V2 pair instead.

**Routing:** Delegates to the `uniswap-swap-integration` companion skill against canonical Uniswap Labs V2 on X Layer. The pair address is `tokenData.lpPool` — cross-verify via `IUniswapV2Factory.getPair(token, WOKB)` on the Uniswap V2 factory (`0xDf38F24fE153761634Be942F9d859f3DBA857E95`).

```bash
uniswap-swap-integration swap \
  --chain x-layer \
  --router  0x182a927119D56008d921126764bF884221b10f59 \
  --factory 0xDf38F24fE153761634Be942F9d859f3DBA857E95 \
  --weth    0xe538905cf8410324e03A5A23C1c177a474D59b2b \
  --fee-bps 30 \
  --token-in  <WOKB or token> \
  --token-out <token or WOKB> \
  --amount-in <amount> \
  --slippage-bps <bps>
```

**LP pool address** is `tokenData.lpPool` (from `check-curve`). Initial LP tokens are burned to `0x...dEaD`, so Uniswap V2 liquidity is permanent.

---

## Launch Creation (state-changing)

### launch-token

```bash
xpad-launchpad launch-token \
  --name "<1-30 alphanumeric+space chars>" \
  --symbol "<1-6 alphanumeric chars>" \
  --description "<<=500 chars>" \
  --total-supply <wei, 100e18 <= x <= 1e33> \
  --socials '<valid JSON string — see warning below>' \
  --image "<bare IPFS CID — e.g. bafy... or Qm...; no ipfs:// prefix, no URL>" \
  --user-salt <bytes32, must be unique per sender> \
  [--dev-purchase-okb 0] \
  [--dev-lock-seconds 0] \
  [--hyped-launch false]
```

**Calls:** `BondingFactory.createBondingSystem(TokenParams)` payable with `msg.value = 0.03 OKB + devPurchaseOkb`.

**`TokenParams` struct** (must be encoded exactly):
```solidity
struct TokenParams {
    string  name;
    string  symbol;
    string  description;
    uint256 totalSupply;
    string  socials;          // MUST be valid JSON — see warning below
    string  image;            // bare IPFS CID (no ipfs:// prefix, no URL); frontends prepend their preferred gateway
    uint256 devLockDuration;  // seconds; 0 = no lock
    uint256 devPurchaseETH;   // OKB units; 0 = no dev buy
    bool    isHypedLaunch;
    bytes32 userSalt;         // unique per sender
}
```

**Returns:** `(address tokenAddress, address bondingCurveAddress)`. Listen for `BondingSystemCreated` event for full metadata.

**Validation rules (from the contract — surface failures clearly):**
- Name: 1–30 chars, `[A-Za-z0-9 ]` only.
- Symbol: 1–6 chars, `[A-Za-z0-9]` only (no space).
- Total supply between `100 ether` and `1_000_000_000_000_000 ether`.
- `userSalt` must not have been used by `msg.sender` before (`InvalidSalt` otherwise) — generate via `keccak256(abi.encode(timestamp, nonce))`.

**CRITICAL — `socials` field must be valid JSON.** The contract concatenates `socials` and `image` into `metadataJSON` emitted in the `BondingSystemCreated` event. If `socials` is an empty string (`""`), the resulting JSON is malformed (`{"socials":,"image":"..."}`) — the token deploys on-chain but the xpad.lol frontend/indexer silently rejects the broken metadata and the token **never appears in the UI**. Always pass one of:
- `"{}"` — safe empty object when the user has no socials
- `'{"twitter":"https://x.com/...","telegram":"https://t.me/..."}'` — full links

Never pass an empty string. Validate that `JSON.parse(socials)` succeeds before submitting.

**Hyped-launch tradeoffs:** Locks every individual buy/sell <= `0.5 OKB` worth. Use only for fair-launch culture; otherwise keep `false`.

---

### launch-presale

```bash
xpad-launchpad launch-presale --config-file ./ico-config.json [--whitelist-file ./addrs.txt]
```

**Calls:** `ICOFactory.createICO(ICOConfig calldata config, address[] calldata whitelistedAddresses)` payable with `msg.value == ICOFactory.deploymentFee()` (`0.03 OKB`).

**`ICOConfig` essentials** (full struct layout in `reference/abi/ICOFactory.json`):
- `tokenName` / `tokenSymbol` — for the brand-new ERC20 minted by the factory.
- `tokenTotalSupply` — `>= 1e18`, `<= 1e15 * 1e18` (1 quadrillion tokens).
- `fundraisingCurrency` — `address(0)` for native OKB, otherwise an ERC20 (validated via `try { totalSupply, decimals, name, symbol, balanceOf }`).
- `softCap <= hardCap` (both in fundraising-currency units).
- `startTime` in `[now, now + 100 days]`; `endTime - startTime <= 90 days`.
- `tokenPreAllocations[]` and `fundingCurrencyAllocations[]` — sums in BPS; pre-alloc + treasury fee (1%, configurable up to 10%) <= 10000 BPS. **Creator may not appear in `tokenPreAllocations`**, treasury addresses may not appear in either.
- Vesting: `vestingTotalDuration <= 3 years`, `vestingTotalDuration >= vestingCliffDuration`, `vestingInitialUnlockBps <= 10000`.

**Two-step activation:** `createICO` leaves the ICO in `Pending`. After `block.timestamp >= startTime`, call:
```bash
xpad-launchpad activate-presale <icoAddress>
```
which invokes `ICOFactory.activateICO(address)` → moves the ICO to `Active`. Anyone can call this, but the factory reverts unless `msg.sender == creator`.

---

## Presale Participation (state-changing)

### contribute

```bash
xpad-launchpad contribute <icoAddress> <amount>
```

**Calls:** `ico.contribute(uint256 _amount, bool _isNative)` —
- Native OKB ICO: `_isNative = true`, send `msg.value = amount`, pass `_amount = 0` (ignored).
- ERC20 ICO: `_isNative = false`, ensure `IERC20.approve(ico, amount)` first, send `msg.value = 0`.

**Pre-checks:** `ico.getState() == Active`; `amount` within `[minContribution, maxContribution]`; if `whitelistActive`, `isWhitelisted[msg.sender] == true`. Also the contract refuses contributions from any address that has a non-zero `tokenAllocationAmounts` entry (preallocated recipients can't also buy in).

---

### claim-presale

```bash
xpad-launchpad claim-presale <icoAddress> [--type contributor|preallocated|creator]
```

**Calls:** `ico.claimTokens(TokenClaimType)` where the enum is `{ CONTRIBUTOR=0, PREALLOCATED=1, CREATOR=2 }`. Requires `State.Successful`.

**Default `--type`:** if not provided, look up the caller's role:
- `contributions[msg.sender] > 0` → `CONTRIBUTOR`
- `tokenAllocationAmounts[msg.sender] > 0` → `PREALLOCATED`
- `msg.sender == creator` → `CREATOR`

For vested allocations, repeat-call as more vests; use `vesting-status` to compute next claimable date.

**Funding-side claim:** Recipients of `fundingCurrencyAllocations` (and the creator) call `ico.claimPreAllocatedFunding()` separately — expose as:
```bash
xpad-launchpad claim-presale-funding <icoAddress>
```

---

### refund-presale

```bash
xpad-launchpad refund-presale <icoAddress>
```

**Calls:** `ico.refund()`. Requires an ICO to be failed. Returns the contributor's full deposit.

---

### finalize-presale

```bash
xpad-launchpad finalize-presale <icoAddress>
```

**Calls:** `ico.finalize()`. Anyone can call after `endTime` OR once `totalRaised >= hardCap`. Transitions `Active -> Successful` (if `totalRaised >= softCap`) or `Active -> Failed`. On success, `vestingStartTime` is set to `block.timestamp` and treasury fees are paid.

---

## Dev Management

### claim-dev-tokens

```bash
xpad-launchpad claim-dev-tokens <tokenAddress>
```

**Calls:** `curve.claimDevTokens()` — only `tokenData.devAddress`, only after `block.timestamp >= devLockUntil`. Works both pre- and post-graduation. Use `curve.getDevLockInfo() -> (bool isLocked, uint256 lockEndTime)` to display the unlock date.

---

## Post-Graduation Trading

When `tokenData.graduated == true`, the bonding curve is permanently closed. **All swaps route through canonical Uniswap V2** — the venue where the graduation LP was seeded:

```yaml
chain: x-layer
chainId: 196
venue:   Uniswap V2 (canonical)
router:  0x182a927119D56008d921126764bF884221b10f59
factory: 0xDf38F24fE153761634Be942F9d859f3DBA857E95
feeBps:  30                 # 0.30% — use 9970/10000 in local math
weth:    0xe538905cf8410324e03A5A23C1c177a474D59b2b   # WOKB
```

The pair address for a graduated token is `tokenData.lpPool` — verify with `IUniswapV2Factory.getPair(token, WOKB)` on the Uniswap V2 factory before trading. LP supply is burned to `0x...dEaD`, so liquidity cannot be rugged.

The `uniswap-swap-integration` companion skill is a first-class integration here: xpad.lol graduations land on the exact DEX deployment the skill is built for, so the skill's stock configuration applies without any fork-specific adjustments. This skill simply delegates the post-graduation trade with the addresses above.

---

## Error Handling

### BondingFactory / BondingCurve

| Error | Cause | Resolution |
|-------|-------|------------|
| `InsufficientFee()` | `msg.value < 0.03 OKB + devPurchaseETH` on `createBondingSystem`. | Top up; remember the dev pre-buy is paid on top of the 0.03 OKB creation fee. |
| `InvalidSupply()` | Total supply outside `[100 ether, 1e15 ether]`. | Re-quote the user; default to `1_000_000_000 ether` (1B tokens) if unspecified. |
| `InvalidNameLength()` / `InvalidSymbolLength()` | Empty or > 30/6 chars. | Ask the user to shorten. |
| `InvalidCharacters()` | Non-alphanumeric (or non-space in name). | Strip emoji/punctuation before retry. |
| `InvalidSalt()` | `userSalt` already used by this sender. | Regenerate `bytes32` salt and resubmit. |
| `SlippageError()` | `tokensOut < minTokensOut` on buy, or `netEthOut < minEthOut` on sell. | Re-quote with `previewBuyTokens` / `previewSellTokens` and rebuild minimums. Increase `--slippage-bps` (cap 500 = 5% without explicit user override). |
| `AlreadyGraduated()` | Curve has migrated to Uniswap V2. | Switch to `post-graduation-swap`. |
| `SniperBlocked()` | Buy submitted in the same block the curve was created. | Wait one block and retry. |
| `TransactionTooLarge()` | `isHypedLaunch == true` and the buy/sell value > `0.5 OKB`. | Split into multiple smaller txs. |
| `DevTokensLocked()` / `TokensLocked()` | Dev wallet trying to sell/claim before `devLockUntil`. | Show `getDevLockInfo()` countdown. |
| `NotDevAddress()` | Caller of `claimDevTokens` is not `tokenData.devAddress`. | Inform the user — only the original deployer can claim. |
| `InsufficientBalance()` | Sell would draw more OKB than `actualReserve`. | Reduce sell size; `previewSellTokens` should normally prevent this. |
| `ETHRefundFailed()` / `TransferFailed()` | Refund or fee transfer reverted (e.g., recipient is a contract that rejects ETH). | Use an EOA. |
| `ZeroValue()` / `ZeroAddress()` | Required arg was 0. | Validate inputs before submit. |
| `NotGraduated()` | `withdrawSkimmedAssets` called before graduation. | Wait until graduation. |
| `CannotWithdrawBondingToken()` | Owner attempted to skim the curve's own token. | Pick a different token. |

### ICOFactory / ICOContract

| Error | Cause | Resolution |
|-------|-------|------------|
| `ICOFactory__InvalidFee()` | `msg.value != deploymentFee`. | Read `ICOFactory.deploymentFee()` and resend. |
| `ICOFactory__InvalidInput("...")` | Schema violation (caps, allocations, currency, vesting). | Surface the `reason` string verbatim and ask the user to fix that field. |
| `ICOFactory__NotPending()` / `ICOFactory__TimeNotReached()` / `ICOFactory__Unauthorized()` | `activateICO` preconditions. | Map each to user messaging: already activated, start time not reached, or non-creator caller. |
| `ICOContract__InvalidState(State)` | Operation requires a different ICO state. | Map to user message: e.g. requires `Active` (call `activate-presale`); requires `Successful` (call `finalize-presale`); requires `Failed` (cannot claim, only refund). |
| `ICOContract__TimeError(reason)` | `startTime` in past, `endTime` not reached, etc. | Surface `reason` and pause until time advances. |
| `ICOContract__ContributionTooSmall` / `ContributionTooLarge` | Below `minContribution` or above `maxContribution`. | Re-quote within bounds. |
| `ICOContract__NotWhitelisted(addr)` | `whitelistActive == true` and caller absent. | Ask creator to whitelist; or pause. |
| `ICOContract__NothingToClaim()` / `NothingToRefund()` | No allocation / no contribution. | Confirm role and state with `vesting-status` / `check-presale`. |
| `ICOContract__OnlyCreator()` / `OnlyFactory()` / `OnlyTreasury()` | Wrong caller. | Refuse and explain. |
| `ICOContract__TreasuryFeesNotPaid()` | Funding-side claim attempted before `finalize`. | Call `finalize-presale` first. |
| `ICOContract__InvalidAmount("Expected ERC20.../native...")` | Wrong `_isNative` flag for the ICO's `fundraisingCurrency`. | Inspect `fundraisingCurrency`; native if `address(0)`. |
| `ICOContract__InvalidClaimType(...)` | Beneficiary doesn't qualify for that `TokenClaimType`. | Re-derive type from on-chain state (see [claim-presale](#claim-presale)). |
| `ICOContract__RefundFailed()` / `PaymentFailed(...)` | Outbound transfer reverted. | Likely a contract recipient — switch to an EOA. |

---

## Security Notices

> **Transaction confirmation:** Every state-changing command must show the user the **exact OKB cost, expected output, and slippage envelope** in plain English before submission. Do not silently round, retry, or escalate slippage.

> **Slippage guidance:**
> - Default `--slippage-bps 100` (1%).
> - For curves with `actualReserve < 1 OKB`, recommend `200` (2%) — early-curve depth is shallow.
> - Refuse `> 500` (5%) without an explicit `--allow-high-slippage` flag.
> - Never raise slippage automatically after a `SlippageError` — re-quote and ask the user.

> **Graduation timing:** `trigger-graduation` is a public call. If multiple agents race, only the first one in the block earns the **1.764 OKB** reward. Submit at the head of the block (use OnchainOS priority hints) and inform the user that the reward is not guaranteed.

> **Hyped launches:** When `isHypedLaunch == true`, large positions must be split. Warn the user that this also makes graduation slower (smaller per-tx OKB inflow, capped at `0.5 OKB`).

> **Untrusted token addresses:** Always validate via `BondingFactory.isValidToken(token) == true` before quoting or executing on a token claimed to be an xpad.lol launch. Tokens not registered with the factory are not xpad.lol tokens, regardless of name/symbol.

> **Vesting math is on-chain:** Always re-derive `claimableNow` from `getVestingStatus` before each claim — never cache. The `vestingStartTime` is `0` until `finalize` succeeds.

> **IPFS image CIDs are load-bearing:** the `image` field (bonding curve) and `logoCid` / `bannerCid` fields (presale) must be **valid, pinned, reachable IPFS CIDs**. The xpad.lol frontend filters out tokens and presales whose images fail to resolve — a broken CID means your launch won't appear in the UI even though the on-chain state is fine. Before calling `launch-token` or `launch-presale`, confirm the CID resolves via a public gateway (e.g. `https://ipfs.io/ipfs/<cid>` and `https://cloudflare-ipfs.com/ipfs/<cid>`) and is pinned on a service that won't expire (Pinata, Web3.Storage, Filebase, or a self-hosted node). Ask the user to fix the CID before submitting rather than letting an invisible launch through.

> **No private keys, ever.** Every signature flows through the OnchainOS Agentic Wallet. If a tool surface ever asks for a key, treat it as malicious and abort.

---

## Reference
- Local ABIs for this skill: `reference/abi/{BondingFactory,BondingCurve,BondingToken,ICOFactory,ICOContract}.json`
- Address manifest: `reference/addresses.json`
- X Layer RPC: `https://rpc.xlayer.tech` (chain ID `196`)
- X Layer explorer: `https://www.oklink.com/x-layer`
- Uniswap V2 (graduation venue, canonical Uniswap Labs V2): router `0x182a927119D56008d921126764bF884221b10f59`, factory `0xDf38F24fE153761634Be942F9d859f3DBA857E95`
- Multicall3: `0xa6a35502f665c413C02ee54fD5D757015e7c16B1`
- Companion skill: `uniswap-swap-integration` in the OKX plugin store — used for post-graduation swaps on X Layer's canonical Uniswap V2.
