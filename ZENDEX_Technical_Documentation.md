# ZENDEX Technology — Technical Documentation

**A Plain-English Guide to How ZENDEX Works**

---

## 1. What Is ZENDEX?

ZENDEX (ZK-AMM) is a **privacy-preserving decentralized exchange (DEX)** built on the Horizen blockchain. It lets users trade cryptocurrency tokens, provide liquidity, and place limit orders — all without revealing their balances, trade amounts, or identity to anyone observing the blockchain.

Most existing DEXes (like Uniswap) record every transaction publicly. Anyone can see exactly who traded what, for how much, and when. ZENDEX solves this by wrapping every financial operation inside a **Zero-Knowledge Proof (ZKP)** — a mathematical technique that proves "I did this correctly" without revealing any of the underlying private data.

Think of it like proving you're old enough to enter a bar without showing your ID — the bouncer gets a verified "yes/no" answer but learns nothing else about you.

---

## 2. Core Problem ZENDEX Solves

| Problem on public DEXes | How ZENDEX solves it |
|---|---|
| Balances are visible on-chain | Balances are hidden inside cryptographic commitments |
| Trade amounts are public | Amounts are encoded in private ZK proofs |
| Wallet addresses can be tracked | Deposits and withdrawals are cryptographically unlinkable |
| Limit orders reveal trading intent | Orders are private until matched and executed |
| Gas costs from repeated proof generation | One "inclusion proof" is reused across many operations |

---

## 3. Key Technologies Used

**UltraPlonk Proofs via Noir + Barretenberg** — The core privacy engine. ZENDEX uses the **UltraPlonk** proving system, a modern ZK construction that produces compact proofs and supports fast on-chain verification. Proofs are generated using the **Barretenberg** library (from Aztec), and the circuits that define the rules are written in **Noir**, a domain-specific language for ZK programming. UltraPlonk is distinct from older systems like Groth16 — it does not require a per-circuit trusted setup, making it more flexible and trustless.

**Poseidon Hash** — A hash function (like a one-way fingerprint) specially optimized for ZK circuits. ZENDEX uses it everywhere to "seal" private data (amounts, keys, blinding factors) into short, opaque commitments.

**Merkle Tree** — A data structure that efficiently stores all user commitments. It allows anyone to prove that a specific commitment exists in a large set without revealing what the others are.

**UUPS Upgradeable Contracts** — ZENDEX's smart contracts are written with the ability to be upgraded by the team over time without users needing to move their funds. (UUPS = Universal Upgradeable Proxy Standard, an OpenZeppelin pattern.)

**Horizen Testnet** — The blockchain ZENDEX is currently deployed on (Chain ID: 2651420). Horizen is an EVM-compatible blockchain well-suited for ZK applications.

---

## 4. How ZENDEX Works — The Big Picture

The entire protocol revolves around one idea: **instead of storing your token balance on-chain, ZENDEX stores a cryptographic commitment to your balance.** Only you know the secret values behind that commitment.

Every operation (deposit, trade, withdraw) requires you to produce a ZK proof that you know the right secrets — and the proof is verified on-chain without revealing those secrets.

### The Commitment

When you deposit funds, the protocol creates a fingerprint of your deposit:

```
commitment = Poseidon(asset_id, amount, Poseidon(random_blinding, public_key_x, public_key_y))
```

This commitment is stored on-chain in a Merkle tree. Nobody can reverse-engineer your balance or identity from it.

### The Nullifier

When you spend a commitment (withdraw, swap, etc.), you produce a **nullifier**:

```
nullifier = Poseidon(Poseidon(random_blinding), public_key_x, public_key_y)
```

The nullifier is published on-chain to mark that commitment as "spent." This prevents double-spending — you can't use the same deposit twice — without revealing which commitment was spent or who owned it.

---

## 5. The Inclusion Proof — ZENDEX's Key Innovation

One of ZENDEX's most important technical contributions is the **reusable inclusion proof**. Here's why it matters.

In a typical ZK system, every time you want to spend a commitment, you must generate a fresh Merkle proof showing your commitment is in the tree. This is computationally expensive and slow.

ZENDEX separates this into two steps:

**Step 1 — After Deposit (done once):** After your deposit transaction is confirmed, the system generates a single **inclusion proof** off-chain. This proof says: *"Commitment X exists in the Merkle tree at epoch Y."*

**Step 2 — Every subsequent operation:** Instead of re-proving Merkle inclusion each time, you present the pre-generated inclusion proof. The on-chain contract verifies a compact tag:

```
tag = Poseidon(epoch_id, commitment, salt)
```

This same inclusion proof can be reused for withdrawals, swaps, liquidity operations, and orders — as many times as needed within the same epoch.

**Benefits:**
- Smaller proofs → lower gas fees
- Faster transaction preparation
- One proof does the work of many
- Higher throughput for the whole system

---

## 6. The Four Main Modules

ZENDEX is organized into four functional modules, each handling a different part of the DeFi experience.

---

### Module 1 — The Vault (Private Asset Management)

The **Vault** is where users privately hold tokens. It supports four operations:

**Deposit** — You lock tokens into the vault and receive a private cryptographic commitment. A ZK proof ensures the commitment is correctly formed. After the deposit is mined, an inclusion proof is generated and stored for reuse.

**Withdraw** — You prove ownership of a commitment (using the inclusion proof + a spend proof) and reclaim your tokens. A nullifier is published to prevent double-withdrawal.

**Split** — You take one commitment and split it into two smaller commitments privately. Useful for making partial payments or distributing assets.

**Join** — You merge two commitments into one larger commitment. Useful for consolidating funds while maintaining privacy.

All four operations require valid ZK proofs verified against on-chain verifier contracts. No private information is ever revealed during any of these operations.

---

### Module 2 — The AMM (Private Trading)

The **AMM (Automated Market Maker)** is ZENDEX's private trading engine. It is modeled on the Uniswap V2 **constant product formula** (`x * y = k`), which automatically prices tokens based on supply ratios in a liquidity pool.

Unlike Uniswap, every ZENDEX trade is private. Three operations are supported:

**Add Liquidity** — Deposit two tokens into a trading pair pool privately. Your LP (liquidity provider) position is stored as a private commitment, not a public balance.

**Remove Liquidity** — Reclaim your share of a pool privately. The output tokens become a new commitment.

**Swap** — Trade one token for another. The swap is validated by a ZK circuit (checking correct pricing, slippage, and nullifier). The output is inserted as a new private commitment in the Merkle tree.

Key constraints:
- Maximum slippage per swap is capped at 5% (500 basis points)
- Any leftover "dust" tokens after operations go to a designated dust recipient
- The AMM routing is handled by a dedicated Router contract that also manages fee collection

---

### Module 3 — The Order Book (Private Limit Orders)

ZENDEX includes a **private limit order book** — a rarity in DeFi. It supports four operations with a full lifecycle:

**Create Order** — A user creates a limit order (e.g., "sell X token at price P") using an inclusion proof and a ZK proof. A "spot commitment" representing the order is stored in the tree.

**Request Cancel** — A user who wants to cancel generates a cancellation ZK proof and submits a cancel request. This is privacy-preserving — only the user can authorize their own cancellation.

**Cancel Order (finalized by Operator)** — A trusted Operator role finalizes the cancellation and releases a refund commitment to the user.

**Execute Orders (by Operator)** — When two orders match (one buyer, one seller), the Operator batches and executes them. The execution is verified on-chain using ZK proofs, and the outputs become new private commitments for each party.

The Operator has a specific role (`ORDER_BOOK_OPERATOR_ROLE`) and can only execute or cancel orders — never move funds unilaterally.

---

### Module 4 — Fees, Staking & Rewards

ZENDEX has an integrated economic system to reward liquidity providers and active participants.

**Fee Structure** — Every trade incurs a 0.25% base fee, split as follows:
- **0.18%** → stays in the liquidity pair as rewards for LPs
- **0.07%** → sent to the RewardsEngine, split three ways:
  - **0.025%** → Protocol Treasury
  - **0.0225%** → User Cashback
  - **0.0225%** → Burn (permanently removed from supply)

**Staking Boosts** — Users can stake the ZKZ governance token to earn a **boost** on their cashback. The boost converts a portion of the "burn" allocation into additional cashback. Boost ranges from 5% (minimum stake, minimum lock) to 25% (maximum stake, maximum lock). Treasury and LP fees are unaffected by boosts.

Boost formula:

```
boost = 500 + 2000 × average(amount_factor, duration_factor)
boost capped at 2500 basis points (25%)
```

Staking parameters: minimum 1,000 ZKZ, lockup from 1 to 24 periods (e.g., 30-day periods).

**Claiming Rewards** — Users call a single `claim()` function which automatically converts all accumulated fee tokens into native ZEN coins via the router and delivers cashback to the user's wallet. The burn portion is sent to the zero address.

---

## 7. The ZK Verification Layer

Every privacy-sensitive operation in ZENDEX is backed by a dedicated ZK circuit written in Noir and proved using Barretenberg's UltraPlonk backend. Each circuit is compiled into a **Solidity verifier contract** that lives on-chain and validates proofs before any state changes occur.

| Circuit | What it proves |
|---|---|
| DepositVerifier | Correct commitment formation on deposit |
| InclusionVerifier | Commitment exists in Merkle tree at a given epoch |
| WithdrawVerifier | Ownership + nullifier validity on withdrawal |
| SplitVerifier | Correct split of one commitment into two |
| JoinVerifier | Correct merge of two commitments into one |
| AddLiquidityVerifier | Valid private liquidity addition |
| RemoveLiquidityVerifier | Valid private liquidity removal |
| SwapVerifier | Correct private swap execution |
| CreateOrderVerifier | Valid limit order creation |
| RequestCancelOrderVerifier | Authorized order cancellation request |

All verifiers are registered in a central **ZendexVerifierHub** contract, making it easy to upgrade individual verifiers as the proof system improves.

Every spend circuit follows the same four-step internal pattern:
1. Verify the inclusion proof (commitment is in the tree)
2. Verify the ECDSA signature (user authorized this action)
3. Compute the nullifier (mark this commitment as spent)
4. Validate all public outputs are consistent

---

## 8. The Merkle Tree (TreeOperator)

All private commitments live in an **Incremental Merkle Tree**, managed by the `TreeOperator` contract. Key properties:

- Hash function: Poseidon (ZK-friendly, much cheaper to use inside circuits than SHA-256)
- Depth: configurable from 1 to 32 levels (default deployment uses depth 8, allowing up to 256 commitments)
- Epoch tracking: each time the tree state changes, a new epoch ID is recorded; inclusion proofs are scoped to a specific epoch, making them tamper-evident
- The tree stores: commitments, nullifiers, LP positions, and order metadata

---

## 9. Contract Architecture Summary

```
ZENDEX ZK-AMM
│
├── VAULT MODULE
│   ├── ZendexVaultManager      ← handles deposit/withdraw/split/join (upgradeable)
│   ├── ZendexVault             ← holds actual token custody (non-upgradeable)
│   └── TreeOperator            ← manages the Merkle tree (non-upgradeable)
│
├── AMM MODULE
│   ├── ZendexAmmManager        ← handles swap/addLiquidity/removeLiquidity (upgradeable)
│   ├── ZendexFactory           ← creates trading pairs (upgradeable)
│   ├── ZendexRouter            ← routes trades and collects fees (upgradeable)
│   └── ZendexPair              ← constant product pool per token pair (non-upgradeable)
│
├── ORDER BOOK MODULE
│   └── ZendexOrderBookManager  ← full limit order lifecycle (upgradeable)
│
├── FEE & REWARDS MODULE
│   ├── RewardsEngine           ← collects, distributes, converts fees (upgradeable)
│   ├── ZendexStaking           ← ZKZ token staking (upgradeable)
│   └── BoostManager            ← calculates boost multipliers (upgradeable)
│
└── VERIFICATION MODULE
    ├── ZendexVerifierHub        ← registry of all ZK verifiers (upgradeable)
    └── 10× Verifier contracts   ← one per ZK circuit (auto-generated from Noir)
```

All manager contracts use the UUPS proxy pattern, meaning the team can deploy improved implementations without changing contract addresses or requiring user migration.

---

## 10. Privacy Guarantees

ZENDEX provides the following provable privacy properties:

**Transaction Unlinkability** — Deposits and withdrawals cannot be linked to each other on-chain. An outside observer sees only that "some commitment was created" and "some nullifier was published," but cannot connect them.

**Amount Privacy** — The value of any deposit, trade, or withdrawal is never revealed on-chain. It exists only inside the commitment.

**Asset Privacy** — Which token is being moved remains private within ZK proofs.

**Balance Privacy** — No user balance is ever stored on-chain; only commitments are stored.

**Liquidity Privacy** — LP positions are private commitments, not public token balances.

**Order Privacy** — Limit orders are not readable by market participants until they are executed.

---

## 11. Supported Assets

| Token | Role | Type |
|---|---|---|
| WZEN | Wrapped native coin for trading | ERC20 |
| USDT | Stablecoin | ERC20 |
| USDC | Stablecoin | ERC20 |
| DAI | Stablecoin | ERC20 |
| ZKZ | Governance/staking token | ERC20 |

Liquidity pairs: WZEN/USDT, WZEN/USDC, WZEN/DAI

---

## 12. Typical User Journey (End to End)

Here is a concrete example of what happens when a user trades privately on ZENDEX:

1. **User deposits USDT.** A UltraPlonk proof is generated client-side confirming the commitment is correctly formed. The transaction is submitted. Tokens are locked in `ZendexVault`.

2. **Inclusion proof is generated.** After the deposit transaction is mined, an off-chain service generates an inclusion proof confirming that the commitment exists in the Merkle tree at the current epoch. This proof is stored and reused.

3. **User swaps USDT → WZEN.** The user generates a swap proof using the inclusion proof from step 2. The proof confirms they own the USDT commitment and that the swap math is correct. The USDT commitment is nullified (spent), and a new WZEN commitment is created.

4. **User places a limit order.** Using the same inclusion proof, the user creates an order. A "spot commitment" representing the order position is stored on-chain.

5. **Order is matched.** The Operator matches the user's order with a counterparty and calls `executeOrders()`. Both parties receive new private commitments representing their trade outputs.

6. **User withdraws WZEN.** The user generates a withdrawal proof. Their WZEN commitment is spent (nullifier published), and the corresponding token amount is transferred from `ZendexVault` to their wallet.

Throughout this entire journey, no on-chain observer ever sees the user's balance, the trade amount, or which commitment belongs to which address.

---

## 13. Glossary

| Term | Simple Explanation |
|---|---|
| ZK Proof | A mathematical proof that "I know a secret" without revealing the secret |
| UltraPlonk | The specific ZK proving system used by ZENDEX — modern, efficient, no per-circuit trusted setup |
| Noir | The programming language ZENDEX uses to write ZK circuits |
| Barretenberg | Aztec's ZK proving library that generates and verifies UltraPlonk proofs |
| Commitment | A cryptographic "seal" on private data; short and opaque |
| Nullifier | A one-time code published when a commitment is spent, to prevent reuse |
| Merkle Tree | A tree-shaped data structure that efficiently proves membership |
| Poseidon | A hash function optimized for ZK circuits |
| Epoch | A snapshot of the Merkle tree state, used to scope inclusion proofs |
| Inclusion Proof | A proof that a commitment is in the Merkle tree at a given epoch |
| AMM | Automated Market Maker — prices tokens automatically based on pool ratios |
| UUPS | A smart contract upgrade pattern that keeps addresses stable |
| Boost | A multiplier that converts burn fees into cashback for stakers |
| Basis Points | 1/100th of a percent. 100 bps = 1%, 2500 bps = 25% |

---

## 14. Summary

ZENDEX is a full-featured decentralized exchange that brings **genuine financial privacy** to DeFi. It does this without sacrificing correctness or security — every private operation is verified on-chain using UltraPlonk proofs generated by Noir circuits and the Barretenberg proving system. The use of reusable inclusion proofs is a meaningful efficiency innovation that reduces gas costs and improves user experience compared to naive ZK approaches. The system is modular, upgradeable, and built on well-audited foundations (OpenZeppelin, Uniswap V2 math, Aztec's Barretenberg).

In short: **ZENDEX lets you trade, provide liquidity, and place limit orders on a public blockchain — while keeping your balances, amounts, and identity completely private.**

---

*Documentation compiled from the ZENDEX ZK-AMM repository: [github.com/lumos-codes-dev/zendex-sc-zk-amm](https://github.com/lumos-codes-dev/zendex-sc-zk-amm)*
