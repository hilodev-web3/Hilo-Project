# Hilo-Project
A simple on-chain higher-or-lower market for crypto and tokenized assets.

Contract Address: 0x99a5eF49aABAb2cf483d7433A9bff9B85DC30c37
⸻

HILO lets users choose a market, predict whether its price will move higher or lower over a short round, and share the losing side’s stake with everyone who made the correct call.

There is no traditional house taking the opposite side of the trade.

HILO uses a parimutuel model: users compete against the pool, not against the protocol. Winners split the losing side’s stake, while HILO charges a small capped fee from the losing pool. Hilo — Higher or lower on crypto & stocks.pdf

Pick a market
      ↓
Call UP or DOWN
      ↓
Enter the round
      ↓
Round settles on-chain
      ↓
Winners split the pot

⸻

Why HILO Exists

Most short-term trading products are difficult to understand.

Users are often exposed to:

* leverage,
* liquidations,
* complicated order types,
* hidden spreads,
* market makers,
* opaque settlement logic.

HILO reduces the experience to one question:

Will the price be higher or lower when the round ends?

A user does not need to place a limit order, configure leverage, or manage a liquidation price.

They select a market, choose a direction, enter a stake, and wait for settlement.

⸻

Core Principles

1. Parimutuel, not house-backed

HILO does not trade against users.

The winning side receives the losing side’s stake, minus the protocol fee.

UP Pool:      4,000
DOWN Pool:    6,000
Total Pool:  10,000

Suppose the market settles UP.

Winning Pool:        4,000
Losing Pool:         6,000
Protocol Fee:          300
Distributable Loss:   5,700

Each winning user receives:

user payout =
user stake +
(user stake / winning pool) × distributable losing pool

Example:

User UP stake: 1,000
Payout =
1,000 + (1,000 / 4,000) × 5,700
Payout =
1,000 + 1,425
Final payout = 2,425

HILO wins when users participate, not when users lose. Hilo — Higher or lower on crypto & stocks.pdf

⸻

2. Symmetric rounds

Every round has two possible directional outcomes:

UP
DOWN

The contract does not manually adjust odds.

The displayed crowd odds are derived from the amount committed to each side.

const upProbability = upPool / (upPool + downPool);
const downProbability = downPool / (upPool + downPool);

Example:

const upPool = 7_500n;
const downPool = 2_500n;
const totalPool = upPool + downPool;
const upShare = Number(upPool) / Number(totalPool);
const downShare = Number(downPool) / Number(totalPool);
console.log({
  up: `${(upShare * 100).toFixed(2)}%`,
  down: `${(downShare * 100).toFixed(2)}%`,
});

Output:

{
  "up": "75.00%",
  "down": "25.00%"
}

These percentages represent the pool distribution, not a guaranteed probability estimate.

⸻

3. On-chain settlement

Each round settles using an on-chain price source.

For token markets, HILO can use a pool-based time-weighted price. For major crypto assets and tokenized stocks, the product documentation describes using Robinhood Chain’s native Chainlink feeds. Hilo — Higher or lower on crypto & stocks.pdf

The basic settlement rule is:

close price > open price → UP wins
close price < open price → DOWN wins
close price = open price → round voids

Illustrative Solidity logic:

enum Outcome {
    Unresolved,
    Up,
    Down,
    Void
}
function resolveOutcome(
    uint256 openPrice,
    uint256 closePrice
) internal pure returns (Outcome) {
    if (closePrice > openPrice) {
        return Outcome.Up;
    }
    if (closePrice < openPrice) {
        return Outcome.Down;
    }
    return Outcome.Void;
}

No administrator should be able to arbitrarily choose the winning side.

⸻

Round Lifecycle

A HILO round can be modeled as a small state machine.

enum RoundStatus {
    Pending,
    Open,
    Locked,
    Settled,
    Voided
}

Typical flow:

Pending
   ↓
Open
   ↓
Locked
   ↓
Settled

Exceptional flow:

Pending
   ↓
Open
   ↓
Locked
   ↓
Voided

Pending

The round exists but has not started.

Open

Users can enter UP or DOWN positions.

Locked

No additional positions can be submitted.

Settled

The final price has been recorded and payouts are available.

Voided

The round did not meet settlement requirements. Users can reclaim their original stake.

HILO’s documentation states that rounds can be voided when the result is flat, when participation is one-sided, or when a pool or price source behaves unexpectedly. Hilo — Higher or lower on crypto & stocks.pdf

⸻

Example Round Data Structure

The following is illustrative pseudocode, not audited production code.

struct Round {
    uint256 id;
    address market;
    address settlementAsset;
    uint64 opensAt;
    uint64 locksAt;
    uint64 settlesAt;
    uint256 openPrice;
    uint256 closePrice;
    uint256 upPool;
    uint256 downPool;
    RoundStatus status;
    Outcome outcome;
}

A user position could be represented as:

struct Position {
    uint256 amount;
    Outcome direction;
    bool claimed;
}

Storage:

mapping(uint256 roundId => Round) public rounds;
mapping(
    uint256 roundId =>
    mapping(address user => Position)
) public positions;

⸻

Entering a Position

A user selects:

* the market,
* the round,
* the direction,
* the stake amount.

Illustrative Solidity:

error RoundNotOpen();
error InvalidDirection();
error ZeroAmount();
error PositionAlreadyExists();
function enterRound(
    uint256 roundId,
    Outcome direction,
    uint256 amount
) external {
    Round storage round = rounds[roundId];
    if (round.status != RoundStatus.Open) {
        revert RoundNotOpen();
    }
    if (
        direction != Outcome.Up &&
        direction != Outcome.Down
    ) {
        revert InvalidDirection();
    }
    if (amount == 0) {
        revert ZeroAmount();
    }
    Position storage position = positions[roundId][msg.sender];
    if (position.amount != 0) {
        revert PositionAlreadyExists();
    }
    settlementToken.transferFrom(
        msg.sender,
        address(this),
        amount
    );
    position.amount = amount;
    position.direction = direction;
    if (direction == Outcome.Up) {
        round.upPool += amount;
    } else {
        round.downPool += amount;
    }
    emit PositionEntered(
        roundId,
        msg.sender,
        direction,
        amount
    );
}

Event:

event PositionEntered(
    uint256 indexed roundId,
    address indexed user,
    Outcome direction,
    uint256 amount
);

⸻

WETH In, Stable Settlement Out

The HILO product flow describes users entering with WETH and having the position priced in a stable dollar asset. The documentation references conversion into USDG for stable settlement. Hilo — Higher or lower on crypto & stocks.pdf

Conceptually:

User deposits WETH
        ↓
WETH is converted
        ↓
Position is denominated in USDG
        ↓
Round settles
        ↓
Winners claim USDG

A simplified interface might look like:

interface ISettlementRouter {
    function convertToSettlementAsset(
        address inputToken,
        address settlementToken,
        uint256 amountIn,
        uint256 minAmountOut
    ) external returns (uint256 amountOut);
}

Illustrative entry flow:

function enterWithWETH(
    uint256 roundId,
    Outcome direction,
    uint256 wethAmount,
    uint256 minUsdgOut
) external {
    weth.transferFrom(
        msg.sender,
        address(this),
        wethAmount
    );
    weth.approve(address(router), wethAmount);
    uint256 usdgAmount = router.convertToSettlementAsset(
        address(weth),
        address(usdg),
        wethAmount,
        minUsdgOut
    );
    _recordPosition(
        roundId,
        msg.sender,
        direction,
        usdgAmount
    );
}

Production implementation would require:

* slippage protection,
* reentrancy protection,
* allowance management,
* route validation,
* minimum output checks,
* price-impact limits,
* deadline enforcement.

⸻

Payout Calculation

A user’s payout depends on:

* their winning stake,
* the total winning pool,
* the losing pool,
* the protocol fee.

function calculatePayout(
    uint256 userStake,
    uint256 winningPool,
    uint256 losingPool,
    uint256 feeBps
) public pure returns (uint256) {
    uint256 fee = losingPool * feeBps / 10_000;
    uint256 distributable = losingPool - fee;
    uint256 profit =
        userStake * distributable / winningPool;
    return userStake + profit;
}

Example:

uint256 payout = calculatePayout({
    userStake: 1_000e18,
    winningPool: 4_000e18,
    losingPool: 6_000e18,
    feeBps: 500
});

With a 5% fee on the losing pool:

Losing pool:          6,000
Fee:                    300
Distributable:        5,700
User share:             25%
User profit:          1,425
Returned stake:       1,000
Total payout:         2,425

The actual production fee should be read from the deployed contract rather than hardcoded in frontend applications.

⸻

Claiming

After settlement, winning users claim their payout.

error RoundNotSettled();
error NothingToClaim();
error AlreadyClaimed();
function claim(uint256 roundId) external {
    Round storage round = rounds[roundId];
    Position storage position = positions[roundId][msg.sender];
    if (round.status != RoundStatus.Settled) {
        revert RoundNotSettled();
    }
    if (position.amount == 0) {
        revert NothingToClaim();
    }
    if (position.claimed) {
        revert AlreadyClaimed();
    }
    if (position.direction != round.outcome) {
        revert NothingToClaim();
    }
    position.claimed = true;
    uint256 winningPool =
        round.outcome == Outcome.Up
            ? round.upPool
            : round.downPool;
    uint256 losingPool =
        round.outcome == Outcome.Up
            ? round.downPool
            : round.upPool;
    uint256 payout = calculatePayout(
        position.amount,
        winningPool,
        losingPool,
        protocolFeeBps
    );
    settlementToken.transfer(msg.sender, payout);
    emit PayoutClaimed(
        roundId,
        msg.sender,
        payout
    );
}

⸻

Voided Rounds

A round should not settle normally when the result is not clean.

Possible void conditions include:

open price == close price
only one side has participation
price source is stale
price source returns invalid data
market liquidity falls below a threshold
settlement transaction cannot verify the result

Refund logic:

function claimRefund(uint256 roundId) external {
    Round storage round = rounds[roundId];
    Position storage position = positions[roundId][msg.sender];
    require(
        round.status == RoundStatus.Voided,
        "ROUND_NOT_VOIDED"
    );
    require(
        position.amount > 0,
        "NO_POSITION"
    );
    require(
        !position.claimed,
        "ALREADY_CLAIMED"
    );
    position.claimed = true;
    settlementToken.transfer(
        msg.sender,
        position.amount
    );
    emit RefundClaimed(
        roundId,
        msg.sender,
        position.amount
    );
}

HILO’s product messaging explicitly emphasizes refunds when a round is flat, one-sided, or otherwise unsuitable for fair settlement. Hilo — Higher or lower on crypto & stocks.pdf

⸻

Frontend Integration

A frontend can read a round and calculate the visible pool split.

type Round = {
  id: bigint;
  opensAt: bigint;
  locksAt: bigint;
  settlesAt: bigint;
  upPool: bigint;
  downPool: bigint;
  status: number;
};
export function getPoolPercentages(round: Round) {
  const total = round.upPool + round.downPool;
  if (total === 0n) {
    return {
      up: 50,
      down: 50,
    };
  }
  return {
    up: Number((round.upPool * 10_000n) / total) / 100,
    down: Number((round.downPool * 10_000n) / total) / 100,
  };
}

Example output:

const percentages = getPoolPercentages({
  id: 42n,
  opensAt: 0n,
  locksAt: 0n,
  settlesAt: 0n,
  upPool: 9_257n,
  downPool: 743n,
  status: 1,
});
console.log(percentages);
{
  "up": 92.57,
  "down": 7.43
}

The HILO app interface displays the active asset, short-term price chart, round countdown, pool distribution, directional controls, and recent round results in a compact mobile layout. Hilo.pdf

⸻

Example Contract Interaction with viem

import {
  createPublicClient,
  createWalletClient,
  custom,
  http,
  parseUnits,
} from "viem";
const publicClient = createPublicClient({
  transport: http(process.env.NEXT_PUBLIC_RPC_URL),
});
const walletClient = createWalletClient({
  transport: custom(window.ethereum),
});
const [account] = await walletClient.getAddresses();
const roundId = 42n;
const amount = parseUnits("25", 18);
// Example enum:
// 1 = UP
// 2 = DOWN
const direction = 1;
const simulation = await publicClient.simulateContract({
  account,
  address: process.env.NEXT_PUBLIC_HILO_CONTRACT as `0x${string}`,
  abi: hiloAbi,
  functionName: "enterRound",
  args: [roundId, direction, amount],
});
const txHash = await walletClient.writeContract(
  simulation.request
);
const receipt = await publicClient.waitForTransactionReceipt({
  hash: txHash,
});
console.log("Position entered:", receipt.transactionHash);

⸻

Example React Hook

import { useState } from "react";
import { parseUnits } from "viem";
type Direction = "UP" | "DOWN";
export function useEnterHiloRound() {
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  async function enterRound(params: {
    roundId: bigint;
    direction: Direction;
    amount: string;
  }) {
    setIsPending(true);
    setError(null);
    try {
      const encodedDirection =
        params.direction === "UP" ? 1 : 2;
      const amount = parseUnits(params.amount, 18);
      // Replace with your contract client.
      const hash = await writeHiloContract({
        functionName: "enterRound",
        args: [
          params.roundId,
          encodedDirection,
          amount,
        ],
      });
      return hash;
    } catch (cause) {
      const message =
        cause instanceof Error
          ? cause.message
          : "Unable to enter round";
      setError(message);
      throw cause;
    } finally {
      setIsPending(false);
    }
  }
  return {
    enterRound,
    isPending,
    error,
  };
}

⸻

Suggested Architecture

┌────────────────────────────┐
│        HILO Frontend       │
│   Next.js / React / viem   │
└─────────────┬──────────────┘
              │
              │ wallet transactions
              ▼
┌────────────────────────────┐
│      HILO Round Engine     │
│                            │
│ create round               │
│ enter UP / DOWN            │
│ lock round                 │
│ settle round               │
│ claim payout               │
│ claim refund               │
└─────────────┬──────────────┘
              │
              │ reads
              ▼
┌────────────────────────────┐
│     On-chain Pricing       │
│                            │
│ Chainlink feeds            │
│ Pool TWAP sources          │
└─────────────┬──────────────┘
              │
              ▼
┌────────────────────────────┐
│      Settlement Assets     │
│       WETH / USDG          │
└────────────────────────────┘

⸻

Supported and Planned Markets

Based on the current product material:

Crypto

PONS
BTC
ETH

Tokenized stocks

NVDA
AAPL
TSLA

The product roadmap also mentions additional tokenized real-world assets, project listings, additional round lengths, sports, and other real-world outcomes. Some of these are planned or coming soon rather than currently live. Hilo — Higher or lower on crypto & stocks.pdf

⸻

Security Considerations

Any production version of this system should be reviewed for:

Contract security

* reentrancy,
* incorrect pool accounting,
* rounding errors,
* double claims,
* settlement replay,
* unauthorized round creation,
* stale oracle data,
* incorrect decimal normalization,
* fee misconfiguration,
* denial-of-service conditions.

Oracle security

* stale price checks,
* minimum observation windows,
* heartbeat validation,
* Chainlink round completeness,
* TWAP manipulation resistance,
* liquidity thresholds,
* deviation checks.

Economic security

* one-sided rounds,
* last-block entry manipulation,
* frontrunning near lock time,
* low-liquidity market manipulation,
* whale concentration,
* insufficient settlement liquidity.

Frontend security

* chain ID validation,
* contract address verification,
* transaction simulation,
* allowance warnings,
* slippage disclosure,
* clear settlement timing,
* clear void and refund states.

⸻

Example Price Feed Validation

error InvalidPrice();
error StalePrice();
function validatePrice(
    int256 answer,
    uint256 updatedAt,
    uint256 maxAge
) internal view returns (uint256) {
    if (answer <= 0) {
        revert InvalidPrice();
    }
    if (block.timestamp - updatedAt > maxAge) {
        revert StalePrice();
    }
    return uint256(answer);
}

For assets with different decimals, normalize prices before comparison.

function normalizePrice(
    uint256 price,
    uint8 decimals
) internal pure returns (uint256) {
    if (decimals == 18) {
        return price;
    }
    if (decimals < 18) {
        return price * 10 ** (18 - decimals);
    }
    return price / 10 ** (decimals - 18);
}

⸻

Minimal User Flow

From the user’s point of view, HILO should remain much simpler than the underlying system.

1. Connect wallet
2. Pick a market
3. Choose UP or DOWN
4. Enter an amount
5. Confirm
6. Wait for settlement
7. Claim payout or refund

The product describes this as “four taps to a position,” emphasizing a fast and simple entry experience. Hilo — Higher or lower on crypto & stocks.pdf

⸻

Developer Philosophy

HILO’s protocol design should follow a few rules:

Settlement should be deterministic.
Pool accounting should be transparent.
A winning result should not depend on an admin decision.
A voided result should return principal.
The protocol should not profit from choosing the winning side.
The frontend should explain the risk before the transaction.

The complexity belongs inside the protocol.

The user experience should remain:

Higher or lower?

⸻

Repository Structure

A possible monorepo structure:

hilo/
├── apps/
│   ├── web/
│   │   ├── app/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── lib/
│   └── indexer/
│       ├── src/
│       └── schema/
│
├── packages/
│   ├── contracts/
│   │   ├── src/
│   │   │   ├── HiloMarket.sol
│   │   │   ├── RoundManager.sol
│   │   │   ├── SettlementRouter.sol
│   │   │   └── PriceAdapter.sol
│   │   ├── test/
│   │   └── script/
│   │
│   ├── sdk/
│   │   ├── src/
│   │   └── package.json
│   │
│   ├── ui/
│   │   ├── components/
│   │   └── tokens/
│   │
│   └── config/
│
├── docs/
│   ├── architecture.md
│   ├── settlement.md
│   ├── markets.md
│   └── risk.md
│
├── foundry.toml
├── pnpm-workspace.yaml
└── README.md

⸻

Local Development

git clone https://github.com/hilo-markets/hilo.git
cd hilo
pnpm install
