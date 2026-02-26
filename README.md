# ClawdGames Platform — Design Document
*Draft v0.1 — Feb 26, 2026*

---

## The One-Liner

> *"Vibe-code a deterministic onchain game in an afternoon. Launch a fair 60-minute token auction. Let your community play to earn — all paired with CLAWD."*

---

## What This Is

ClawdGames is a platform with two audiences:

1. **Game builders** — developers (or AI agents) who vibe-code a game using our SDK, then launch a token for it
2. **Players** — people who play the game using that token, win tokens back, never touch a wallet

The platform handles everything between them: token launch, fair price discovery, permanent liquidity, fee splits, and the randomness that makes games provably fair.

---

## The Three Steps

### Step 1: Build
The game builder fetches a skill file, vibe-codes a deterministic game using the SDK, and deploys it. The game must use onchain randomness and implement a `/simulate` route.

### Step 2: Launch
The builder comes to clawdgames.xyz, fills in token name/symbol/logo/game URL, signs one transaction. A 60-minute fair auction runs. The token launches on Uniswap v4 at the price everyone paid.

### Step 3: Play
Players find the game, sign in with email, spend tokens to play, win tokens back. No wallets. No crypto knowledge. Just play.

---

## The Stack

```
Builder vibe-codes game using skill file + SDK
        ↓
Game deployed at gameUrl (e.g. puzzlequest.xyz)
        ↓
Builder registers on clawdgames.xyz → 60-min CCA auction via Flow Protocol
        ↓
GAME Token (ERC-20, Base) launched on Uniswap v4, paired with CLAWD
        ↓
Players visit gameUrl → sign in with email (account abstraction)
        ↓
Player spends GAME tokens → SDK calls Game Contract
        ↓
Game Contract requests seed from Randomness Oracle (Base)
        ↓
Deterministic outcome: win = tokens back, lose = tokens to prize pool
        ↓
20% of all trading fees → buy CLAWD → burn to 0xdead forever
```

---

# PART 1: BUILDING A GAME

## 1.1 The Game Builder Skill File

When a builder wants to make a ClawdGames-compatible game, they (or their AI agent) fetch:

```
GET https://clawdgames.xyz/skill/skill.md
```

This skill file tells them exactly how to build a compatible game. It covers:
- How to integrate the SDK
- How the randomness oracle works
- How to implement the `/simulate` route
- What the game contract interface must look like
- How to test locally before launch

The skill file is the complete spec. A vibe-coder or AI agent should be able to read it and build a working game without asking any questions.

## 1.2 What Makes a Valid ClawdGames Game

A game must satisfy four requirements:

**1. Deterministic outcomes**
Given the same seed, the game always produces the same result. No hidden server state. No client-side randomness. Pure function: `outcome = f(seed, gameState)`.

**2. Onchain randomness**
The seed comes from the ClawdGames Randomness Oracle contract on Base — not from `Math.random()`, not from the server, not from the client. The oracle uses a combination of block hash + player address + nonce to produce a `bytes32` seed that is unpredictable before the block is mined but verifiable after.

**3. The `/simulate` route**
```
GET /simulate?seed=0x{bytes32}
```
Returns the full deterministic game outcome for any given seed. Must work for any valid seed, including past seeds. This is how players verify results and how the platform audits games.

**4. Token integration via SDK**
The game uses the SDK to handle:
- Charging the player tokens to play
- Requesting the onchain seed
- Resolving the outcome
- Paying out tokens if win

## 1.3 The Game SDK Interface

```typescript
import { ClawdGamesSDK } from '@clawdgames/sdk'

const sdk = new ClawdGamesSDK({
  gameTokenAddress: '0x...',    // the GAME token for this game
  gameContractAddress: '0x...',  // the deployed game contract
  rpcUrl: 'https://mainnet.base.org',
})

// Charge player and play one round
// Returns: { seed, outcome, tokensWon, txHash }
const result = await sdk.playRound({
  playerAddress: '0x...',
  betAmount: 10,               // tokens to spend
  gameState: { /* game-specific */ }
})

// Simulate any seed without spending tokens (off-chain)
// Returns: { outcome, tokensWon }
const preview = sdk.simulate({
  seed: '0x1234abcd...',
  betAmount: 10,
  gameState: { /* game-specific */ }
})

// Get the randomness seed for a completed round (for verification)
const seed = await sdk.getSeed({
  blockHash: '0x...',
  playerAddress: '0x...',
  nonce: 42
})
```

## 1.4 The Randomness Oracle Contract

Deployed on Base. Called by the game contract during `playRound()`.

```solidity
interface IRandomnessOracle {
  /// @notice Returns a deterministic seed for a given player + nonce
  /// Uses: keccak256(blockhash(block.number - 1), playerAddress, nonce)
  /// Unpredictable before block mines, verifiable after.
  function getSeed(address player, uint256 nonce) external view returns (bytes32);
}
```

**Why not Chainlink VRF?**
VRF requires a callback and adds latency + cost. For games with small token amounts on a fast L2, block-hash-based randomness is sufficient and instant. Miners on Base cannot practically manipulate this for small-stakes games. If game stakes grow large enough to incentivize miner manipulation, upgrade to VRF.

## 1.5 The Game Contract Interface

Every game deploys its own game contract that implements:

```solidity
interface IClawdGame {
  /// @notice Play one round. Charges betAmount tokens, resolves outcome, pays out if win.
  /// @param player    The player's address (receives tokens if win)
  /// @param betAmount Tokens to spend
  /// @param gameState Encoded game-specific state (e.g. move choices)
  /// @return outcome  Encoded outcome (game-specific)
  /// @return payout   Tokens sent back to player (0 if loss)
  function playRound(
    address player,
    uint256 betAmount,
    bytes calldata gameState
  ) external returns (bytes memory outcome, uint256 payout);

  /// @notice Pure deterministic simulation — no state changes, no token spend.
  /// @param seed      The bytes32 seed to simulate with
  /// @param betAmount Tokens that would be spent
  /// @param gameState Encoded game-specific state
  /// @return outcome  What would happen
  /// @return payout   Tokens that would be won
  function simulate(
    bytes32 seed,
    uint256 betAmount,
    bytes calldata gameState
  ) external pure returns (bytes memory outcome, uint256 payout);
}
```

## 1.6 The `/simulate` HTTP Route

The game's web server must expose:

```
GET /simulate?seed=0x{bytes32}&betAmount={number}&gameState={encoded}
```

Response:
```json
{
  "seed": "0x1234...",
  "outcome": { /* game-specific result */ },
  "tokensWon": 0,
  "isWin": false,
  "verifyUrl": "/simulate?seed=0x1234..."
}
```

This route calls the contract's `simulate()` function off-chain (eth_call, no gas). The result is identical to what would happen onchain with the same seed.

## 1.7 Game Economics — How Token Flow Works

```
Player has 100 GAME tokens
        ↓
Plays a round, bets 10 GAME
        ↓
SDK calls playRound(player, 10, gameState)
  → transfers 10 GAME from player to game contract
  → calls oracle.getSeed(player, nonce)
  → runs deterministic logic
        ↓
If WIN:
  → game contract sends back 18 GAME (example: 1.8x payout)
  → player net: -10 + 18 = +8 GAME
        ↓
If LOSS:
  → 10 GAME stays in game contract (prize pool)
  → player net: -10 GAME
```

The game contract accumulates GAME tokens from losses. These fund future wins. Game designers set their own win probability and payout multiplier — the math must be sustainable (house edge > 0 or the contract goes bankrupt).

**Platform does not dictate game mechanics beyond the interface.** A slot machine, a dice game, a card game, a puzzle — all valid as long as they use the randomness oracle and implement the interface.

## 1.8 No Wallet UX for Players

Players never see a wallet, private key, or seed phrase. Account abstraction handles it:

- Player signs in with email or social (Google, Twitter, etc.)
- A smart wallet is created silently in the background
- `"Play for 10 PUZZLE"` → player taps button → game plays
- Winnings arrive in their smart wallet automatically
- They can later export their wallet or connect to any dApp if they want

The player experience is indistinguishable from a normal mobile game. The token layer is invisible.

**Account abstraction provider:** TBD (options: Privy, Coinbase CDP, Biconomy, ZeroDev)

---

# PART 2: THE TOKEN LAUNCH

## 2.1 Registration on ClawdGames

When a game is ready, the builder comes to clawdgames.xyz and provides:

| Field | Required | Example |
|---|---|---|
| Token name | Yes | "Puzzle Quest" |
| Token symbol | Yes | "PUZZLE" |
| Logo URL | Yes | Square PNG/JPG, 1:1, <1MB, direct URL |
| Game URL | Yes | https://puzzlequest.xyz |
| Description | No | "A sliding puzzle game where winning earns tokens" |

The platform:
1. Fetches live CLAWD/USD price for `currencyPriceUsd`
2. Deploys a `GameTokenFeeSplitter` contract for the creator's wallet
3. Builds launch tx via Flow API
4. Creator signs ONE transaction (~$0.001 ETH gas)
5. 5-minute countdown starts
6. 60-minute auction runs
7. Platform auto-calls `deployLiquidity()` when auction ends + graduates
8. Platform sets fee recipient to the splitter contract
9. Token is live

## 2.2 The Canonical Launch Config (Non-Negotiable)

Every ClawdGames launch uses these exact parameters. The platform hardcodes them:

```javascript
custom: {
  auctionPercent: 40,            // 40% to community via CCA
  reservePercent: 50,            // 50% to v4 liquidity (permanent)
  vestingPercent: 8,             // 8% to creator, 6-month linear vest
  deployerImmediatePercent: 2,   // 2% to creator at liquidity deploy
  deployerShareBps: 500,         // 5% of raise to creator
  vestingDuration: 15552000,     // 6 months (seconds)
  cliffDuration: 0,              // no cliff
  startDelayBlocks: 150,         // ~5 min countdown
  durationBlocks: 1800,          // 60-minute auction
  floorFdvUsd: 10000,            // $10K floor
  requiredRaiseUsd: 500,         // $500 minimum to graduate
  currency: "0x9f86dB9fc6f7c9408e8Fda3Ff8ce4e78ac7a6b07",  // CLAWD
  currencyPriceUsd: "<live CLAWD/USD at launch time>",
}
```

## 2.3 Token Allocation

| Who | % | Tokens (of 1B) | Notes |
|---|---|---|---|
| Community (auction) | 40% | 400,000,000 | Fair CCA price discovery |
| v4 Liquidity | 50% | 500,000,000 | Permanent, locked forever |
| Creator (vested) | 8% | 80,000,000 | Linear 6 months, no cliff |
| Creator (immediate) | 2% | 20,000,000 | At liquidity deploy |

## 2.4 What the Creator Gets

| Stream | Amount (example: $5K raise, $50K FDV) | When |
|---|---|---|
| Cash from raise | $250 (5% of $5,000) | At auction settlement |
| Immediate tokens | 20M PUZZLE = $1,000 | At liquidity deploy |
| Vested tokens | 80M PUZZLE = $4,000 over 6 months | Monthly |
| Swap fees (ongoing) | 80% of 75% of 1.7% of all volume | Forever |

The **swap fees are the real prize**. A game with $10K/day volume pays ~$37K/year to the creator. The creator only earns this if the game stays alive and people keep playing.

## 2.5 The Auction Experience

```
T-0:    Creator signs tx. 5-minute countdown begins.
        → Creator posts to Twitter/Discord: "PUZZLE launching in 5 min"

T+5m:   Auction opens. 60-minute window.
        → Live feed: current clearing price, # wallets, CLAWD raised
        → Bidders enter USDC → auto-swapped to CLAWD → bid submitted

T+65m:  Auction closes. Clearing price locked.
        → Flow deploys v4 pool at EXACT clearing price
        → PUZZLE live on Uniswap v4
        → Nobody paid less than clearing price. No insiders.

T+65m+: Bidders claim PUZZLE tokens.
        → Creator claims 2% immediate tokens.
        → 8% starts vesting linearly.
```

## 2.6 If the Auction Fails (Doesn't Reach $500)

- All bidders get full CLAWD refund via `exitBid`
- Creator sweeps unsold tokens via `/deployer/sweep/build-tx`
- No harm done. Game can try again.

---

# PART 3: THE FEE ECONOMY

## 3.1 The Fee Splitter Contract

Deployed once per game at launch. Immutable. No admin keys.

```
Every trade on the PUZZLE/CLAWD v4 pool generates 1.7% fee
        ↓
MEV bots collect fees from pool (they earn 0.05% tip for this)
        ↓
Flow hook auto-converts fees to USDC
        ↓
Creator (or anyone) calls claimFees() → USDC lands in splitter
        ↓
Anyone calls splitter.distribute(USDC)
        ↓
    80% → creator wallet (ongoing income)
    20% → swaps to CLAWD → burns to 0xdead
```

The creator never needs to understand any of this. The platform's creator dashboard shows: "You have $142 in unclaimed fees. [Claim]"

## 3.2 The CLAWD Flywheel

```
More games launch
  → USDC bids auto-swap to CLAWD on launch  [buy pressure]
  → 20% of all game swap fees buy + burn CLAWD  [deflationary]
  → CLAWD scarcer → more valuable
  → Launching here is more prestigious
  → More creators want to launch here
  → Repeat
```

Every successful game permanently removes CLAWD from supply. The platform's success is directly expressed in CLAWD's scarcity.

---

# PART 4: WHAT STILL NEEDS BUILDING

| Component | What It Is | Priority |
|---|---|---|
| `clawdgames.xyz/skill/skill.md` | The game builder skill file — complete spec for building a compatible game | 🔴 First |
| `RandomnessOracle.sol` | Onchain seed contract on Base | 🔴 First |
| `IClawdGame.sol` | Game contract interface + example implementation | 🔴 First |
| Game SDK (`@clawdgames/sdk`) | JS/TS library wrapping playRound, simulate, getSeed | 🔴 First |
| `GameTokenFeeSplitter.sol` | Immutable fee split: 80% creator, 20% CLAWD burn | 🔴 First |
| ClawdGames launch UI | Form: name/symbol/logo/gameUrl → builds Flow tx | 🟡 Second |
| CLAWD/USD price feed | Live TWAP for `currencyPriceUsd` at launch time | 🟡 Second |
| USDC→CLAWD swap | Pre-bid swap so bidders use USDC not CLAWD | 🟡 Second |
| Auto deploy-liquidity keeper | Monitors `GET /launches/graduated`, calls deployLiquidity | 🟡 Second |
| Creator dashboard | Fee claims, vesting claims, game stats | 🟢 Third |
| Account abstraction | Invisible wallets for players | 🟢 Third |
| Auction live feed UI | Real-time price, wallets in, CLAWD raised | 🟢 Third |

---

# PART 5: FLOW PROTOCOL INTEGRATION REFERENCE

*This section is for the agent building the launch + bidding flows. It contains every API call, contract address, and function selector needed.*

## 5.1 Flow Contracts (Base Mainnet)

| Contract | Address |
|---|---|
| LiquidLaunch | `0x87c281F8287B97Ca2167a85e6e356b74C75aa233` |
| AuctionManager | `0xF762AC1553c29Ef36904F9E7F71C627766D878b4` |
| FullRangeStrategy | `0x63BFc6F4Db30959c8dc12ddBE702a12EeEdE86E5` |
| USDC (Base) | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| WETH (Base) | `0x4200000000000000000000000000000000000006` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| **CLAWD** | **`0x9f86dB9fc6f7c9408e8Fda3Ff8ce4e78ac7a6b07`** |

## 5.2 Flow API Base URL

```
https://api.flow.bid
```

## 5.3 Key API Calls

**Launch a token:**
```javascript
POST /launches/build-tx
Body: { deployer, name, symbol, metadata: { logo, description }, custom: { ...canonicalConfig } }
Returns: { to, data, value, predictedTokenAddress, predictedAuctionAddress, auctionTiming }
```

**Submit a bid (2 txs — sequential):**
```javascript
POST /bids/build-tx
Body: { bidder, auctionAddress, maxFdvUsd, amount, currencyPriceUsd }
Returns: { transactions: [{ step:1, approve... }, { step:2, submitBid... }], params }
// step 1 selector: 0x095ea7b3 (approve CLAWD)
// step 2 selector: 0x1ab71b17 (submitBid)
// WAIT for step 1 confirmation before step 2
```

**Deploy liquidity after auction (2 txs — sequential):**
```javascript
POST /liquidity/build-tx
Body: { auctionAddress }
Returns: { transactions: [{ step:1, checkpoint }, { step:2, deployLiquidity }] }
// Precondition: isGraduated === true AND liquidityDeployed === false
// GET /launches/{auctionAddress} to check
```

**Bidder claims tokens:**
```javascript
POST /claims/build-tx
Body: { auctionAddress, bidId }
Returns: { transaction, params: { claimMethod } }
// claimMethod: "claimFullyFilledBid" | "claimPartiallyFilledBid" | "exitBid"
```

**Creator claims LP fees:**
```javascript
POST /deployer/fees/build-tx
Body: { auctionAddress }
Returns: { transaction }  // claimFees() — selector 0x446568bd
```

**Set fee recipient to splitter:**
```javascript
POST /deployer/fee-recipient/build-tx
Body: { auctionAddress, newRecipient: splitterContractAddress }
// Must be called by deployer (creator wallet)
```

**Creator claims vested tokens:**
```javascript
POST /deployer/vesting/build-tx
Body: { auctionAddress }
Returns: { transaction, vestingInfo: { totalVesting, claimed, remaining } }
```

**Monitor graduated auctions (for keeper):**
```javascript
GET /launches/graduated
// Returns auctions where isGraduated===true AND liquidityDeployed===false
```

**Safety check before bidding:**
```javascript
GET /launches/{auctionAddress}/safety
// Check: rawMetrics.tokenWasDeployed === true
```

## 5.4 Block Timing

```
Base: ~2 seconds per block
150 blocks  ≈ 5 minutes   (startDelayBlocks)
1800 blocks ≈ 60 minutes  (durationBlocks)

GET /block/current → { blockNumber, timestamp (ms) }
startTimestamp = currentTimestamp + (startBlock - currentBlock) * 2000
```

## 5.5 Flow Skill File URLs

| Resource | URL |
|---|---|
| Skill root | https://www.flow.bid/skill/skill.md |
| Launch | https://www.flow.bid/skill/resources/launch.md |
| Submit bid | https://www.flow.bid/skill/resources/submit-bid.md |
| Claim tokens | https://www.flow.bid/skill/resources/claim-tokens.md |
| Deploy liquidity | https://www.flow.bid/skill/resources/deploy-liquidity.md |
| Deployer admin | https://www.flow.bid/skill/resources/deployer-admin.md |
| Constants & schemas | https://www.flow.bid/skill/references/constants-and-schemas.md |

---

## CLAWD Token Reference

| | |
|---|---|
| Contract (Base) | `0x9f86dB9fc6f7c9408e8Fda3Ff8ce4e78ac7a6b07` |
| Name | clawd.atg.eth |
| Symbol | CLAWD |
| Decimals | 18 |
| Liquidity (Feb 2026) | ~$1.84M in Uniswap v4 |
| 24h Volume | ~$497K |
| FDV | ~$7.34M |
| DexScreener | https://dexscreener.com/base/0x9fd58e73d8047cb14ac540acd141d3fc1a41fb6252d674b730faf62fe24aa8ce |
| Website | https://clawdbotatg.eth.link |
| Twitter | https://x.com/clawdbotatg |

---

*Next: Write the game builder skill file at `clawdgames.xyz/skill/skill.md`*
