# Proof of Commitment (POC): A Scheme for Costly Social Consensus

> **Implementation Status**: This document defines the mechanism layer (POC) of OCP. The on-chain consensus narrative framework is in [OCP Whitepaper](./OCP_WHITEPAPER_EN.md). For the prediction market application, see [PREDICTION_MARKET_APPLICATION_SPEC.md](./PREDICTION_MARKET_APPLICATION_SPEC.md).

---

## Abstract

This paper proposes a **triadic on-chain protocol** to form anti-manipulation capital consensus over external propositions. The protocol runs in three phases:

1. **Staking**: participants weight beliefs with capital and form an initial distribution.
2. **Challenge**: minority participants can pay a cost to stress-test the provisional majority, opening a challenge window.
3. **Consensus**: if and only if one side can withstand all profitable challenges, the protocol outputs final consensus.

This protocol does not produce truth; it is a **stress-testing furnace for consensus**, serving as a trust-minimized base layer for prediction, arbitration, and governance. Its core logic is **Proof of Commitment (POC)**: the authority of consensus is proven by verifiable capital lock-in.

**Keywords**: staking, challenge, consensus, Proof of Commitment (POC), winner-takes-all, game theory

---

## 1. Introduction: The Cost of Consensus

### 1.1 Problem

How can decentralized systems make **irreversible, anti-manipulation collective commitments** on external facts or subjective choices?

Oracles rely on off-chain authorities or heavy governance. On-chain courts (e.g., Kleros) rely on majority voting. Prediction markets (e.g., Polymarket, Augur) rely on external settlement sources. The common issue is that **truth or final judgment depends on external arbiters**, creating trust and manipulation risk.

### 1.2 Core Thesis

The strongest form of credible commitment is the willingness to bear **large, verifiable financial risk**. This protocol encodes that principle: no external judge is introduced; **capital itself** forms consensus through game dynamics. The value of consensus comes from surviving costly challenges.

### 1.3 Protocol Philosophy

**The protocol does not produce truth; it produces the most costly consensus.**

How to use it—event prediction, dispute resolution, governance voting, or belief signaling—is an application-layer choice.

**This protocol is a game machine that converts staking into consensus, and challenge is its only necessary conversion engine.**

---

## 2. Mechanism Design: Staking — Challenge — Consensus

### 2.1 Staking Phase

**Staking**: a participant unilaterally binds capital to one proposition within a specific time window. This is both a belief signal and an explicit cost.

- **Three-state outcomes**: YES, NO, INVALID (fact broken / premise absent / unverifiable)
- **Deadline $t_0$**: end of staking phase, followed by cooldown
- **Participants**: any address with depositable assets can stake on YES, NO, or INVALID with **full lock-up**; each participant can choose only one side (can add more on the same side)

The initial weight distribution is the capitalized expression of beliefs. The leading side is the side with capital weight > 50% (between YES and NO, for Y/N challenge rights).

### 2.2 Cooldown and Challenge Phase

**Cooldown** (duration Δ, default 24h): after $t_0$, anyone can post bond $b$ to challenge (the losing Y/N side, or INVALID challenge).

- **Y/N challenge right**: only the disadvantaged side between YES/NO
- **INVALID challenge right**: anyone can claim the event is unresolvable
- **Bond $b$**: protocol rule: $b$ equals the vault minimum deposit amount; token and numeric value are set by the vault creator
- **No challenge**: finalization at cooldown end

**Challenge phase** (duration Δ, same as cooldown): starts if a challenge is filed in cooldown. **No new challenge can be opened**. New capital can flow into YES, NO, or INVALID, and existing stakers may re-vote/reposition entirely to another side (one side at a time per user). This dynamic phase is the protocol’s security core.

**One-shot finality**: protocol-level single round only. If no challenge in cooldown, finalization is immediate. If challenged, finalization happens at challenge-phase end. This is immutable at protocol level to guarantee globally consistent Finalized Consensus.

### 2.3 Consensus Formation

**Consensus**: the stable terminal state that emerges after a capital distribution can continuously resist profitable challenges.

- **Terminal condition**: cooldown ends with no challenge, **or** challenge phase ends
- **Three-state settlement**:
  - YES > 50% → YES wins and takes NO-side funds (including opponent bond)
  - NO > 50% → NO wins and takes YES-side funds (including opponent bond)
  - Otherwise → INVALID wins, all users refunded; dividends follow the vault’s original logic
- **If INVALID challenge fails**: challenger bond $b$ is confiscated and distributed pro-rata to vault depositors (excluding challenger)

### 2.4 Parent-Child Events and History

To improve real-world accuracy, the protocol supports **parent-child event references**. If a parent event (e.g., “match takes place”) ends as INVALID, child events (e.g., “Lakers win”) auto-refund. The protocol maintains event and participant histories (e.g., cumulative winning-side stakes) to reward truth-maintaining behavior.

### 2.5 Terminology Mapping

| Legacy wording | Layer |
|---|---|
| deposit, commitment window, voting weight | **Staking** |
| challenge, bond, cooldown, challenge phase | **Challenge** |
| settlement, winner-takes-all, finality, INVALID refund | **Consensus** |

### 2.6 Key Properties

| Property | Description |
|---|---|
| **Minimality** | only two operations: stake and challenge |
| **Three-state finality** | YES/NO winner-takes-all; INVALID full refund |
| **One-shot finality** | immutable protocol rule; avoids infinite games |
| **Oracle-free** | consensus fully endogenous |
| **$b$ = minimum vault deposit** | rule defined by protocol; value/token chosen by vault creator |

---

## 3. Theoretical Analysis: Challenge as the Converter

The center of analysis is not “whether capital is always truthful,” but **how challenge converts fragile stake distributions into robust consensus**.

### 3.1 Game-Theoretic Model: Seriousness of Challenge

A successful challenge requires both **(a) high private confidence** and **(b) the ability to attract marginal stake**.

Let challenger subjective probability be $q = \mathbb{P}(T=\text{challenger side})$, expected gain if successful be $\pi$, challenge operating cost be $c$, and bond be $b$:

$$
\mathbb{E}[U_{\text{challenge}}] = q(\pi - c) - (1-q)(b + c)
$$

Challenge occurs iff:

$$
q \geq \frac{b+c}{\pi+b}
$$

Only high-confidence challengers with expected ability to mobilize challenge-period capital will act. Bond $b$ filters noisy/frivolous disputes.

### 3.2 Security Philosophy: Challenge as Immune System

System security is not the absolute size of stake, but whether consensus survives challenge.

- **Challenge is the immune system**: fragile distributions are eliminated under stress; only resilient ones survive to finality.
- **Attack strengthens defense**: failed malicious challengers transfer bond to defenders, producing anti-fragile dynamics.

### 3.3 Challenge-Period Capital Flow

Let challenge-phase inflow be $\Delta W_A, \Delta W_B$. Under typical conditions (information discoverability, profit-seeking capital, and enough informed participants):

$$
\mathbb{E}[\Delta W_A - \Delta W_B \mid T=A] > 0,\quad \mathbb{E}[\Delta W_A - \Delta W_B \mid T=B] < 0
$$

Even when only part of new capital is informed, the expected bias can hold if signal is not fully drowned by noise.

### 3.4 Comparison with Schelling Point

This protocol is a **capitalized, contestable Schelling game**:

| Schelling game | OCP/POC mechanism |
|---|---|
| voting weight | stake weight |
| coordination cost | lock-up + bond |
| coordination failure | direct capital loss |
| focal point | winner of challenge-period competition |

Equivalent framing: **Capital-Weighted Schelling Game with One-Shot Challenge**.

---

## 4. Protocol Properties and Security Model

### 4.1 Oracle-Free

Consensus and settlement are fully endogenous and decided by on-chain capital distribution.

### 4.2 Trust Minimization

Trust only code and participant incentive alignment. No centralized admin and no off-chain committee.

### 4.3 Security Boundary

When capital is systemically misled (e.g., ideology-dominated events, unverifiable information, highly correlated signals), output can be **costly error** rather than truth. The protocol claims costly-tested consensus, not guaranteed truth.

---

## 5. Application Scenarios

All applications are concrete instantiations of the same Staking–Challenge–Consensus flow.

### 5.1 Prediction Markets

- **Staking**: users buy/sell YES/NO shares and deposit into vaults
- **Challenge**: losing side can post bond and open challenge period
- **Consensus**: at challenge end, leading side takes all and settles

See [PREDICTION_MARKET_APPLICATION_SPEC.md](./PREDICTION_MARKET_APPLICATION_SPEC.md).

### 5.2 Decentralized Arbitration

- **Staking**: plaintiff/defendant collateralize on side A/B
- **Challenge**: losing side challenges and tries to attract “jury capital”
- **Consensus**: challenge end yields final ruling

### 5.3 High-Stakes Governance

- **Staking**: capital is locked for approve/reject
- **Challenge**: opposition can attempt reversal during challenge window
- **Consensus**: only challenge-resilient distributions become executable final outcomes

### 5.4 Belief Signaling and Commitment

The core value is the observed commitment hardness during staking and challenge, even when payout itself is secondary.

---

## 6. Discussion, Limits, and Future Work

### 6.1 Parameter Effects

| Parameter | Set by | Too small | Too large |
|---|---|---|---|
| $b$ (= minimum vault deposit) | vault creator | noisy challenges increase | valid challenges suppressed |
| Δ (cooldown/challenge duration) | application layer | insufficient information aggregation | lower capital efficiency |

### 6.2 Limitations

- **Low liquidity**: weak consensus hardness under sparse participation
- **Highly ambiguous events**: output may diverge from truth
- **Truth but no challenger**: informed users may rationally avoid challenge if expected challenge inflow is still unfavorable

### 6.3 Future Work

- Cross-chain implementations
- Proportional bond design relative to leader-side stake
- Richer governance/arbitration application specs

---

## 7. Conclusion

POC provides a base primitive to **capitalize social belief and stress-test it**. Its output measures hardness of collective commitment, not absolute truth.

**The protocol is a game machine that converts staking into consensus, and challenge is its only necessary converter.**

If Bitcoin formalized trustless value transfer, POC formalizes **trustless costly commitment** for broader social coordination.

---

## Appendix A: Glossary (Staking–Challenge–Consensus)

| Term | Precise definition |
|---|---|
| **Staking** | unilateral capital binding to one proposition during a period; belief signal plus explicit cost |
| **Challenge** | disadvantaged side pays extra cost to request one bounded public re-competition |
| **Consensus** | stable winner-takes-all distribution that survives profitable challenge opportunities |
| **POC** | authority of consensus proven by verifiable capital lock-in, not identity claims |
| **Staking phase** | until $t_0$, users stake to YES/NO/INVALID with full lock-up |
| **Cooldown** | duration Δ after $t_0$ where challenge may be filed |
| **Challenge phase** | duration Δ after challenge; re-vote/new stake allowed; no new challenges |
| **One-shot finality** | immutable protocol-level rule for globally consistent final consensus |
| **Capital confidence** | maximum commitment capital can bear for an outcome |
| **INVALID** | broken premise / absent fact / unverifiable proposition; all refunded |
| **$b$** | challenge bond equal to vault minimum deposit amount |

---

## Appendix B: References

- Schelling, *The Strategy of Conflict*  
- Wolfers & Zitzewitz (2004), “Prediction Markets,” NBER w10504  
- Marx, *Das Kapital*, Vol. 1  
- Polymarket / UMA dispute mechanism literature

---

*Document Version: 1.0 | POC Mechanism Whitepaper | Three-state game, $b$ as minimum deposit, parent-child events*
