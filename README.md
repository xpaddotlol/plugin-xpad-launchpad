# xpad-launchpad

**Launch, trade, and participate in xpad.lol bonding-curve tokens and presales on X Layer (OKX L2) — with automatic Uniswap V2 graduation.**

Built for the **OKX Build X AI Hackathon — Skills Arena** track.

This plugin is a pure skill plugin: documentation + ABIs + address manifest that teach an AI agent running on [OnchainOS](https://onchainos.xyz) how to drive the xpad.lol protocol safely from natural language.

---

## What it does

xpad.lol runs two on-chain systems on X Layer (chain ID `196`, native gas **OKB**):

1. **Bonding-curve launchpad** — permissionless ERC20 launches on a virtual-reserve curve. At **80 OKB** of raised reserve the curve auto-graduates: a canonical Uniswap V2 pair is created, LP burned, and the graduating caller earns **1.764 OKB**.
2. **Presale (ICO) platform** — fixed-price sales with soft/hard caps, optional vesting, optional whitelisting, optional preallocations, refunds on failure.

The skill exposes one clean command surface for all of it.

### Commands at a glance

**Discovery (read-only):**
- `list-launches`, `check-curve`, `preview-buy`, `preview-sell`, `check-graduation`
- `list-presales`, `check-presale`, `vesting-status`

**Trading (Agentic Wallet required):**
- `buy`, `sell`, `trigger-graduation`, `post-graduation-swap`

**Launch creation:**
- `launch-token`, `launch-presale`, `activate-presale`

**Presale participation:**
- `contribute`, `claim-presale`, `claim-presale-funding`, `refund-presale`, `finalize-presale`

**Dev management:**
- `claim-dev-tokens`

See [`SKILL.md`](./SKILL.md) for full specs.

---

## Agentic Wallet

All state-changing operations flow through an **OnchainOS Agentic Wallet**. The wallet used for the hackathon demo is:

```
Agentic Wallet: 0x03d90628cd1ddb700d346542144fd7b119436046
On-chain activity: https://www.oklink.com/x-layer/address/0x03d90628cd1ddb700d346542144fd7b119436046
```

The private key never leaves OnchainOS. This skill will refuse any command that would require a raw key.

---

## Companion agent

A reference AI agent that uses this skill end-to-end — `agent-xpad-launch-hunter` — lives in a sibling repo. It demonstrates the full loop: discover → preview → buy → graduate → post-graduation Uniswap V2 swap.

- Repo: `https://github.com/xpaddotlol/agent-xpad-launch-hunter`

---

## Install

After this PR is merged into the [OKX plugin store](https://github.com/okx/plugin-store):

```bash
npx skills add okx/plugin-store --skill xpad-launchpad
```

Or install locally from a clone of `okx/plugin-store`:

```bash
git clone https://github.com/okx/plugin-store.git
cd plugin-store
./scripts/install-local.sh skills/xpad-launchpad
```

---

## Quick start

A realistic end-to-end flow — discover a curve, preview, buy, then sell on graduation:

```bash
# 1. Browse all launches
xpad-launchpad list-launches --limit 20

# 2. Inspect a specific token
xpad-launchpad check-curve 0xTOKEN_ADDRESS

# 3. Quote before trading
xpad-launchpad preview-buy 0xTOKEN_ADDRESS 1   # 1 OKB in

# 4. Execute with 1% slippage
xpad-launchpad buy 0xTOKEN_ADDRESS 1 --slippage-bps 100

# 5. Later — the curve has graduated, swap on Uniswap V2
xpad-launchpad post-graduation-swap 0xTOKEN_ADDRESS 5 --side sell --slippage-bps 100
```

Step 5 routes through **canonical Uniswap V2** (where the graduation LP lives) by delegating to the `uniswap-swap-integration` companion skill.

---

## Integration with OnchainOS

Every state-changing command is brokered through the OnchainOS Agentic Wallet (`onchainos wallet ...`). The agent never sees a private key; it receives a signing-capable session handle. The skill:

- Quotes expected output via `previewBuyTokens` / `previewSellTokens` / `getClaimableAmount` before submit.
- Computes `minTokensOut` / `minEthOut` from the user-approved slippage.
- Submits the tx via OnchainOS.
- Re-reads state (`tokenData`, `getState`, `getVestingStatus`) after confirmation and reports the tx hash linked to OKLink.

---

## Integration with Uniswap AI Skills

When a bonding curve graduates, the skill delegates post-graduation trades to the **`uniswap-swap-integration`** companion skill. xpad.lol's `BondingFactory` seeds graduation liquidity directly into **canonical Uniswap Labs V2** on X Layer — the same deployment the Uniswap skill already knows how to drive. No fork-specific init-code-hash or fee-BPS adjustments are required; the skill's stock configuration applies as-is.

### Uniswap V2 on X Layer (graduation venue)

| Param | Value |
|-------|-------|
| `router` | `0x182a927119D56008d921126764bF884221b10f59` |
| `factory` | `0xDf38F24fE153761634Be942F9d859f3DBA857E95` |
| `weth` (WOKB) | `0xe538905cf8410324e03A5A23C1c177a474D59b2b` |

---

## Security notes

- **Read-only before write.** Every buy / sell / claim is quoted first and only submitted after explicit user confirmation.
- **Slippage is bounded.** Default `100 BPS`. Refuses above `500` without an explicit override.
- **Token validation.** All bonding-curve tokens are checked via `BondingFactory.isValidToken` before any value transfer. Token lists and social sources are treated as untrusted input.
- **Graduation race.** The **1.764 OKB** graduator reward is not guaranteed; it goes to the first successful tx in the block.
- **No private keys.** If a tool surface asks for a raw key, the skill aborts.
- **LP is burned.** Post-graduation Uniswap V2 liquidity is held by `0x...dEaD` and cannot be rugged.

Full threat model and error table are in [`SKILL.md`](./SKILL.md#security-notices).

---

## Repo layout

```
.
├── plugin.yaml                     # plugin-store metadata + manifest
├── .claude-plugin/plugin.json      # Claude plugin manifest (mirror)
├── SKILL.md                        # AI agent instructions
├── reference/
│   ├── addresses.json              # X Layer address manifest
│   └── abi/                        # Minimal JSON ABIs
│       ├── BondingFactory.json
│       ├── BondingCurve.json
│       ├── BondingToken.json
│       ├── ICOFactory.json
│       └── ICOContract.json
├── CONTRIBUTING.md
├── LICENSE                         # MIT
└── README.md
```

---

## License

MIT — see [`LICENSE`](./LICENSE).

---

## Credits and links

- **xpad.lol** — the launchpad protocol this skill wraps.
- **X Layer** — OKX's Ethereum L2 (chain ID `196`). RPC: `https://rpc.xlayer.tech`, explorer: `https://www.oklink.com/x-layer`.
- **Uniswap V2** — canonical Uniswap Labs V2 deployment on X Layer; xpad.lol graduation LPs land directly here.
- **OnchainOS** — Agentic Wallet infrastructure for signing.
- **Uniswap AI Skills** — companion plugin `uniswap-swap-integration` used for post-graduation swaps.
- **OKX Build X Hackathon** — Skills Arena track.
