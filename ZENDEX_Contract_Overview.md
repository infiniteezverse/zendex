# ZENDEX ZK-AMM — Technical Contract Overview

**Document Status: INTERNAL TESTNET DRAFT**
> This document reflects the current private testnet deployment. Contract addresses, chain IDs, and fee parameters will be updated at each stage prior to public release.

---

## Deployment Stages

| Stage | Network | Chain ID | Status |
|---|---|---|---|
| **Current** | Horizen Testnet | 2651420 | Active — private testnet |
| **Next** | Base Public Testnet (Sepolia) | TBD | Pending handoff |
| **Final** | Base Mainnet | 8453 | Pending audit + launch |

> Fee split percentages and token parameters will be finalized and confirmed against contract implementation before public testnet. Numbers in this document are placeholder values from the current testnet deployment.

---

## 1. What Is ZENDEX?

ZENDEX (ZK-AMM) is a **privacy-preserving decentralized exchange (DEX)**. It lets users trade cryptocurrency tokens, provide liquidity, and place limit orders — all without revealing their balances, trade amounts, or identity to anyone observing the blockchain.

Most existing DEXes (like Uniswap) record every transaction publicly. Anyone can see exactly who traded what, for how much, and when. ZENDEX solves this by wrapping every financial operation inside a **Zero-Knowledge Proof (ZKP)** — a mathematical technique that proves "I did this correctly" without revealing any of the underlying private data.

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

## 3. Key Technologies

**UltraPlonk Proofs via Noir + Barretenberg** — The core privacy engine. ZENDEX uses the **UltraPlonk** proving system, a modern ZK construction that produces compact proofs and supports fast on-chain verification. Proofs are generated using the **Barretenberg** library (from Aztec), and the circuits that define the rules are written in **Noir**, a domain-specific language for ZK programming. UltraPlonk does not require a per-circuit trusted setup, making it more flexible and trustless than older systems like Groth16.

**Poseidon Hash** — A hash function optimized for ZK circuits. ZENDEX uses it to "seal" private data (amounts, keys, blinding factors) into short, opaque commitments.

**Merkle Tree** — A data structure that efficiently stores all user commitments. It allows anyone to prove that a specific commitment exists in a large set without revealing what the others are.

**UUPS Upgradeable Contracts** — Smart contracts built with the ability to be upgraded without users needing to move their funds. (UUPS = Universal Upgradeable Proxy Standard, an OpenZeppelin pattern.)

**Base (L2) Deployment** — ZENDEX deploys on Base, an EVM-compatible Layer 2. Gas is paid in ETH. ZEN is a native ERC-20 token on Base (`0xf43eb8de897fbc7f2502483b2bef7bb9ea179229`) and does not require wrapping.

---

## 4. How ZENDEX Works

The protocol revolves around one idea: **instead of storing your token balance on-chain, ZENDEX stores a cryptographic commitment to your balance.** Only you know the secret values behind that commitment.

Every operation (deposit, trade, withdraw) requires producing a ZK proof that you know the right secrets — verified on-chain without revealing those secrets.

### The Commitment

```
commitment = Poseidon(asset_id, amount, Poseidon(random_blinding, public_key_x, public_key_y))
```

Stored on-chain in a Merkle tree. Cannot be reverse-engineered to reveal balance or identity.

### The Nullifier

```
nullifier = Poseidon(Poseidon(random_blinding), public_key_x, public_key_y)
```

Published on-chain when a commitment is spent. Prevents double-spending without revealing which commitment was spent or who owned it.

---

## 5. The Inclusion Proof — Key Innovation

In a typical ZK system, every spend requires generating a fresh Merkle proof — computationally expensive and slow.

ZENDEX separates this into two steps:

**Step 1 — After Deposit (done once):**
An off-chain service generates a single **inclusion proof** after the deposit is confirmed:
*"Commitment X exists in the Merkle tree at epoch Y."*

**Step 2 — Every subsequent operation:**
The pre-generated inclusion proof is presented. The on-chain contract verifies a compact tag:

```
tag = Poseidon(epoch_id, commitment, salt)
```

The same inclusion proof is reused for withdrawals, swaps, liquidity operations, and orders — within the same epoch.

**Benefits:**
- Smaller proofs → lower gas fees
- Faster transaction preparation
- One proof covers many operations
- Higher throughput across the system

---

## 6. The Four Main Modules

---

### Module 1 — The Vault (Private Asset Management)

The **Vault** is where users privately hold tokens. Four operations:

**Deposit** — Lock tokens into the vault, receive a private cryptographic commitment. After mining, an inclusion proof is generated and stored for reuse.

**Withdraw** — Prove ownership of a commitment via inclusion proof + spend proof. Reclaim tokens. Nullifier published to prevent double-withdrawal.

**Split** — Divide one commitment into two smaller commitments privately.

**Join** — Merge two commitments into one larger commitment privately.

All operations require valid ZK proofs verified against on-chain verifier contracts.

---

### Module 2 — The AMM (Private Trading)

The **AMM** is ZENDEX's private trading engine, modeled on Uniswap V2's **constant product formula** (`x * y = k`).

**Add Liquidity** — Deposit two tokens into a pair pool privately. LP position stored as a private commitment.

**Remove Liquidity** — Reclaim pool share privately. Output becomes a new commitment.

**Swap** — Trade one token for another. Validated by ZK circuit (pricing, slippage, nullifier). Output inserted as a new private commitment.

Key constraints:
- Maximum slippage per swap: 5% (500 basis points)
- Leftover dust goes to a designated dust recipient
- Routing handled by the Router contract which also manages fee collection

---

### Module 3 — The Order Book (Private Limit Orders)

A **private limit order book** — rare in DeFi. Full lifecycle:

**Create Order** — Create a limit order using an inclusion proof + ZK proof. A "spot commitment" is stored in the tree.

**Request Cancel** — User generates a cancellation ZK proof. Only the user can authorize their own cancellation.

**Cancel Order** — Finalized by the Operator. Refund commitment released to user.

**Execute Orders** — When orders match, the Operator batches and executes. Verified on-chain via ZK proofs. Outputs become new private commitments for each party.

The Operator (`ORDER_BOOK_OPERATOR_ROLE`) can only execute or cancel orders — never move funds unilaterally.

---

### Module 4 — Fees, Staking & Rewards

#### Fee Splitter

Every trade incurs a fee collected in the traded asset. All fees are collected by the `RewardsEngine` and converted to **ZEN** via the router before distribution — creating continuous protocol-native buy pressure on ZEN with every trade regardless of which asset pair is being traded.

> Note: Fee percentage and split allocations are pending final confirmation against contract implementation before public testnet.

| Recipient | Allocation |
|---|---|
| LP & Maker Rewards | 50%+ |
| Protocol Treasury | TBD |
| TBD | TBD |
| TBD | TBD |

#### ZKZ Staking Boosts

> Note: ZKZ staking activates in Phase 2 after protocol traction is established. ZKZ will not be live at initial launch.

Users stake ZKZ governance token to increase their ZEN fee allocation from the Fee Splitter. The boost multiplier is calculated based on the amount staked and the lockup duration.

```
boost = 500 + 2000 × average(amount_factor, duration_factor)
boost capped at 2500 basis points (25%)
```

Staking parameters: minimum 1,000 ZKZ, lockup 1–24 periods.

#### Claiming Rewards

ZKZ stakers call `claim()` to collect their accumulated boosted ZEN allocation. The function delivers ZEN directly to the user's wallet. The remaining fee allocations (treasury, burn) are distributed to their respective destinations at the same time.

---

## 7. The ZK Verification Layer

Every privacy-sensitive operation is backed by a dedicated Noir circuit, proved via Barretenberg's UltraPlonk backend, compiled into a **Solidity verifier contract** deployed on-chain.

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

All verifiers are registered in **ZendexVerifierHub**, allowing individual verifiers to be upgraded as the proof system improves.

Every spend circuit follows the same four-step internal pattern:
1. Verify the inclusion proof (commitment is in the tree)
2. Verify the ECDSA signature (user authorized this action)
3. Compute the nullifier (mark this commitment as spent)
4. Validate all public outputs are consistent

---

## 8. The Merkle Tree (TreeOperator)

All private commitments live in an **Incremental Merkle Tree** managed by `TreeOperator`.

- Hash function: Poseidon
- Depth: configurable 1–32 levels (testnet default: depth 8, up to 256 commitments)
- Epoch tracking: new epoch ID recorded on each tree state change; inclusion proofs scoped to a specific epoch
- Stores: commitments, nullifiers, LP positions, order metadata

---

## 9. Contract Architecture

```
ZENDEX ZK-AMM
│
├── VAULT MODULE
│   ├── ZendexVaultManager      ← deposit/withdraw/split/join (upgradeable)
│   ├── ZendexVault             ← token custody (non-upgradeable)
│   └── TreeOperator            ← Merkle tree (non-upgradeable)
│
├── AMM MODULE
│   ├── ZendexAmmManager        ← swap/addLiquidity/removeLiquidity (upgradeable)
│   ├── ZendexFactory           ← creates trading pairs (upgradeable)
│   ├── ZendexRouter            ← routes trades, collects fees (upgradeable)
│   └── ZendexPair              ← constant product pool per pair (non-upgradeable)
│
├── ORDER BOOK MODULE
│   └── ZendexOrderBookManager  ← full limit order lifecycle (upgradeable)
│
├── FEE & REWARDS MODULE
│   ├── RewardsEngine           ← collects, converts, distributes fees (upgradeable)
│   ├── ZendexStaking           ← ZKZ token staking (upgradeable) [Phase 2]
│   └── BoostManager            ← calculates boost multipliers (upgradeable) [Phase 2]
│
└── VERIFICATION MODULE
    ├── ZendexVerifierHub        ← registry of all ZK verifiers (upgradeable)
    └── 10× Verifier contracts   ← one per Noir circuit (auto-generated by Barretenberg)
```

All manager contracts use UUPS proxy pattern — upgradeable without changing contract addresses or requiring user migration.

---

## 10. Privacy Guarantees

**Transaction Unlinkability** — Deposits and withdrawals cannot be linked on-chain. An observer sees only that "some commitment was created" and "some nullifier was published."

**Amount Privacy** — Trade and deposit values are never revealed on-chain.

**Asset Privacy** — Which token is being moved remains private within ZK proofs.

**Balance Privacy** — No user balance is stored on-chain; only commitments.

**Liquidity Privacy** — LP positions are private commitments, not public balances.

**Order Privacy** — Limit orders are not readable by market participants until executed.

---

## 11. Supported Assets

| Token | Role | Type | Contract |
|---|---|---|---|
| ZEN | Protocol token — fee distribution & rewards | ERC20 | `0xf43eb8de897fbc7f2502483b2bef7bb9ea179229` |
| WETH | Wrapped ETH for trading | ERC20 | TBD |
| USDC | Stablecoin | ERC20 | TBD |
| ZKZ | Governance/staking token | ERC20 | TBD — Phase 2 |

**ZEN Token Supply:** Hard capped at **21,000,000 ZEN** (enforced on-chain via `ERC20Capped`). Cannot be changed.

**Day 1 Liquidity Pairs:** ZEN/USDC, ZEN/WETH

**Phase 2 Pairs:** ZEN/ZKZ (activates with ZKZ launch)

> Gas on Base is paid in ETH, not ZEN. ZEN is the fee and reward settlement token for ZENDEX operations.

---

## 12. Typical User Journey

1. **User deposits USDC.** UltraPlonk proof generated client-side. Tokens locked in `ZendexVault`.

2. **Inclusion proof generated.** Off-chain service generates inclusion proof after deposit is mined. Stored and reused for all subsequent operations.

3. **User swaps USDC → ZEN.** Swap proof generated using the inclusion proof. USDC commitment nullified, new ZEN commitment created.

4. **User places a limit order.** Same inclusion proof used. Spot commitment stored on-chain.

5. **Order matched.** Operator calls `executeOrders()`. Both parties receive new private commitments.

6. **User withdraws ZEN.** Withdrawal proof generated. Nullifier published. ZEN transferred from `ZendexVault` to user's wallet.

No on-chain observer sees the user's balance, trade amount, or which commitment belongs to which address at any point.

---

## 13. Glossary

| Term | Explanation |
|---|---|
| ZK Proof | A mathematical proof that "I know a secret" without revealing the secret |
| UltraPlonk | The ZK proving system used by ZENDEX — no per-circuit trusted setup required |
| Noir | The language used to write ZENDEX's ZK circuits |
| Barretenberg | Aztec's proving library that generates and verifies UltraPlonk proofs |
| Commitment | A cryptographic seal on private data; short and opaque |
| Nullifier | A one-time code published when a commitment is spent, preventing reuse |
| Merkle Tree | A tree-shaped data structure that efficiently proves membership |
| Poseidon | A hash function optimized for use inside ZK circuits |
| Epoch | A snapshot of the Merkle tree state; scopes inclusion proofs |
| Inclusion Proof | A proof that a commitment exists in the Merkle tree at a given epoch |
| AMM | Automated Market Maker — prices tokens based on pool ratios |
| UUPS | Smart contract upgrade pattern that keeps addresses stable |
| Boost | A multiplier increasing ZEN fee allocation for ZKZ stakers |
| Basis Points | 1/100th of a percent. 100 bps = 1%, 2500 bps = 25% |

---

## 14. Pre-Launch Checklist

> Items to confirm and update before each deployment stage.

**Before Public Testnet (Base Sepolia):**
- [ ] Confirm final fee split percentages match contract implementation
- [ ] Update Chain ID from 2651420 to Base Sepolia
- [ ] Confirm `RewardsEngine` fee-to-ZEN conversion timing (at trade vs. at claim)
- [ ] Confirm WETH and USDC contract addresses on Base Sepolia
- [ ] Key signing handoffs complete

**Before Mainnet (Base):**
- [ ] Update Chain ID to 8453
- [ ] Confirm all contract addresses
- [ ] External audit complete
- [ ] Fee parameters finalized and confirmed on-chain
- [ ] ZKZ deployment timeline confirmed for Phase 2

---

*ZENDEX ZK-AMM — Internal Technical Reference — Not for public distribution until public testnet.*
