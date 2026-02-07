# On-Chain Consensus Protocol (OCP)

## 共识上链协议 · Proof of Commitment (POC)

> **OCP Master Whitepaper**. For mechanism details see [POC Whitepaper](./POC_WHITEPAPER_CN.md), for applications see [PM Application Spec](./PREDICTION_MARKET_APPLICATION_SPEC.md).

---

## Design Goals

**OCP’s fundamental purpose is to improve the accuracy with which consensus reflects the real world and digital space—without a single external signal source, “council of elders,” or oracle.** All protocol design choices serve this end.

---

## Abstract

If the Bitcoin protocol solved “value on-chain,” the On-Chain Consensus Protocol (OCP) aims to solve “**facts on-chain**.” It provides a permissionless finality layer for any public dispute about the external world.

This document presents the **On-Chain Consensus Protocol (OCP)**. Its core thesis: blockchains can carry not only money and contracts but **consensus about the external world**. Through a minimal “stake–challenge” game, OCP lets anyone bring a dispute on-chain and ultimately produce **Finalized Consensus**—the protocol’s unique, authoritative on-chain fact asset. The underlying logic is **Proof of Commitment (POC)**: the authority of consensus comes from **verifiable capital lock-up**, not identity or claims. The protocol does not produce “truth”; it outputs **capital-confidence outcomes**—the maximum commitment capital is willing to bear for a given outcome.

The protocol uses a **three-outcome game**: YES (fact holds), NO (fact reversed), and INVALID (fact broken / premise absent / unverifiable). When an event is cancelled, truth is unknowable, or the proposition is ambiguous, INVALID acts as a circuit breaker and triggers full refunds, avoiding unfair binary outcomes for either side.

This whitepaper describes OCP as a base layer and presents its first application: a fully endogenous prediction market.

**Keywords**: consensus on-chain, facts on-chain, Proof of Commitment (POC), stake, challenge, three-outcome game, finalized consensus, capital confidence, prediction market, oracle-free

---

## 1. Introduction: The Last Mile of Consensus

### 1.1 The Problem

Real-world consensus—who won the match? which proposal is better?—is hard to turn into **trustworthy, programmable finality** in the digital world. How can decentralized systems make irreversible, manipulation-resistant collective commitments about external facts or subjective choices?

### 1.2 Existing Approaches

- **Oracles** (Chainlink, UMA, etc.): Depend on off-chain authorities or complex governance; centralization and bribeability risks
- **On-chain courts** (Kleros): Majority voting, but consensus is decoupled from capital
- **Prediction markets** (Polymarket, Augur): Depend on external settlement sources or reporter games; truth relies on external referees

Common issue: **the last mile of consensus**—how to bring off-chain disputes on-chain and resolve them irreversibly—has not been solved in an elegant way.

### 1.3 Our Approach

**OCP**: A protocol layer that treats consensus itself as a **settleable asset** and finalizes it through a permissionless game. At its core is **Proof of Commitment (POC)**—the authority of consensus is proved by verifiable capital lock-up, not identity or claims. It does not rely on “people telling the truth,” but on “lying or being wrong being economically costly.”

---

## 2. OCP Protocol Design: Two Actions, Three Outcomes

### 2.1 Objectives

Define how a dispute over a **binary proposition** is “moved” on-chain and resolved. The protocol’s only output is **Finalized Consensus**—a programmable, irreversible “on-chain fact” asset.

**Three outcomes**: The protocol outputs three possible results—YES (fact holds), NO (fact reversed), INVALID (fact broken / premise absent / unverifiable). When a match is cancelled, truth is unknowable, or the proposition is ambiguous, a binary verdict is unfair to one side; INVALID must be introduced as a circuit breaker. **Typical cases where YES or NO cannot be decided** include: contradictory evidence, chain reorg invalidating a relied-on fact, timeout (e.g. no clear majority by end of challenge period or external data source remains undecided), etc.—all go through the INVALID path.

**Important**: OCP outputs **capital-confidence results**—the maximum commitment capital is willing to bear for an outcome—not an objective “truth” verdict. Under typical conditions the two are highly correlated; in edge cases they may diverge (see §5.1).

### 2.2 Core Three Elements

| Element | Action / Output | Definition |
|--------|------------------|------------|
| **Stake** | `stake(position, amount)` | Entry point for consensus on-chain. Participants bind capital one-way to YES, NO, or INVALID, fully locked until settlement |
| **Challenge** | `challenge(type, bond)` | Stress test of consensus. **Y/N challenge**: only the disadvantaged side (the side with less capital among YES/NO) may initiate; **INVALID challenge**: anyone may initiate, claiming the event cannot be decided |
| **Finalized Consensus** | On-chain readable | The protocol’s unique, authoritative output. Emerges when the cooling period ends (no challenge) or when the challenge period ends; consumed as on-chain fact by contracts and downstream applications |

### 2.3 Challenge Bond b

**Protocol definition**: b = vault minimum deposit.

- **Rule is protocol-given**: The challenge bond always equals the vault minimum deposit.
- **Value is set by vault creator**: When creating a vault, the deposit token (unit of account) and minimum deposit (numerical value of b) are specified.
- **Property**: b is an **absolute amount** in the deposit token; it does not depend on external price APIs or oracles.

Participation threshold and challenge threshold are symmetric: minimum participation unit = challenge cost.

### 2.4 Challenge Period: Re-voting Allowed

After a challenge is triggered, an **open challenge period** begins. During this period:

- New capital may flow into YES, NO, or INVALID;
- **Existing stakers may re-vote and change position**—when new information appears (e.g. match cancelled), participants can move to INVALID, etc., improving how well consensus reflects the real world.

### 2.5 Flow and Protocol Finality Rules

**OCP defines the following finality conditions** (globally consistent; not overridable by applications):

- **Cooling period (Δ)** is unique: after the event’s designated time (stake) deadline, the cooling period begins; challenges may be initiated during the cooling period; **no challenge** → Finalized Consensus forms immediately when the cooling period ends;
- **If a challenge occurs during** the cooling period → enter **challenge period (Δ)**; **no further challenges are accepted**; when the challenge period ends, the protocol **immediately** forms Finalized Consensus.

**Three-outcome finality rules**:

| Condition | Result | Settlement |
|-----------|--------|------------|
| YES weight > 50% | YES wins | YES side takes all NO-side funds (including opponent bond) |
| NO weight > 50% | NO wins | NO side takes all YES-side funds (including opponent bond) |
| Otherwise | INVALID wins | Full refund for all; profit share extracted per original vault logic |

The protocol has only this single round. This ensures global consistency and authority of the “Finalized Consensus” state—analogous to Bitcoin’s “heaviest chain” defining block finality.

```
Event creation → Open stake period (until t₀) → Deadline
    ↓
Cooling period (Δ): Y/N disadvantaged side or anyone may post bond b to challenge (including INVALID challenge)
    ↓
No challenge → Protocol finality, output Finalized Consensus
Challenge → Open challenge period (Δ): YES/NO/INVALID may receive flows; participants may re-vote
    ↓
Challenge period ends → Protocol finality
    ├─ YES>50% → YES takes all
    ├─ NO>50%  → NO takes all
    └─ Else    → INVALID, full refund for all
```

### 2.6 INVALID Settlement Details

| Case | Creator (vault creator) b | Challenger b | Stakers |
|------|---------------------------|--------------|---------|
| **INVALID wins** | Refunded | Refunded | Principal returned to source |
| **INVALID loses** (YES or NO wins) | — | Forfeited; distributed proportionally to vault depositors (**excluding the challenger**) | Winning side takes all |

### 2.7 Parent–Child Events and Root Events

To improve how well consensus reflects the real world, the protocol supports **parent–child event references**:

- **Parent event**: A precondition proposition (e.g. “The match on May 1, 2026 proceeded normally”)
- **Child event**: A proposition that depends on the parent (e.g. “Lakers won”)

If the parent event finalizes as INVALID (e.g. match cancelled), the child event **automatically triggers refund logic**—no extra adjudication; the protocol layer guarantees settlement consistent with reality.

A root event is one with no parent; parent–child chain depth is implementation-constrained (e.g. depth ≤ 64).

### 2.8 History and Ledger

The protocol maintains an **event history ledger**—every event’s stakes, challenges, and finality results are recorded. **Participant history** (e.g. cumulative winning stakes, challenge records) is used to incentivize users to uphold truth and is infrastructure for improving consensus accuracy. History is unforgeable, like Bitcoin’s blockchain.

### 2.9 Application-Layer Flexibility: Chaining Rounds

Unlike parent–child events in §2.7, here each round is an independent OCP event; the previous round’s consensus is only input to the next round’s proposition, with no protocol-layer automatic refund propagation. Applications that need to model multi-round games (e.g. complex arbitration, multi-stage governance) can **chain multiple OCP events**: use the previous round’s Finalized Consensus as input to a new event in the next round. Each OCP event itself still follows the one-shot finality rules above.

### 2.10 Relation to Proof of Commitment (POC)

OCP is the **goal** (consensus on-chain, programmable finality); **POC** (Proof of Commitment) is the **mechanism** (stake–challenge–consensus). The POC whitepaper details mechanism, game theory, and philosophical metaphor. The two are the same protocol’s goal and mechanism descriptions.

---

## 3. Why It Works: Game Theory and “Expensive Consensus”

### 3.1 Not Truth, but Cost of Being Wrong

OCP does not rely on “people telling the truth,” but on **“lying or being wrong being economically costly.”** A challenger profits only when they have a strong information advantage and can move the market (attract re-votes).

### 3.2 Rational Condition for Challenging

Let the challenger’s subjective probability be $q$, payoff if challenge succeeds $\pi$, bond $b$, cost $c$. A challenge occurs if and only if:

$$
q \geq \frac{b+c}{\pi+b}
$$

**Conclusion**: Only high-confidence actors challenge. The bond filters noise and keeps challenges serious.

### 3.3 Security Philosophy

OCP does not output “truth,” but **“consensus that cannot be overturned at any cost that was actually paid.”** That is exactly the credible basis needed for social coordination. If an attack (malicious challenge) fails, the bond rewards the defenders; the system is antifragile.

---

## 4. Utility Evolution: From Native Apps to AI Alignment

OCP is not merely a gambling tool but a **general-purpose consensus engine**. As adoption grows, its utility evolves along: native applications → infrastructure → value alignment.

### 4.1 Primary Utility: Native Consensus Applications (Prediction & Governance)

At this stage, OCP directly serves as the core consensus layer for decentralized applications (DApps), addressing the pain of “decisions and settlement depending on third parties.”

#### 4.1.1 Oracle-Free Prediction Markets

This is a direct mapping of the OCP mechanism. Traditional prediction markets rely on external oracles to establish outcomes, introducing centralization and data-bribe risk.

- **Endogenous settlement**: OCP’s “stake–challenge” game puts settlement in the hands of market participants. Any long-tail event (e.g. “tomorrow’s rainfall in a given community”) can form and settle consensus as long as there is counter-party stake—no need to wait for an authoritative data feed.
- **Manipulation resistance**: An attacker must post exponentially growing bonds to oppose an honest majority; cost is far higher than bribing a single oracle node.

#### 4.1.2 Optimistic Governance & Dispute Arbitration

DAO governance today suffers from “voter apathy” and “low execution efficiency.” OCP enables an **optimistic governance** pattern:

- **Proposal as stake**: Any proposer can initiate a governance action via OCP (stake YES).
- **Default pass, challenge on exception**: If no valid challenge (stake NO) is made during the cooling period (Δ), the proposal is deemed passed and executed. This greatly raises governance efficiency, turning “every vote must be cast” into “correct on demand.”
- **Objective KPI verification**: For on-chain protocol performance (e.g. “Did the grant recipient complete the development milestone?”). OCP turns it into a challengeable proposition, without relying on a single admin’s subjective signature.

### 4.2 Infrastructure Utility: “Consensus-as-a-Service” in the Sync Era

As Ethereum moves toward Based Rollups and real-time proving, on-chain execution reaches second-scale sync, but **external facts** (oracle data, cross-chain state, real-world events) remain at minute- or hour-scale confirmation. OCP introduces an **Infrastructure-as-Consensus** view, bridging “technical finality” and “fact finality.”

#### 4.2.1 Truth Synchronicity

ZK-Rollups answer “is the computation correct?”; OCP aims to answer “is the input true?” In a synchronous composability setting, OCP does not pursue “slower but more accurate” facts but **“same-speed and committable”** facts, enabling true end-to-end atomic operations.

#### 4.2.2 Economic Pre-finality

To match L2’s millisecond execution, OCP offers a layered finality model:

- **Probabilistic guarantee**: When a proposition is backed by high $cumulativeWinningStake$, the economic cost to reverse it grows non-linearly.
- **Instant confidence**: A sequencer can reference OCP state with “high confidence” as a layer of **pre-confirmation finality** within seconds of ZK proof generation, without waiting for the physical cooling period to end. This gives synchronous applications “usable certainty” in practice.

#### 4.2.3 INVALID as a Sync Circuit Breaker

In atomic cross-chain calls, binary logic (YES/NO) alone cannot handle real-world ambiguity (e.g. data source failure, disputed event). OCP’s INVALID state acts as a native logic breaker: when the underlying fact enters a gray zone, INVALID triggers predefined refund/rollback contract logic. This prevents cascading asset loss from “correct execution of wrong facts” (ZK proving the execution of a false fact) without triggering heavy L1-level chain reorgs.

### 4.3 Ultimate Utility: AI Value Alignment & Consensus Market

When AI agents begin to manage assets and decisions autonomously, the core question is: “Align to whose values?” Current AI alignment is often defined by a few centralized labs—inherently fragile and closed. OCP offers a decentralized alternative.

#### 4.3.1 Value-on-Chain

OCP allows subjective disputes over value preferences, safety boundaries, and acceptable behavior to be brought on-chain.

- **From “fact” to “value”**: One can stake not only “who won the match” but “whether this AI’s behavior is ethical.”
- **Decentralized alignment target**: Global participants express positions with capital commitment; the challenge mechanism filters noise and malice. The resulting Finalized Consensus becomes a **“alignment target”** that AI systems can read and execute.

#### 4.3.2 Expensive Consensus as an Anchor for Human Will

What if AI’s values are not hard-coded but continuously shaped by a global consensus market? OCP provides a continuously running consensus market. In it, only collective will that has passed expensive play (skin-in-the-game) becomes input to AI. This does not replace technical alignment but supplies decentralized, verifiable value input—alignment target shifts from a single principal’s instructions to human collective will refined by expensive consensus. In the long run, OCP can serve as infrastructure for “value on-chain”: not only facts but values can be finalized; not only money but alignment targets can be programmed.

---

## 5. Discussion: Boundaries, Parameters, and Future

### 5.1 Applicability Boundaries

OCP is best suited for **publicly verifiable, time-bounded** events. Protocol output may diverge from truth when:

- Information is not publicly verifiable
- Capital is dominated by ideology
- Truth exists but no one challenges (informed parties expect re-votes to favor the other side)

In such cases the output is “expensive error”—a capital-confidence result, not objective truth. This defines the protocol’s domain of applicability.

### 5.2 Key Parameters

The protocol defines finality rules and the **definition** of b (b = vault minimum deposit). The following are set at vault or application layer:

| Parameter | Set by | Too small | Too large |
|-----------|--------|-----------|-----------|
| $b$ (challenge bond = vault min deposit) | Vault creator | More noise challenges | Legitimate challenges suppressed |
| Δ (cooling period, challenge period) | Application layer | Insufficient information aggregation | Lower capital efficiency |

---

## 6. Conclusion

The On-Chain Consensus Protocol (OCP) turns **real-world disputes** into **on-chain programmable finalized consensus**. Its mechanism is Proof of Commitment (POC): stake–challenge–consensus. The output is a capital-confidence result—a measure of social epistemic hardness—not absolute truth.

**Blockchains can carry not only money and contracts but consensus about the external world.** OCP is the protocol-layer implementation of this vision.

---

## 7. Appendix: Terminology and Document Index

| Term | Definition |
|------|------------|
| OCP | On-Chain Consensus Protocol; goal is to bring consensus on-chain and form Finalized Consensus; fundamental purpose is to improve how accurately consensus reflects the real world |
| **POC (Proof of Commitment)** | Consensus authority is proved by verifiable capital lock-up, not identity or claims |
| Facts on-chain | Relative to “value on-chain”; the core problem OCP solves: turning external-world disputes into on-chain programmable Finalized Consensus |
| **Three-outcome game** | Finalized Consensus has three results: YES, NO, INVALID |
| **b (challenge bond)** | Protocol defines b = vault minimum deposit; vault creator sets deposit token and minimum deposit |
| **INVALID** | Fact broken / premise absent / unverifiable; triggers full refund for all |
| **Finalized Consensus** | The protocol’s unique, authoritative output; programmable, irreversible on-chain fact asset |
| **Cooling period** | One of Δ. Window after stake deadline and before finality when challenges may be initiated; if no challenge in period, direct finality; if challenge, enter challenge period |
| **Challenge period** | One of Δ. Opens after a challenge; YES/NO/INVALID may receive flows; participants may re-vote; finality when period ends |
| **Cumulative winning stake / Participant history** | Recorded in event history ledger; cumulative stakes won in past finalities, etc., used to incentivize truth maintenance and improve consensus accuracy; see §2.8 |
| Capital confidence | Maximum commitment capital is willing to bear for an outcome |
| Creator | Vault creator; sets deposit token and minimum deposit when creating vault |

| Document | Content |
|----------|---------|
| [OCP Whitepaper](./OCP_WHITEPAPER_EN.md) | This document; narrative framework for consensus on-chain |
| [POC Whitepaper](./POC_WHITEPAPER_CN.md) | Mechanism details, game theory, stake–challenge–consensus philosophy |
| [Prediction Market Application Spec](./PREDICTION_MARKET_APPLICATION_SPEC.md) | Prediction market mechanics |

---

*Document version: 1.0 | On-Chain Consensus Protocol (OCP) Whitepaper | Three-outcome game, b = minimum deposit, parent–child events*
